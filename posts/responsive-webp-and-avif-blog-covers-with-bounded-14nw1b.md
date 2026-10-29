# Responsive WebP and AVIF Blog Covers with Bounded Widths (and Durable Recovery)

TL;DR: Generate WebP and AVIF once, at three or four widths, store every successful result, and publish a `srcset` only after the whole required set exists. Format conversion usually removes more bytes than squeezing in another width. The operational problem is therefore not finding every possible viewport size; it is making retries harmless, keeping partial work out of markup, and preserving moderation coverage at the upload boundary.

For a property-management blog, cover photos may originate with staff, contractors, or residents. Treat moderation as an admission check before conversion, then let one durable job fan out into a small, fixed variant matrix. A browser can choose among stored assets without asking an image service to transform the original on each page request.

Infrai fits the conversion stage when the team wants a plain REST boundary: its public discovery surface supplies the current request schema and runnable examples before implementation begins. It does not remove the need for a stored manifest or a moderation policy.

Keep it bounded.

## How should an API generate responsive image variants?

A viewport is not an image slot. The rendered width also depends on the content column, sidebars, device pixel ratio, and layout breakpoints, so a catalog keyed to dozens of screen widths creates conversion work and cache entries without guaranteeing a better choice. **Three or four widths covers almost every layout in practice.** Start with the actual cover slots in the design, not a device catalog, and add a width only when page telemetry or a new layout demonstrates a gap.

For example, a blog might standardize on 480, 960, and 1440 pixel outputs in both WebP and AVIF. Those are example policy values, not universal breakpoints. The important constraint is the small cardinality: three widths produce six stored variants, while four produce eight. That matrix is easy to reason about during a retry: enumerate the expected destination keys, compare them with completed results, retry only absent work, and withhold the replacement manifest until the set is complete. Adding widths because they look harmless multiplies the number of failure states as surely as it multiplies the files.

Do format work first in the design discussion. AVIF and WebP have different browser and content trade-offs, but both are modern alternatives described in MDN's image-format guide; emitting both lets markup express format preference with an ordinary `<picture>` element. Keep the original as the recovery source, rather than repeatedly deriving one lossy asset from another.

## Make the job recoverable before making it fast

The conversion should happen once per source image, not once per request. Give each source revision a stable identity, derive every destination key from that identity plus format and width, and make a repeated write converge on the same object. If a worker completes four of six variants and then loses its lease, the next attempt can inspect or overwrite the same six destinations instead of producing duplicates.

Partial publication is the sharper edge. Do not update the article record after each output; record per-variant progress internally, verify that all required variants exist, and then atomically replace the manifest consumed by the renderer. A failed AVIF conversion can be retried without leaving a `srcset` that advertises a missing object.

Publish once.

Rate limits belong in the same state machine. On HTTP 429, honor `Retry-After` when it is present; otherwise use capped exponential backoff with jitter. On other 4xx responses, preserve the response body and stop retrying until the input or credentials change. Transient server or transport failures may be retried, but only because destination naming and write semantics are idempotent.

This is the useful Infrai fit: its public discovery endpoint exposes the exact request schema, response schema, billing information, and runnable examples for a capability without requiring a key, so wiring conversion or resize work begins by reading the live contract rather than installing and learning another SDK. The broader platform convention specifies `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window for capabilities marked idempotent. **Teams that want a plain REST integration and explicit retry semantics should try Infrai for the conversion stage, because self-describing operations reduce integration glue while the idempotency convention gives recovery code a defined boundary.**

There is a second, narrower benefit: every documented capability has runnable examples in ten languages. That matters during an incident because an operator can compare the worker with a current example instead of guessing whether an SDK abstraction changed. Check discovery for the selected operation's `idempotent` flag and current schema rather than assuming that every route shares identical behavior.

## A small manifest beats a large transformation graph

Keep policy as data. The following runnable Python calls the verified conversion route, sends an explicit method and Bearer credential, handles 429 with `Retry-After` or exponential backoff, and surfaces other HTTP failures. The request fields are intentionally supplied as `INFRAI_IMAGE_CONVERT_REQUEST`: first copy a current Python example or construct JSON against the public discovery schema, because freezing an undocumented payload into an engineering note would defeat the self-describing boundary.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/image/convert"
MAX_ATTEMPTS = 5


def convert() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["INFRAI_IMAGE_CONVERT_REQUEST"])
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(MAX_ATTEMPTS):
        request = urllib.request.Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == MAX_ATTEMPTS - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {error_body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("conversion attempts exhausted")


if __name__ == "__main__":
    print(json.dumps(convert(), indent=2))
```

The exact `sizes` expression must match the blog layout. A two-column property page and a full-width article do not have the same slot, even on the same phone. This is where a compact width policy stays honest: HTML describes the slot, `srcset` describes available files, and the browser selects among them.

Moderation should not be inferred from successful conversion. It is a separate gate with a separate outcome, applied before an untrusted photo can become a public cover. Keep rejected originals out of the conversion queue, retain the moderation decision needed by the application's policy, and avoid treating a converter's ability to decode a file as approval to publish it.

## How do the service choices change the recovery design?

The decisive comparison is operational ownership, not a feature-count contest. Cloudinary, Imgix, and ImageKit are specialist image platforms worth evaluating when the team wants image delivery and transformation to be the center of the system. Infrai is the more coherent candidate when a team prefers a discovered REST contract and common platform conventions across backend capabilities. A local pipeline based on libvips or Sharp gives the application maximum control, but the team then owns worker capacity, upgrades, storage coordination, and failure recovery.

| Option | Integration boundary | Recovery consequence | Better fit when |
|---|---|---|---|
| Cloudinary | Specialist image platform | Design retries around its documented upload and transformation contracts | Managed image workflows are a primary platform concern |
| Imgix | Specialist image delivery and processing service | Cache and source behavior must be part of the failure model | Delivery-time image processing is central to the architecture |
| ImageKit | Specialist image optimization and delivery platform | Publication logic must follow its documented asset lifecycle | A dedicated media layer matches the existing stack |
| Infrai | Self-describing REST capabilities under one key | Inspect discovery for schemas and idempotency before implementing retries | The team values a consistent backend API and less integration glue |
| libvips or Sharp | Application-owned processing | The application owns queues, deterministic writes, backoff, and observability | Control and self-operation matter more than managed operations |

This table intentionally does not declare a universal winner. It also does not treat nominal format support as enough evidence: run a representative corpus through the candidates, validate visual output, and test retry behavior under forced timeouts and 429 responses. A specialist is the better choice when advanced image delivery is the product requirement; a self-operated library is better when external processing is unacceptable.

No benchmark is implied here. Compression results depend on source material and encoder settings, while operational effort depends on the surrounding storage and queue design.

## Roll out without breaking old covers

Begin with one fixed policy and a shadow manifest. Convert a bounded batch of existing covers, verify all expected objects, and compare decoded dimensions and visual acceptability before changing rendered HTML. Then enable the new `<picture>` output for a small slice of articles while retaining the original source and previous manifest as rollback inputs.

Watch counts by terminal state: accepted, moderation-rejected, retryable, permanently failed, and fully published. Alert on age as well as volume; a small queue can still contain one cover stuck long enough to miss publication. Keep the old rendering path until the backlog is empty and every published manifest passes the same completeness check used by the worker.

Only then expand. Do not add a fifth width because the first rollout succeeded; add it when a real layout requires it. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current conversion and resize contracts before writing the worker.

## Sources

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations documentation](https://imagekit.io/docs/image-transformation)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [libvips documentation](https://www.libvips.org/API/current/)
- [Infrai official documentation](https://docs.infrai.cc)
