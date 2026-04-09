# solution.md

## Q1: Design Choices — The Settlement Contract

### 1) When will the review screen and backend diverge?
There is a time gap between:
- the frontend snapshot: `useSettlement()` calls `fetchWorklogs(userId, "OPEN")` on mount (and then every 30s), and
- settlement execution: `POST /api/settlements/run/:userId` triggers `runSettlementForUser(userId)` which **re-fetches** open worklogs and **recalculates amounts** from DB.

Anything that changes the database in that gap can cause divergence. Concrete scenarios:

- **Segments added after review**: a new `time_segments` row is inserted after the last poll. The UI still shows the old `wl.amount`; the backend includes the new segment when it calls `getTimeSegments()` at execution.
- **Adjustments added/changed after review**: a new `adjustments` row (bonus/penalty) is inserted after the last poll. Backend includes it via `getAdjustments()` and `calculateWorkLogAmount()`.
- **A previously settled worklog is reopened**: `POST /api/worklogs/:workLogId/segments` calls `db.reopenWorkLog(workLogId)` (sets `status='OPEN'`). The backend will now pick up a worklog that was not part of the admin’s reviewed list.
- **Concurrent settlement**: another admin (or `POST /api/settlements/run`) settles first, flipping worklogs to `SETTLED`. The admin’s tab can remain stale until the next poll; the backend will find fewer/no open worklogs.
- **Server restart / multi-instance**: the in-memory `settledWorkLogIds` Set is per-process; behavior can differ across restarts/instances even for identical user actions.

### 2) Does the admin have any signal when numbers diverge?
No explicit signal.

The success screen renders `settlementResult.totalAmount` from the backend response (not `previewTotal`). The admin has to notice the discrepancy manually; there is no “approved vs executed” diff.

Also, the confirm button label is built as:

```tsx
`Confirm Settlement — $${previewTotal.toFixed(2)}`
```

so the UI will display a **double `$`** (e.g., `Confirm Settlement — $$50.00`), which is confusing in a financial flow.

### 3) What problems does the lack of idempotency create? How does `settledWorkLogIds` interact?
The settlement POST carries **no worklog IDs, no amounts, and no idempotency key**. That creates:

- **Retry ambiguity**: if the request succeeded but the response was lost, retrying cannot safely “resume”; it becomes a new “settle whatever is open now” command.
- **Partial failure dead-end**: if worklogs are marked `SETTLED` but `createRemittance()` throws, the DB now has no OPEN worklogs, and the system cannot “try again” to create the missing remittance.

What happens in this repo on retry when the backend returns `{ message: "No open worklogs found for this user." }`:
- the frontend’s `runSettlement()` treats HTTP 200 as success and returns JSON,
- `confirmSettlement()` then tries `result.remittance.id` and throws a **TypeError** (caught and shown as “Settlement failed. Please try again.”).

The in-memory `settledWorkLogIds` Set:
- **Helps narrowly**: avoids duplicate processing within the same process lifetime for IDs already added to the Set.
- **Does not solve idempotency** across restarts or multiple instances.
- **Can harm correctness**: it is never cleared; a legitimately reopened worklog ID can be skipped forever in that process.

### 4) What would a robust settlement contract look like?
The contract should be explicit, idempotent, and validated server-side.

**Frontend sends**:

```json
{
  "idempotencyKey": "uuid-v4",
  "workLogIds": ["WL-003", "WL-004"],
  "expectedTotal": 350.00
}
```

Optionally include per-worklog expected amounts and/or a snapshot/version (`reviewedAt`, ETag/watermark).

**Backend validates before executing**:
- The `idempotencyKey` is persisted and returns the same remittance on retry.
- The submitted `workLogIds` are still eligible (`OPEN`, belong to the user, not already settled).
- Recomputed totals match expected totals (otherwise return `409 Conflict` with fresh breakdown for re-review).
- Perform settlement inside a **single DB transaction** (status updates + remittance + line items), preventing “SETTLED but unpaid” partial state.

Note: a pure frontend “pre-flight refetch” conflicts with `candidate/frontend/AGENTS.md` guidance (“Do not re-fetch immediately before a user-initiated mutation”). Correctness needs a backend contract, not just UX staleness checks.

---

## Q2: Trace the State

### Setup (seed math)
Bob (`USR-002`) has two OPEN worklogs:
- **WL-003**: segments \(2×50 + 3×50 = 250\), adjustment ADJ-002 \(−500\) → amount **−250**
- **WL-004**: segments \(5×60 = 300\), no adjustments → amount **300**

So `previewTotal` shown by the UI is: \(-250 + 300 = 50\).

### 1) Poll has not fired since ADJ-004. What does the component still have?
The tab is stale; local React state still contains:
- WL-003 at **−$250.00**
- WL-004 at **$300.00**
- `previewTotal` **$50.00**

The confirm button still displays **`Confirm Settlement — $$50.00`** (double `$`).

### 2) Backend computes fresh data. What is WL-003 now and what is the remittance total?
ADJ-004 is inserted: \(+300\) on WL-003.

Recompute WL-003:
- segments: \(250\)
- adjustments: \(-500 + 300 = -200\)
- total: \(250 - 200 = 50\)

WL-004 remains \(300\).

Remittance total: \(50 + 300 = 350\) → **$350.00**.

### 3) What does the admin see? Where could discrepancy be prevented?
The success screen shows **$350.00** (backend-confirmed), despite the admin approving **$$50.00** in the button label.

Where to prevent/catch:
- **Best**: backend validates an explicit contract (IDs + expected totals + idempotency) and rejects mismatches.
- **Also**: show approved vs executed totals with an explicit warning (but that is after the fact).

### 4) UX only, or financial harm?
Financial harm is possible whenever the admin’s decision depends on the amount (approval thresholds, fraud review, payout caps). A large adjustment inserted between page load and confirm can change the executed payout materially without the admin ever seeing that amount before authorizing the action.

---

## Q3: Predict the Failure Mode

### Setup
In `runSettlementForUser` the engine:
1. Computes amounts for WL-003 and WL-004,
2. Marks each worklog `SETTLED` (and adds its ID to `settledWorkLogIds`),
3. Then calls `createRemittance(userId, totalAmount)` (this is the call that throws in the scenario).

### 1) Database state after the crash
- WL-003: **SETTLED**
- WL-004: **SETTLED**
- Remittance for Bob: **does not exist** (creation threw)
- Line items: **do not exist**
- Bob has been paid: **$0** (no remittance recorded)

### 2) Frontend after 500
`confirmSettlement()` catches and displays `"Settlement failed. Please try again."`. The worklogs table remains visible because `setWorklogs([])` only runs on success.

### 3) Retry behavior (backend)
On retry, `getOpenWorkLogsByUser("USR-002")` returns **[]** (it filters `status='OPEN'`). The API responds HTTP 200 with:

```json
{ "message": "No open worklogs found for this user." }
```

### 4) Retry behavior (frontend)
The frontend expects a `SettlementResponse` with `remittance`. It attempts `result.remittance.id` and throws a TypeError, which is caught and surfaces as `"Settlement failed. Please try again."` again.

### 5) Compound state and recovery
- **Database**: worklogs are SETTLED, but no remittance exists → unpaid settlement, inconsistent ledger.
- **In-memory**: `settledWorkLogIds` contains WL-003/WL-004 for that process lifetime.
- **New admin session**: sees “No open worklogs to settle.” (because DB says SETTLED).
- **Recovery**: requires manual intervention (transactional repair / reopen worklogs + restart to clear Set / or manually create the missing remittance). Normal operation cannot fix “SETTLED but unpaid”.

---

## Q4: What Happens With THIS Input?

### Step 1 — POST: add segment to WL-002
`POST /api/worklogs/WL-002/segments` inserts a new segment and then calls `db.reopenWorkLog("WL-002")`, so WL-002 becomes **OPEN** (even though it was previously settled).

Assume server restart: `settledWorkLogIds` is empty.

### Step 2 — Review fetch: which worklogs are OPEN for USR-001?
`getOpenWorkLogsByUser("USR-001")` now returns **WL-001** and **WL-002**.

### Step 3 — Amount calculations shown in the table
**WL-001**:
- segments: \(4×75 = 300\), \(3.5×75 = 262.5\)
- total: **$562.50**

**WL-002** (includes the previously settled segments + adjustment, plus the new segment):
- TS-003: \(6×75 = 450\)
- TS-004: \(2×75 = 150\)
- new segment: \(1.5×75 = 112.5\)
- ADJ-001: \(-150\)
- total: \(450 + 150 + 112.5 - 150 = 562.5\) → **$562.50**

Preview total: \(562.5 + 562.5 = 1125.0\) → UI label shows **`Confirm Settlement — $$1125.00`**.

### Step 4 — Settlement execution
Backend recomputes the same amounts (same underlying inputs) and creates a remittance for **$1,125.00**.

### Step 5 — Ledger and overpayment
- REM-001 (January): **$450.00** (already paid for WL-002)
- REM-002 (February): **$1,125.00**
- Total paid: \(450 + 1125 = 1575\) → **$1,575.00**

Correct life-to-date total owed from all segments and adjustments (counted once) is:
- WL-001: $562.50
- WL-002: $562.50
- Correct total: **$1,125.00**

Overpayment: \(1575 - 1125 = 450\) → **$450.00**

### Step 6 — Why “silent agreement” is worse
Preview and backend execution match because they share the same flawed model (“OPEN worklog means pay the full recomputed amount”), so the admin loses the only signal (disagreement) that might have triggered scrutiny. The system confidently overpays while appearing correct.

---

## Q5: Fix Evaluation

## Fix A — Backend: Use `filterNewSegments`

### 1) Which segments does it keep for WL-002?
With `lastSettledAt = 2025-01-31T18:00:00Z`, `filterNewSegments` keeps only segments where `createdAt > lastSettledAt`:
- TS-003 (Jan 12): discarded
- TS-004 (Jan 13): discarded
- new February segment: kept

### 2) What amount does it compute (segments filtered, adjustments not filtered)?
`newSegments` total: \(1.5×75 = 112.50\)

`adjustments` still includes ADJ-001 \(−150\) (not filtered).

Amount: \(112.50 - 150.00 = -37.50\) → **−$37.50** (incorrect; it double-applies an already-settled adjustment).

### 3) Strict `>` edge case
If a segment is created at a timestamp exactly equal to `lastSettledAt`, strict `>` excludes it. Even if rare, the impact is an underpayment that can be hard to detect.

## Fix B — Frontend: Pre-flight validation

### 4) Does it prevent Q2’s stale preview problem?
Yes, it can detect that the total changed since load (freshTotal != previewTotal) and force a re-review.

### 5) Can the data change again between pre-flight and execution?
Yes — that is a TOCTOU (time-of-check vs time-of-use) race. It reduces the window, but does not guarantee correctness.

Also, this repo’s frontend standards explicitly discourage refetching immediately before a user-initiated mutation, so Fix B conflicts with the stated UX pattern.

### 6) Are GET `/api/worklogs` and settlement guaranteed to compute the same amounts?
They are equivalent today, but the logic is duplicated:
- GET computes amounts inline in `api.ts`
- settlement uses `calculateWorkLogAmount`

Duplication risks drift over time.

## Root cause (Question 7)
The fundamental flaw is that the system does not durably record **what inputs were already settled** (segment/adjustment attribution). Without a segment-/adjustment-level settlement ledger (or equivalent durable marking), reopened worklogs can re-include already-paid items.

Structural fix:
- Persist idempotency and settlement attribution (which segments/adjustments were included in which remittance/line item), and
- Settle only “unsettled” inputs,
- In a single transaction (status updates + remittance + attribution), so double counting and partial failure are structurally prevented.

