# Signed-Scan Retention: Large Multipart Uploads, Sharp, and Private Storage

Short answer: for an education platform retaining signed documents until an explicit deletion deadline, commit the large image before starting Sharp, keep originals and derivatives private, and make deletion a durable state transition rather than a storage afterthought.

The throughput problem is easy to misdiagnose. The question is not whether Node.js can receive a large image, or whether Sharp can resize it. The question is where bytes wait, which event makes the source trustworthy, and how the system proves that a document is gone when its retention deadline arrives. A thumbnail that looks fine can still be evidence of a broken retention design.

I would treat the original scan as an immutable input and every thumbnail as disposable output. That gives the pipeline a clean dependency: upload, commit, inspect, resize, deliver, delete. It also gives operations something they can audit when a learner's signed enrollment document reaches its deadline.

## What must private storage prove before Sharp processes large image uploads for retention?

Use a multipart upload for genuinely large scans, but release the Sharp job only after the multipart upload has been completed. A part upload is transport state. It is not yet the object that downstream work should read.

Commit first.

The control flow is deliberately boring:

1. Create an upload record containing the document ID, object key, deadline, and upload state.
2. Send parts to private object storage and persist each part number and returned integrity value.
3. Complete the multipart upload from the recorded part list.
4. Change the application record to `committed` and enqueue one thumbnail job.
5. Let the Node.js worker read the committed original through the storage interface, use Sharp to resize it, and write derivatives under separate keys.
6. Return short-lived signed URLs for authorized viewing, then delete the original and derivatives when policy says to do so.

The last uploaded part is not the commit event. If a client disappears after part 12 of 12 but before completion, the thumbnail worker must see no usable source. Enqueueing on the final part creates a race between transport cleanup and image processing, and that race will eventually produce a missing or partial input.

Keep the key layout explicit. For example, `documents/8f2a/original` can be the source and `documents/8f2a/preview/w320.webp` can be a derivative. The application record should carry the document ID, source key, derivative keys, transform version, state, and deletion deadline. Prefixes help an operator list objects; they are not a substitute for that index.

Here is the small state model I would test before connecting a real storage client. It is Python because the invariant matters more than an SDK, while the production worker can remain Node.js with Sharp.

```python
from dataclasses import dataclass
from enum import Enum


class UploadState(Enum):
    OPEN = "open"
    COMMITTED = "committed"
    ABORTED = "aborted"
    DELETED = "deleted"


@dataclass(frozen=True)
class Document:
    document_id: str
    source_key: str
    derivative_keys: tuple[str, ...]
    delete_after: str
    state: UploadState


def release_thumbnail_job(document: Document) -> tuple[str, ...]:
    if document.state is not UploadState.COMMITTED:
        raise ValueError("only a committed source can release thumbnail work")
    return document.derivative_keys


def can_delete(document: Document, now: str) -> bool:
    return document.state is UploadState.COMMITTED and now >= document.delete_after
```

The production implementation needs an idempotent transition around `COMMITTED`. A lost response after completion must not cause two logical jobs, and two workers must not silently apply different transform policies to the same derivative key. Put the job identity on the document ID plus transform version, and make the worker safe to retry. If a queue is unavailable, a database record with a claim timestamp is still preferable to a process-local flag.

## What fails when throughput is measured only at the upload endpoint?

Measuring request duration alone hides the work that matters. Track part retries, completion latency, bytes per second, time from commit to first derivative, queue age, decode failures, and deletion lag. A large file can finish its network transfer quickly and still occupy a Sharp worker long enough to starve smaller documents.

The most expensive design mistake is usually unbounded concurrency. Set separate limits for incoming parts and image decoding. Upload concurrency consumes network and storage connection capacity; Sharp concurrency consumes CPU and memory. They are different budgets. A queue that admits twenty 200 MB scans because the upload endpoint is healthy can exhaust the worker host when all twenty begin decoding.

There is no honest universal multipart threshold. Client network quality, scan dimensions, retry cost, memory limits, and the accepted deadline all matter. I’m not sure a threshold derived from one average file will survive a school district uploading scans over a weak connection. Measure the distribution first, then choose the boundary and document why it exists.

Security checks belong before decoding. OWASP recommends allowlisting extensions, validating the actual file type rather than trusting `Content-Type`, changing stored filenames, limiting file size, and authorizing uploaders. A signed upload URL only delegates byte transfer; it does not make an uploaded image safe. Keep the bucket private, validate before Sharp reads the object, and constrain image dimensions and resource use.

Deletion needs the same operational attention as ingestion. Store the deadline in application state and run a reconciler that finds committed records past that deadline. Delete the source and all known derivatives, record the deletion result, and make the action retryable. The catch is that a storage listing cannot reliably tell you which transform versions belong to a document, so the application index must be complete before the deadline arrives. In an edtech system, that record should also preserve the policy version that produced the deadline, the actor or workflow that approved retention, and the last observed state for each object, because a later change to course policy must not silently move an already-issued deadline. If the source was committed but a preview job is still queued, the deletion worker should cancel or supersede that job before removing the source; otherwise a retrying worker can recreate a derivative after the deadline. If deletion of one derivative succeeds and the next call is interrupted, the record must say which keys remain so the reconciler can continue without treating a partial result as a clean completion. This is why I would make deletion a small state machine with an audit event, not a nightly list-and-delete script.

If an upload is abandoned, retain the upload identifier long enough to abort it explicitly. Otherwise staged parts can remain as accounting and cleanup work even though no document exists. The cleanup policy must distinguish an open upload from a committed object; a lifecycle rule that is convenient for ordinary objects may not express the product's hour-level or day-level deadline for unfinished multipart state.

One more boundary is worth naming: a deletion deadline is not automatically a legal erasure guarantee. Replication, backups, access logs, caches, and provider retention controls can change what “deleted” means. The product owner and compliance reviewer must define the claim. Your mileage may vary by jurisdiction and contract, so record the scope instead of promising that one object-delete call erases every copy.

## The storage model behind private thumbnail delivery

S3-compatible storage can make the data path portable, but compatibility is a starting point, not an operational contract. Compare the controls that affect this workload: multipart behavior, private delivery, lifecycle precision, object recovery, regional placement, observability, and how much of the index your application must own.

| Decision area | Direct specialist storage | S3-compatible abstraction | Application consequence |
| --- | --- | --- | --- |
| Multipart throughput | Provider-specific limits and tuning are visible | Common request shape can reduce adapter work | Benchmark the actual part size and concurrency |
| Private delivery | Native policy and audit controls may be broader | Signed URLs and policy surface may be narrower | Keep authorization in the application |
| Deletion deadline | Lifecycle and retention semantics vary | Abstraction may expose fewer lifecycle controls | Reconcile from a durable deadline index |
| Recovery | Versioning, retention, and replication may be available | Some controls may not be portable | Do not promise recovery or erasure without testing |
| Team ownership | More provider configuration to understand | Less storage-specific code to maintain | Confirm the abstraction does not hide a required control |

The right choice is the one whose missing controls you can name. For this edtech workload, an abstraction is unsuitable when the retention contract requires provider-native legal hold, object versioning, cross-region recovery, or a deletion audit that the abstraction cannot expose. Stick with a direct storage product when those controls are part of the requirement, even if that means maintaining another integration.

Conversely, direct storage is a poor fit for a small team that has not budgeted for policy configuration, credential rotation, lifecycle review, and provider-specific testing. A common interface can reduce adapter code and let the application use plain HTTP from any language, but it cannot remove the need to understand where data is replicated, how private links expire, or which deletion states are observable.

The comparison should happen after the state machine is written. Otherwise “S3-compatible” becomes a comfort label, and the team discovers during an audit that it can upload and resize but cannot prove the deadline transition.

## A rollout test for signed-scan deletion

Start with a non-production bucket and synthetic documents. Test a successful commit, a missing part, a lost completion response, an abandoned upload, a repeated thumbnail job, an oversized or mislabeled file, and a deadline-triggered deletion. Each test should assert both storage state and application state; a green HTTP response is not enough.

Then move one document class at a time. Route large scans through multipart, keep a simpler path for small files if that makes sense, and compare throughput and retry rates against the same acceptance criteria. Run Sharp workers with explicit CPU and memory limits. Alert on queue age, derivative lag, orphaned uploads, and deletion lag rather than waiting for a learner to report a missing preview.

Keep the rollout reversible at the metadata layer. New keys should identify the transform version, old originals should remain untouched until the new derivative is verified, and a failed thumbnail should not extend the source's retention deadline without an explicit policy decision. Delete through a worker that can resume from its record, not through a one-off script whose progress exists only in terminal output.

The compact decision rule is this: optimize multipart throughput only after commit, thumbnail work only after commit, and deletion from an indexed deadline. The storage vendor is secondary to those boundaries. If the chosen service cannot make one of them observable, it is the wrong fit for the document contract.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [DigitalOcean Spaces documentation](https://docs.digitalocean.com/products/spaces/)
