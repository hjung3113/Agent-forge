---
status: accepted
---

# Separate ExecutionRevocation from ResultInvalidation

Stopping an Attempt from continuing and discarding an already-accepted Attempt's result are different operations with different triggers and different blast radius: a policy tightening that forbids *future* execution should not by itself erase a result that was already validated, and an already-EXITED Attempt has no running process to cancel. We considered collapsing both into a single `CANCELLED`/`POLICY_VIOLATION` transition, but that would either let a stale-but-accepted result survive a revocation it shouldn't, or force us to void results any time execution is merely stopped going forward. We instead record two distinct operations — `ExecutionRevocation` and `ResultInvalidation` — each with an explicit fence so neither can be raced:

- **`ExecutionRevocation`** is recorded before cancellation and blocks further starts. It carries a revocation epoch/ordering token. Any `RUNNING`/`STARTING` Attempt's `EXITED → VALIDATING → SUCCEEDED` transition and any authoritative-attempt assignment for that Attempt is a compare-and-swap against that epoch: acceptance that would occur *after* the revocation epoch is automatically ineligible, even if the Attempt happened to exit cleanly in the window between the revocation being recorded and the cancel signal actually reaching the process. `ExecutionRevocation` does not by itself void a result that was already authoritative *before* its epoch — that is `ResultInvalidation`'s job, applied explicitly.
- **`ResultInvalidation`** targets an EXITED/VALIDATING/SUCCEEDED result and clears `authoritative_attempt_id` only via compare-and-swap on the targeted Attempt id: if the pointer no longer equals the Attempt being invalidated (for example, a retry already made a later Attempt authoritative), the clear does not happen and the current authoritative result is left alone — invalidation instead applies only to the specific evidence/artifacts that consumed the targeted Attempt. Original hashes are preserved either way, and nothing invalidated becomes re-authoritative through later policy loosening.

Both operations require an authenticated operator or an approved deterministic rule; an LLM may only propose either.
