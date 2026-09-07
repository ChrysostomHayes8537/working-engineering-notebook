# US-EU SaaS Image Delivery: A Thumbnail Generation and Object Storage Decision

Uploads crossing US and EU boundaries make the storage location, retry model, and deletion path more important than the resizing library. **Short answer:** for a small SaaS, accept each original into regional object storage, enqueue deterministic derivative work, and write a small fixed set of thumbnails back to object storage; put a CDN in front for delivery, but don't make on-request transformation part of the correctness path unless unpredictable formats and sizes justify that extra dependency.

This is an architecture decision record, not a product ranking. The choice optimizes for a boring invariant: once an upload is acknowledged, the original is durable in the intended region and every thumbnail can be rebuilt from it. Fast first display matters, but it comes after recoverability.

Durability first.

## What invariants should govern small SaaS image thumbnail generation across US and EU storage?

The original is the source of truth. A thumbnail is a disposable derivative whose identity should include the source object's immutable version, the transformation specification, and the encoder version. If any of those inputs change, the object key changes too. That rule prevents a retry from overwriting an unrelated result and prevents a CDN from serving an old bitmap under a reused URL.

Keep region ownership explicit. An EU upload should enter an EU processing path and an EU bucket; a US upload should follow the equivalent US path. The application may keep shared metadata, but the work item needs a region and an immutable source reference so a worker never guesses where bytes belong. I'm not sure whether a particular SaaS must keep derivatives in the same jurisdiction as originals; counsel and the actual data classification decide that. The architecture should make either policy enforceable rather than hiding movement inside a third-party transform URL.

The failure boundaries are equally important. A client disconnect must not erase an acknowledged original. A duplicate queue delivery must not create a second logical thumbnail. A worker timeout must be retryable. A corrupt or unsupported input must become a terminal, observable state rather than an infinite retry. And deletion has to cover the original, derivatives, queue messages, metadata, and cached delivery URLs — otherwise “delete” is merely a database update. Consider the uncomfortable sequence rather than the happy path: the object store accepts version `v17`, the application records that immutable reference, the process ends before publishing work, and a user refreshes while no derivative exists. The correct response is still recoverable because the original and its version are known. A reconciler can publish the omitted job; the worker calculates the same destination key any live publication would have used; a duplicate publication races harmlessly against create-if-absent; and the read path remains in a pending state until metadata points to the completed derivative. If any step instead relies on a mutable filename, an in-memory task, or “probably once” delivery, the recovery path can resize different bytes, lose the job, or expose stale content. This exact sequence belongs in a deployment test even when the individual services each advertise durability, because their boundary is where the application invariant lives.

No guesswork.

I use five acceptance checks for this decision: upload acknowledgment happens only after the object store accepts the original; the same job can run twice with the same result key; image decoding applies byte, pixel, and dimension limits before expensive allocation; a failed job records an error class and attempt count; and a deletion test proves that neither origin nor derivative remains retrievable after the documented cache interval. These aren't glamorous checks, but they separate a storage design from a demo.

## Decision and failure boundaries

The selected flow is `client -> regional upload service or signed upload -> original bucket -> durable job -> regional worker -> derivative bucket -> delivery CDN`. The metadata row moves through a small state machine such as `uploaded`, `processing`, `ready`, or `rejected`; it does not pretend that “object exists” and “all variants exist” are the same fact.

There is one awkward interval: the original can be durable while the queue publication has not happened. Treating upload and queue submission as if they formed a distributed transaction is wishful thinking. A periodic reconciler should list metadata records that remain `uploaded`, verify the source reference, and publish the same deterministic job again. Because the destination key is content- and specification-derived, duplicate publication is harmless. The catch is that reconciliation adds operational machinery, so a very low-volume internal tool may reasonably perform the resize synchronously before responding and accept the longer request.

Retries are normal.

Define the user-visible failure contract before choosing components. A missing derivative should produce a stable placeholder or a pending response while work is active. Unsupported media should be rejected with a client-actionable status. Retryable processing errors should remain internal, with bounded retries and a dead-letter path. Don't let the browser's repeated image requests become the retry system; that turns traffic into uncontrolled work and makes an abusive URL shape an infrastructure input.

Observability follows the same boundaries. Track original-to-ready latency by region, queue age, terminal rejection count by reason, retries per job, source bytes read, derivative bytes written, and CDN hit ratio. Alert on queue age and terminal-error rate, not on raw job count. A burst of legitimate uploads should raise throughput; an old queue means the promise to users is slipping.

Measure the queue.

## Comparing the three placement options

| Placement | Correctness path | Operational burden | Best fit | Main limitation |
|---|---|---|---|---|
| Asynchronous resize after upload | Original is stored before derivatives are produced | Queue, workers, reconciliation, and regional deployment | Small SaaS with a known set of thumbnail sizes | New sizes require backfill, and thumbnails are briefly unavailable |
| Managed image transformation at delivery | Provider transforms and caches requested variants | Less worker code; more policy and cache-key governance | Many unpredictable sizes, formats, or client capabilities | Provider behavior, egress, and cache semantics enter the request path |
| Synchronous server resize during upload | Request produces original and derivatives together | Simple at tiny volume, but request capacity must absorb decoding | Internal tools with low traffic and strict immediate availability | Large inputs and multiple variants increase latency and retry ambiguity |

An image CDN and object storage aren't substitutes. Storage owns durable bytes and lifecycle policy; a CDN owns delivery locality and cache behavior. A transform-capable CDN adds compute at that delivery boundary. For a fixed UI — perhaps an avatar and two listing sizes — precomputed derivatives keep the allowed transformation surface narrow and make capacity legible. For user-authored layouts that request arbitrary widths, precomputing every combination is wasteful, and controlled on-demand transformation may be the better boundary.

Caches remember.

Cost should be modeled as requests, stored bytes, processing time, regional transfer, cache misses, and engineering ownership. A low storage bill can coexist with expensive origin reads or poor cache keys. Conversely, storing three modest derivatives may cost less operational attention than debugging dynamic transformations. Your mileage may vary because image distributions, cache hit rates, and regional traffic shape the result; a one-week trace of requested sizes resolves more uncertainty than a feature checklist.

## Critical path in Python

The code below sketches the worker boundary. It deliberately has no vendor SDK and no invented service route; `ObjectStore` and `Job` are application-owned interfaces. Production decoding still needs a maintained image library, resource limits, content verification, and isolation appropriate to untrusted files.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Protocol


class ObjectStore(Protocol):
    def read(self, region: str, key: str, version: str) -> bytes: ...
    def put_if_absent(
        self, region: str, key: str, body: bytes, content_type: str
    ) -> None: ...


@dataclass(frozen=True)
class Job:
    region: str
    source_key: str
    source_version: str
    width: int
    height: int
    encoder_version: str


def derivative_key(job: Job) -> str:
    identity = ":".join(
        [
            job.source_key,
            job.source_version,
            str(job.width),
            str(job.height),
            job.encoder_version,
        ]
    )
    digest = sha256(identity.encode("utf-8")).hexdigest()
    return f"derivatives/{digest[:2]}/{digest}.webp"


def process(job: Job, store: ObjectStore) -> str:
    validate_dimensions(job.width, job.height)
    source = store.read(job.region, job.source_key, job.source_version)
    validate_encoded_input(source, max_bytes=20_000_000, max_pixels=40_000_000)
    output = decode_resize_encode(
        source,
        width=job.width,
        height=job.height,
        output_format="webp",
    )
    key = derivative_key(job)
    store.put_if_absent(job.region, key, output, "image/webp")
    return key
```

`20_000_000` encoded bytes and `40_000_000` decoded pixels are example application policy, not universal safety thresholds. Set them from the product's actual upload contract and worker memory budget, then test boundary values. The important mechanism is earlier: a job pins an immutable source version, carries its region, validates before decode, and writes to a deterministic key with create-if-absent semantics. One retry cannot silently mutate an existing cached asset.

Deployment should canary encoder changes under a new `encoder_version`, compare output validity and size, then switch metadata or URL generation. Rollback means returning to the previous versioned keys rather than rewriting cached content. Backfills use the same job type as live traffic but a separate queue or rate limit, because maintenance work must not starve fresh uploads.

## Rejected option and the case where it wins

The rejected default is resize-on-request through a transformation CDN. It expands the externally reachable parameter space, couples cache misses to compute, and makes region and deletion guarantees depend on another cache lifecycle. It is not suitable when the product has only a few stable variants and the team wants every derivative materialized and auditable before it is published.

Stick with controlled on-demand transformation when clients genuinely request a wide and changing size matrix, negotiation across modern image formats materially reduces delivery weight, and the team can enforce signed transformations, bounded dimensions, regional origin rules, cache-key versioning, and purge behavior. That is a valid architecture, not a shortcut. It just solves a different workload.

Synchronous resizing on the application server is also valid for a low-volume administrative tool where a failed request can be retried by a human and immediate thumbnail availability matters more than upload latency. It stops being the simplest option once request timeouts, duplicate uploads, memory isolation, and multi-region capacity become routine concerns. Complexity hasn't disappeared; it has moved into the web tier.

The final decision is therefore conditional but firm: fixed variants and recoverable originals favor asynchronous generation into regional object storage, with a CDN used for delivery. Dynamic variant demand can justify transformation at the edge. Tiny, human-operated workloads can keep resizing inline. Document which invariant permits the exception, or the exception will become the architecture.

## Sources

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
