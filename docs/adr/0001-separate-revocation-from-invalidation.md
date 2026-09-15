---
status: accepted
---

# Separate ExecutionRevocation from ResultInvalidation

Stopping an Attempt from continuing and discarding an already-accepted Attempt's result are different operations with different triggers and different blast radius: a policy tightening that forbids *future* execution should not by itself erase a result that was already validated, and an already-EXITED Attempt has no running process to cancel. We considered collapsing both into a single `CANCELLED`/`POLICY_VIOLATION` transition, but that would either let a stale-but-accepted result survive a revocation it shouldn't, or force us to void results any time execution is merely stopped going forward. We instead record two distinct operations — `ExecutionRevocation` (recorded before cancellation; blocks further starts; does not by itself void prior completion unless an accompanying invalidation is recorded) and `ResultInvalidation` (targets an EXITED/VALIDATING/SUCCEEDED result; atomically clears `authoritative_attempt_id`; preserves original hashes; never becomes re-authoritative through later policy loosening). Both require an authenticated operator or an approved deterministic rule; an LLM may only propose either.
