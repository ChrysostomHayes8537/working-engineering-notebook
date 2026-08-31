# Browser Multipart Upload: Presigned Parts, Complete, and Abort

Use a browser direct upload with multipart presigned parts only when the application owns a durable commit record: it should authorize each part, verify the submitted inventory, then choose complete or abort. The deciding constraint is not transfer speed. It is whether a file can become visible without a trustworthy record of which bytes, principal, and object key the application meant to publish.

Large files turn a pleasant-looking browser upload into a distributed transaction with no transaction manager. The browser knows its local `File` slices, object storage knows which parts it accepted, and the application knows who is allowed to create an object. Treating a progress bar as the authority across those boundaries is how incomplete transfers get mistaken for finished objects.

Commit is publication.

This architecture decision record uses an S3-compatible multipart protocol as the storage boundary. A multipart upload has an initiation step, individual part uploads, and a completion call that supplies the ordered part numbers and ETags; an abort discards the in-progress upload. The exact API surface differs among compatible services, so the protocol contract, documented limits, and lifecycle rules need testing against the selected endpoint rather than an assumption that a familiar client library hides every difference.

## What should browser multipart presigned parts do before complete or abort?

The browser should move bytes; it should not decide publication. A small Node.js control plane, or an equivalent service, creates an upload intent before it issues a presigned URL. That intent contains the authenticated actor, normalized object key, opaque upload identifier, expected byte ranges, part size, expiry, and a nonterminal state. The browser can then upload a slice for an authorized part number and retain the returned ETag exactly as returned.

The important distinction is easy to skip. A successful part response says storage accepted a part; it does not create the final object. Completion is the commit point because it receives the part inventory that defines the object. If a browser times out after sending a part, it has uncertainty, not proof that a retry is required. Reconciliation against the storage-side inventory is safer than repeatedly transmitting the same range until a local counter looks right.

Keep retries bounded. A retry must reuse the same part number for the same byte range, while a different file needs a new intent even if the filename matches. A moving window of presigned parts limits how many authorizations remain usable after a user cancels, but it adds control-plane round trips and can stall a fast connection when the signing path is slow. Presigning a whole plan is reasonable for short sessions with a modest, documented part count. It is not a universal default.

The protocol needs one more boundary: uploaded bytes are not necessarily published bytes. A workflow that must inspect content can upload to a quarantine key, record the completed object, inspect it, and promote it under separate authorization. This introduces another state transition and extra storage operations, yet it makes the policy boundary visible instead of pretending that a browser-provided content type proves anything.

## Decision record: who owns the final object state?

The selected design is an application-owned intent with direct browser-to-storage data transfer. Storage remains the record of accepted parts; the application database remains the record of authorization and publication intent. The control plane must make state transitions atomic, because a second tab, a cancellation request, and a retry can all act on the same upload identifier.

| Option | Appropriate use | Failure boundary | Trade-off |
|---|---|---|---|
| Browser calls complete directly | Low-risk objects where the browser is the publication authority | A client can commit an inventory the application did not re-check | Fewer control calls, weaker server-side policy boundary |
| Application proxies the file | Inline inspection is mandatory before storage receives any byte, or clients cannot reach storage | Application capacity carries bandwidth and backpressure | Clearer centralized control, an extra data hop |
| Direct transfer with an application commit | Large browser files and a distinct authorization record | Completion and abort can race | More coordinator state, predictable ownership |

This is deliberately not a claim that direct upload fits every system. The catch is operational discipline: the application now has to persist intent before authorization, reconcile ambiguous outcomes, and clean up abandoned multipart work. Stick with a proxy when inline inspection before storage, network reachability, or centralized flow control matters more than removing the application from the byte path.

The state machine can stay small: `initiated`, `uploading`, `completing`, `completed`, and `aborted`. Terminal transitions should be idempotent from the caller's perspective. A second completion request for a completed intent returns the stored result; an abort request after completion reports the terminal state rather than attempting to erase a published object. This is less dramatic than a clever retry loop. It is the part that holds under a tab close or a duplicated request.

The recovery case deserves a precise walkthrough. Imagine that a browser has uploaded the last authorized range and sends its ETag inventory to the control plane, then loses its connection before it receives a response. The next action cannot be chosen from the browser's spinner: the complete request may have reached the control plane, the control plane may have claimed the state but not yet called storage, or storage may have completed the object while the database update is still pending. The handler must therefore read the intent first, return a persisted terminal result when one exists, and let only the process that atomically claimed `completing` invoke the external complete call. After a process restart, a recovery worker can inspect intents left in that intermediate state, query the storage-side result through the provider's documented mechanism where available, and finish recording the terminal outcome. The exact reconciliation API is provider-specific, which is why the application needs an explicit reconciliation policy instead of pretending an HTTP timeout says what storage did. This is also why deleting an intent record immediately after a client cancellation is a bad bargain: it removes the only join key for understanding the state of uploaded parts.

## Critical state transitions for complete and abort

The following example is intentionally generic. `storage` represents an implementation of the chosen compatible protocol, and `repo.claim` is an atomic compare-and-set operation. Its value is in the ordering of checks, not in any particular SDK.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Part:
    number: int
    etag: str


def complete_upload(repo, storage, intent_id: str, actor_id: str, parts: list[Part]):
    intent = repo.get_for_actor(intent_id, actor_id)
    if intent.state == "completed":
        return intent.result
    if intent.state == "aborted":
        raise ValueError("upload is already aborted")

    ordered = sorted(parts, key=lambda part: part.number)
    numbers = [part.number for part in ordered]
    if numbers != list(range(1, intent.expected_parts + 1)):
        raise ValueError("part inventory is incomplete or duplicated")
    if any(not part.etag for part in ordered):
        raise ValueError("each part requires its returned ETag")
    if not repo.claim(intent_id, expected="uploading", next_state="completing"):
        return complete_upload(repo, storage, intent_id, actor_id, parts)

    result = storage.complete_multipart(
        key=intent.object_key, upload_id=intent.upload_id, parts=ordered
    )
    repo.mark_completed(intent_id, result)
    return result


def abort_upload(repo, storage, intent_id: str, actor_id: str):
    intent = repo.get_for_actor(intent_id, actor_id)
    if intent.state in {"completed", "aborted"}:
        return intent.state
    if not repo.claim(intent_id, expected="uploading", next_state="aborted"):
        return repo.get_for_actor(intent_id, actor_id).state
    storage.abort_multipart(key=intent.object_key, upload_id=intent.upload_id)
    return "aborted"
```

There is a policy choice hidden in `expected_parts`. It works when the service fixes the file size and part size before issuing presigned parts. A streaming source with an unknown final size needs a different final-part rule and server-side range validation. Do not copy this check into that case unchanged.

An abort path deserves the same attention as completion. An explicit cancel can invoke abort, but a browser closing cannot be relied on to finish that request. Expire inactive intents in the application and configure the storage service's documented incomplete-multipart lifecycle cleanup as a backstop where it exists. The lifecycle setting is a safety net, not a substitute for a visible operational state.

Nothing cleans itself up reliably.

## Validation, observability, and the rejected shortcut

Test the state boundaries rather than only the happy path. Kill a test worker immediately before and after the external complete call; race complete against abort; submit reordered, duplicate, and missing ETags; resume after a browser restart; and let an intent expire while some parts are outstanding. Those cases show whether an object can be published twice, whether cleanup can race a commit, and whether support staff can explain an upload that the client calls "stuck."

Record an internal intent ID, an upload-ID fingerprint, part number, byte range, attempt count, transition, and duration. Do not put full presigned URLs in logs: their query parameters are authorization material. Separate metrics for intent creation, authorization issuance, browser-reported part transfer, reconciliation, complete, abort, and age of nonterminal intents are more useful than a single upload-success percentage. A large file browser direct upload can look healthy while its authorization window or completion path is the actual bottleneck.

The rejected shortcut is to let the browser call complete with whatever ETags it has collected and to call abort only from a UI cancel button. It looks compact, but it gives the least durable participant final authority and leaves tab-close cleanup to chance. It can be suitable for disposable, low-risk objects whose clients are explicitly trusted to publish them. For application data with ownership, retention, or inspection rules, the control-plane commit is the narrower and more auditable boundary.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://www.backblaze.com/cloud-storage/pricing
