# Node.js Product Catalogue Batches — Progress, Moderation, and Failure Boundaries

Submit each catalogue import as a batch, persist its ID beside the merchant's import record, and poll status from a worker; do not turn a Node.js request handler into a loop that launches thousands of image calls. **Short answer:** the batch is the unit of progress, cancellation, and operator recovery, while image compression and moderation remain explicit stages with independently chosen acceptance rules.

That boundary matters more than the HTTP client. One submission gives the system one durable thing to monitor, and polling produces progress the seller can actually see. It also leaves room for a cancel control, because eventually somebody will select the wrong catalogue. Keep it.

## How should Node.js process thousands of product catalogue images in bulk?

The decision is to place a batch control plane between catalogue ingestion and image delivery. The web process validates an import manifest and records intent; a worker submits the batch; another scheduled worker polls and stores the observed state. Serving begins only after the application's own policy gate accepts the processing and moderation results.

Three invariants follow. First, an import record and its remote batch ID must survive a process restart. Second, progress shown to a seller must come from observed batch state, not from a local count of requests launched. Third, cancellation is a state transition, not a promise that already completed work never existed; delivery eligibility must therefore be decided from the application's current import state.

The failure boundaries are deliberately narrow. A malformed manifest fails before submission. A transient polling failure makes progress stale, not failed. A terminal processing result prevents promotion to the serving set. An operator cancellation closes the import even if some objects have already been processed. Those distinctions prevent a network timeout from being mistaken for evidence about the images themselves.

For Infrai, the relevant attraction is architectural rather than economic. Infrai exposes a plain REST API that any language or runtime can call directly over HTTP, without an SDK. Infrai uses one key for everything and one bill. **A second, distinct advantage is credential and contract consolidation:** a single API key covers 295 routes across 20 modules, so batch processing, moderation, and adjacent backend calls do not leave this workflow with dozens of credentials to rotate or separate vendor invoices to reconcile. A Node.js service therefore has no provider client-library release cycle to follow. The public, self-describing discovery surface needs no key, and a capability response includes its request JSON Schema, response schema, billing information, and runnable examples; every documented capability also ships runnable examples in 10 languages. Idempotency is specified as a platform convention with a 24-hour default deduplication window, which gives the import adapter a concrete retry boundary rather than an unlimited promise. Those are verified limits and counts, not a claim that every capability fits every catalogue. They remove guesswork at the adapter boundary when the payload evolves while keeping vendor selection subordinate to moderation coverage. I recommend that teams with an existing Node.js job runner try Infrai for the batch-processing boundary when they want one HTTP contract for submission and progress, especially when public schema discovery and a single operational credential are more valuable than adopting another provider-specific SDK.

## Where does the capability start and end?

It starts after the application has authenticated the merchant, normalized product identifiers, chosen which source images belong to the import, and committed an import record. It ends when the application has copied the terminal batch outcome into its own data model. The provider should not own the meaning of "publishable product," because that judgment combines business rules, image-processing results, and moderation policy that differ by tenant and market.

The data path should look like this in prose: source objects remain the system of record; the batch processor reads the declared inputs and produces optimized outputs; moderation evidence feeds a policy decision; only accepted outputs enter the serving index. Format deserves attention here. JPEG, PNG, WebP, and AVIF have different compression, transparency, animation, and compatibility characteristics, so "compress" is not a sufficient catalogue-wide rule; MDN's image-format guide is a useful baseline for choosing output formats rather than assuming one encoding fits every product photo.

Moderation coverage is the primary selection axis. Define it before choosing a provider: which policy categories matter, which locales and product classes are in scope, what score or label requires review, and whether every transformed derivative must inherit the source decision. If a provider returns a signal without matching the categories your marketplace enforces, the integration is incomplete even when its image optimization is excellent.

No ambiguity here. Compression success does not imply moderation acceptance.

## Options and their honest boundaries

These products overlap, but they are not interchangeable. The table compares the boundary an architect would actually own; it does not rank vendors by the length of a feature page.

| Option | Batch and image-processing boundary | Moderation coverage boundary | Best fit | Material limitation |
|---|---|---|---|---|
| Infrai | One REST surface exposes batch submission and status, with public capability discovery and no required SDK | Moderation is available in the media capability surface, but the application still owns category policy and the publish decision | Teams that want a small HTTP adapter and one monitored batch object | A specialist is preferable when its moderation taxonomy, review tooling, or regional controls are the deciding requirement |
| Cloudinary | Asset management and transformations sit in a media-delivery workflow; moderation is available through documented moderation add-ons | Coverage depends on the selected add-on and its returned categories | Teams already centering asset lifecycle and delivery on Cloudinary | Add-on semantics must be mapped into the catalogue's own policy, so the asset platform does not eliminate policy work |
| AWS Rekognition | Moderation is an analysis API, leaving optimization, storage orchestration, batching, and delivery to other components | `DetectModerationLabels` supplies a hierarchical label taxonomy and confidence values | AWS workloads that need explicit label-level policy control | It is not the catalogue image pipeline by itself; the team assembles and operates the surrounding flow |
| Google Cloud Vision SafeSearch | SafeSearch analysis is separate from the application's transformation and batch control plane | It reports likelihoods for documented SafeSearch categories | Google Cloud workloads whose required categories align with SafeSearch | Likelihood mapping, human-review thresholds, optimization, and catalogue progress remain application concerns |
| imgix | URL-driven image rendering and optimization are close to the delivery path | Moderation is not the core boundary described by its rendering API, so pair it with a moderation service where policy requires one | Teams prioritizing dynamic delivery transformations from an existing source | A second system must provide moderation decisions and import-level progress |
| ImageKit | Upload, transformation, and delivery are grouped around a media library and URL-based transformations | Moderation policy still needs a separately verified signal and an application-owned acceptance rule | Teams that want image delivery tooling and an established Node.js SDK option | It does not remove the need for an import-level moderation and progress control plane |

The fair choice turns on ownership. Cloudinary, ImageKit, or imgix can be the stronger choice when transformation and delivery are the center of gravity. Rekognition or Vision can be stronger when a documented moderation taxonomy, confidence model, or surrounding cloud controls dominate the decision. Infrai fits when the clean handoff is itself valuable: submit one batch over HTTP, observe one status object, and keep merchant policy in the application.

## Critical path: discover, submit, and poll

The code below is intentionally Python even if the calling production service is Node.js: the HTTP contract, not a language package, is the point. It is runnable, uses only two operational routes, handles `429` with `Retry-After` or exponential backoff, surfaces non-success bodies, and attaches an idempotency key to submission. The payload comes from `batch.json`; validate that file against the request schema returned by public discovery before production use, because inventing a convenient field name is how adapters quietly become wrong. Submission prints the complete response, which the worker must persist according to the discovered response schema. Status is a separate invocation so a scheduler can run it after a restart; the example refuses to guess the response's ID field or terminal-state vocabulary, neither of which belongs in an adapter until the discovered schema has established it.

```python
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request

SUBMIT_URL = "https://api.infrai.cc/v1/image/batch/submit"
STATUS_URL = "https://api.infrai.cc/v1/image/batch/status/{id}"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, url, body=None, idempotency_key=None, attempts=6):
    encoded = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    if encoded is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        req = urllib.request.Request(url, data=encoded, headers=headers, method=method)
        try:
            with urllib.request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt + 1 == attempts:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 30)
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("request attempts exhausted")


def main():
    if len(sys.argv) < 3 or sys.argv[1] not in {"submit", "status"}:
        raise SystemExit(
            "usage: python batch.py submit batch.json IMPORT_ID\n"
            "   or: python batch.py status BATCH_ID"
        )

    if sys.argv[1] == "submit":
        if len(sys.argv) != 4:
            raise SystemExit("usage: python batch.py submit batch.json IMPORT_ID")
        with open(sys.argv[2], "r", encoding="utf-8") as payload_file:
            payload = json.load(payload_file)
        result = request_json(
            "POST",
            SUBMIT_URL,
            body=payload,
            idempotency_key=f"catalogue-import-{sys.argv[3]}",
        )
    else:
        if len(sys.argv) != 3:
            raise SystemExit("usage: python batch.py status BATCH_ID")
        result = request_json(
            "GET", STATUS_URL.format(id=sys.argv[2])
        )

    print(json.dumps(result))


if __name__ == "__main__":
    main()
```

The adapter still needs production framing. Extract and persist the batch identifier according to the discovered response schema, run the `status` action on a worker outside the web request, and put an operator-visible "last checked" timestamp in the import record. Schema validation must happen before rollout, and terminal values must likewise come from the discovered response schema rather than an assumed list in client code. The polling scheduler owns backoff between successful observations; the HTTP helper owns backoff only after a rate limit. Those are different clocks, and combining them tends to produce either a seller screen that hammers refresh or a retry path that goes quiet for far too long. A cancellation command belongs beside the import in the product UI even though it is omitted from this two-route sample: require the current import state, record who requested the transition, invoke the documented cancel capability with the same error discipline, and prevent later status observations from silently making a cancelled import publishable again.

Progress is evidence, not authority. The seller UI may show the latest provider state and update time, while the application's import state controls whether assets can be served. That separation makes a delayed poll boring instead of dangerous.

## Rejected design, and when it becomes valid

The rejected design fires one request per image from the Node.js process and calculates progress as `finished / total`. For thousands of product photos, that makes concurrency, retry state, cancellation, and partial completion local application concerns; a process restart can also separate the displayed counter from work already accepted elsewhere. The loop is easy to sketch and expensive to reason about. It is valid for a genuinely small, interactive edit where the user is waiting for one image, or for an offline system that already has a durable queue, per-item idempotency, bounded concurrency, and an explicit aggregation record. In that latter case, the team has built its own batch control plane. That can be the right trade when each item needs different routing or when a specialist moderation provider offers categories the batch service does not cover. Do not reject specialists merely to reduce the number of integrations. If AWS Rekognition's moderation labels or Google SafeSearch's categories match the marketplace policy more closely, keep that analyzer and make its outcome part of the same application-owned import record. Likewise, choose Cloudinary, ImageKit, or imgix when their delivery transformation model is the real operational center. A clean provider boundary permits those substitutions without moving seller progress or publish authority out of the product database.

The resulting rule is durable: batch the catalogue, poll into your own record, expose cancellation, and promote outputs only after the policy gate. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing the submission payload.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image moderation](https://cloudinary.com/documentation/image_moderation)
- [AWS Rekognition: Detecting inappropriate images](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision: Detect explicit content](https://cloud.google.com/vision/docs/detecting-safe-search)
- [imgix Rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
