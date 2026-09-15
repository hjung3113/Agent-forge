---
status: accepted
---

# DONE is terminal; completion validity is a sidecar record

A DONE Task's evidence can later turn out to be unacceptable — for example, an Attempt it depended on gets `ResultInvalidation`ed. We considered adding a `REOPENED` state between `DONE` and `VERIFYING` so the Task machine could formally walk the completion back and try again. We rejected that: reopening a terminal state invites rewriting history (what happens to Attempts spawned after "DONE" that assumed the task was over?), and it blurs the guarantee that DONE is something downstream systems (GitHub issue closure, operator dashboards) can treat as final. Instead, `Task.state` stays `DONE` forever once reached, and a separate `TaskCompletionValidity {task_id, done_event_id, status: valid|voided, reason, authorization}` record — written in the same transaction as DONE, and voided atomically if a dependency is later invalidated — tracks whether that completion is still trustworthy. Consumers must check validity explicitly (`DONE(voided)`), not assume `state == DONE` means "still good." No new Attempts run against a terminal DONE Task, voided or not; remediation is always a new Task. This trades a slightly more awkward read path (two fields instead of one state) for never having to reconcile "what does re-entering VERIFYING after DONE even mean."
