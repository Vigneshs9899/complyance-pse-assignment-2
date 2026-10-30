# Case report — PSE-A2-002

## 1. Summary and impact

- **Customer:** Cedar Demo Malaysia (fictional)
- **Priority:** P2
- **Scope:** One customer's import workflow. Twelve records were accepted, but only seven had genuine results while the import was marked completed. Five records remained pending.
- **Current status:** Local code fix implemented, 27/27 automated tests passing, and the supplied Cedar lab case successfully reconciled. Production deployment, independent engineering approval, and customer verification have not occurred.
- **Reported observation:** Customer reported 12 accepted records, seven visible results, and a completed job. This was treated as a report requiring investigation, not an established diagnosis.
- **Verified baseline:** `case-002` was completed with 12 total, 7 processed, and 5 pending. Original results existed for record IDs 1–5 and 11–12. Maple `control-002` was paused with 3 pending records.
- **Safety:** Do not blindly re-upload the partially processed file. The approved temporary workaround is the single-record import flow for genuinely new records only.

Evidence: `evidence/baseline-import.json`, `baseline-records.json`, `baseline-results.json`, `baseline-events.json`, `baseline-maple.json`.

## 2. Draft customer communications

All messages below are drafts for the fictional scenario. No customer communication was sent.

### Acknowledgment — simulated 26 October 2026, 09:35 GST / 13:35 MYT

Hello,

Thank you for reporting the incomplete import. I understand that 12 records were accepted, the job appears complete, but only seven results are visible. I appreciate that this is interrupting your workflow.

I will own the investigation and verify the import status, individual source records, results, and processing events before recommending any recovery action.

Please do not upload the same file again for now, as this could create duplicate imports. For genuinely new records only, the existing single-record import flow may be used.

I will provide the first progress update by 10:00 GST / 14:00 MYT, even if the investigation is still in progress.

Regards,  
Product Support

### Checkpoint update — simulated 26 October 2026, 10:00 GST / 14:00 MYT

Hello,

I have started checking the import at the individual-record level rather than relying only on the job's completed status.

The initial investigation confirms that the supplied case contains 12 source records, with seven processed and five still pending, although the import is marked completed. The separate Maple control import remains paused.

The cause has not yet been confirmed at this checkpoint. I am tracing the batch-processing behavior and checking how the pending records are selected.

Please continue to avoid re-uploading the original file. I will share another update after the processing path and a safe recovery approach have been verified.

Regards,  
Product Support

### Final update / conditional closure — simulated 26 October 2026, 12:00 GST / 16:00 MYT

Hello,

The investigation identified a batch-processing issue in which records could be skipped while the pending-record list was changing between batches.

A local code correction has been implemented and verified with automated integration tests. The supplied Cedar training case was then requeued using the approved lab-only recovery command.

The local verification now shows all 12 original source records processed, 12 genuine results present, and zero pending records. The original record IDs and earlier results were retained. The separate Maple control import remains paused and unchanged.

This confirms recovery in the supplied local training environment only. It does not confirm a production deployment or restoration.

The next step is independent engineering review and an approved release through the appropriate process, followed by customer-side verification. Support should retain ownership of customer communication until those steps are confirmed.

Regards,  
Product Support

## 3. Reproduction and root cause

### Environment and reproduction

- Operating system: Windows PowerShell.
- Runtime: Bun 1.4.4 and Node.js v24.16.0.
- Application: supplied TypeScript/Bun support lab.
- Default server port: 4328.
- Default configured batch size: 5.
- Original database: `starter/.data/lab.sqlite`.
- Regression tests use isolated test databases and the actual HTTP application.

Baseline inspection used:

- `GET /health`
- `GET /imports/case-002`
- `GET /imports/case-002/records`
- `GET /imports/case-002/results`
- `GET /imports/case-002/events`

### Expected versus observed

Expected behavior: every valid source record should be processed exactly once, produce one genuine result, and leave no pending records when the import is completed.

Observed baseline:

- 12 source records.
- Records 1–5 processed.
- Records 6–10 pending.
- Records 11–12 processed.
- Seven genuine results.
- Import state incorrectly reported as completed.

### Confirmed root cause

In `src/worker.ts`, the worker maintained an increasing offset while querying only pending records.

The pending set shrank after each committed batch. After the first five records were processed, the pending query returned only the remaining seven records. Increasing the offset to five then skipped the next five pending records and returned records 11–12. The worker subsequently advanced the offset and received an empty batch, allowing the import to be marked completed while five records remained pending.

### Permanent change

Changed the worker to always fetch the first remaining pending batch using offset `0`.

The worker continues processing until no pending records remain. The existing configured batch size, repository transaction behavior, API contract, and workspace boundaries were retained.

Files changed:

- `src/worker.ts`
- `tests/http.test.ts`
- `tests/harness.ts`

The harness shutdown check was adjusted to accept the expected `SIGTERM` termination signal on Windows while still rejecting unexpected termination outcomes.

## 4. Tests and existing-case reconciliation

| Check | Actual result |
| --- | --- |
| Regression before fix | New 12-record test failed: expected 12 processed and 0 pending; observed 7 processed and 5 pending. |
| Regression after fix | Passed. |
| TypeScript | `bun run check-types` passed. |
| Full integration suite | `bun run test`: 27 tests passed, 0 failed. |
| Input volumes | Existing 1-record and 5-record tests passed; 12-record regression passed. |
| Batch size 5 | 12 records processed correctly across bounded batches. |
| Batch size 3 | After test-server restart with batch size 3, 12 records processed in four batches. |
| Result identity and normalization | Tests verified source/result identities and normalized labels. |
| Validation and workspace isolation | Existing validation and isolation tests passed. |
| Restart persistence | Restart test passed, preserving results, IDs, events, and paused control. |
| Original case before recovery | 12 total, 7 processed, 5 pending; original evidence preserved. |
| Recovery command | `bun run resume-case` returned the existing `case-002` as queued with 5 pending. |
| Original case after recovery | 12 processed, 12 genuine results, 0 pending. |
| Maple control | Remained paused with 3 pending records. |

### Original case reconciliation

The approved lab-only command was run against the original database:

`bun run resume-case`

The command requeued the existing import; it did not create a new import.

The final event history showed:

- Original batch: record IDs 1–5.
- Original batch: record IDs 11–12.
- `resume_requested` with 5 pending.
- Recovery batch: record IDs 6–10.
- `processing_completed`.

Final records retained IDs 1–12 and references `DOC-001` through `DOC-012`. Final results contained exactly one result for each original record ID, with normalized labels `SAMPLE 001` through `SAMPLE 012`.

No duplicate result IDs were observed. The seven earlier results were retained.

Evidence:

- `evidence/final-import.json`
- `evidence/final-records.json`
- `evidence/final-results.json`
- `evidence/final-events.json`
- `evidence/final-maple.json`

### Outstanding checks and limitations

- The lab is local training software; no production environment was accessed.
- No production deployment or release was performed.
- No customer-side verification was possible or attempted.
- Independent engineering review remains outstanding.
- Automated tests verify the local application behavior, not a production rollout.

## 5. Escalation and customer ownership

### Decision

Request independent engineering review before any production release. The local change resolves the reproduced cause and passes the regression suite, but it has not been reviewed or deployed through a production process.

Support retains responsibility for customer continuity and the next update. No engineering owner has accepted this request yet.

### Linear-ready draft — not submitted

**Title:** P2 — Import worker skips pending records when processing batches

**Priority:** P2

**Impact:** An import can be marked completed while source records remain pending and genuine results are missing.

**Reproduction:**
1. Process a 12-record import with batch size 5.
2. Worker queries pending records with an increasing offset.
3. After the first batch is committed, the pending result set shrinks.
4. The next offset skips records 6–10 and processes records 11–12.
5. The import can be marked completed with five pending records.

**Evidence:**
- Baseline import, records, results, and events.
- Regression test failure before the fix.
- 27 passing integration tests after the fix.
- Final original-case records, results, and event history.

**Local change:** Fetch the first remaining pending batch with offset zero on each iteration.

**Engineering request:**
- Independently review the worker change and regression coverage.
- Confirm transaction and retry behavior.
- Determine the appropriate release and deployment process.
- Confirm production verification requirements before customer closure.

**Requested owner:** Engineering reviewer / worker-code owner (requested, not assigned).

**Requested checkpoint:** After engineering review, provide an acceptance or change request and the next release decision.

**Support ownership:** Continue customer updates and avoid claiming production restoration until deployment and customer-side verification are confirmed.

### Exit criteria

The case should not be represented as production-resolved until:

1. Engineering review is complete.
2. An approved release is deployed through the normal process.
3. Production processing and result identities are verified.
4. The customer confirms the expected results are visible.

If any verification fails, support should reopen investigation with the new evidence. No restoration ETA is claimed.

## 6. Submission notes

### Evidence index

**Baseline**
- `evidence/baseline-import.json`
- `evidence/baseline-records.json`
- `evidence/baseline-results.json`
- `evidence/baseline-events.json`
- `evidence/baseline-maple.json`

**After recovery**
- `evidence/final-import.json`
- `evidence/final-records.json`
- `evidence/final-results.json`
- `evidence/final-events.json`
- `evidence/final-maple.json`

**Test evidence**
- Add the saved test output as `evidence/test-results.txt`.

### AI assistance

ChatGPT was used to help interpret the assignment requirements, inspect the reported code path, reason about the regression test, troubleshoot the Windows test harness, and draft documentation and customer communications.

The candidate ran the commands locally, reviewed the code changes, executed the tests, inspected API responses, and verified the original case reconciliation. No AI integration was added to the application.

### Limitations

All results are from the supplied local training environment. Production behavior, independent review, release approval, deployment, and customer confirmation remain unverified.