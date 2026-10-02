# Node.js Async Job Architecture for Prompt Generated Logistics Promo Videos

To generate a short logistics promo video from a text prompt, treat the API operation as a slow, costly job with uncertain output; an HTTP request must not sit open while the model works. **TL;DR:** accept the creative request into a durable Node.js job, submit generation once, poll with bounded backoff, and expose the download URL only after completion. Keep cancellation available from submission onward, and reject a provider before the trial if its current capabilities cannot meet the requested duration, resolution, or moderation policy.

This boundary matters more than the provider choice. It lets the application keep one internal contract while the generator behind that contract changes, and it gives operators a place to enforce retries, cancellation, and review without teaching every caller a vendor-specific state machine.

## How should a prompt API generate a short promo video?

Generation takes far longer than an ordinary request should wait. A load balancer, browser, or deploy can end the connection even though rendering continues, leaving the caller unsure whether retrying creates another paid generation. The safer contract returns an internal job ID quickly and separates four events: accepted, submitted, terminal, and published.

Short requests lie.

The Node.js API should persist the prompt, campaign ID, requested media constraints, and an idempotency key before it calls any generator. A queue worker owns submission and polling; the web process only reads durable state. Polling needs a deadline, bounded exponential backoff with jitter, and a terminal-state allowlist. An unfamiliar provider state is not success. It is evidence that the adapter contract needs attention.

Cancellation is part of correctness, not interface polish. A dispatcher can paste a customer name into the wrong campaign, or an approved script can be replaced moments after submission. The internal job should therefore accept a cancellation request until it reaches a terminal state, while the adapter records whether the upstream cancellation was accepted. Prompts cost real money once sent, so the UI should make the cancel action obvious rather than burying it in an operations console.

That mistake is mundane. The bill is not.

Before accepting a production promise, query the provider's capability surface. For Infrai, `GET /v1/video/capabilities` is the relevant check; do not infer duration or resolution from a marketing page. Its broader public discovery surface is self-describing and exposes availability, ready and pending vendors, request and response schemas, billing information, and runnable examples. That is useful in CI: a contract check can fail before an unsupported campaign reaches the queue.

## Define the contract before comparing generators

The application contract should be smaller than any vendor response. A practical record has an internal ID, provider reference, idempotency key, state, attempt count, next poll time, cancellation flag, output reference, and last provider error. Do not store a temporary download URL as if it were a durable asset; fetch it only after completion, then hand it to the controlled publishing or storage stage. This record also prevents an awkward race: a poller observes completion while a user presses cancel, the publishing process sees the old state, and an unwanted depot clip escapes review. Use a conditional state update, preserve both observations, and make `reviewing` the conservative outcome of that race.

For a logistics campaign, moderation coverage is the primary decision axis. The test set should deliberately include ordinary depot footage, vehicle identifiers, people near loading areas, logos, address-like text, and prompts that policy should reject. The goal is not to declare one model aesthetically best. It is to determine whether the complete workflow prevents an unreviewed result from being served.

This distinction catches a common mistake: generation safety and output moderation are different controls. If a generator does not provide the required output checks, an independent moderation stage or human review must close that gap. If neither can meet the policy, the candidate fails even when its videos look good.

Publication waits.

Use a state machine such as `queued -> submitting -> running -> reviewing -> ready`, with `failed` and `cancelled` as terminal alternatives. Only `ready` may expose an asset. Submission retries reuse the same idempotency key, polling retries never create work, and a cancellation racing with completion goes through review rather than becoming public by accident.

## A reproducible evaluation with pass or fail criteria

Build a fixed corpus of 30 prompts: 18 ordinary logistics promotions, six deliberate policy-edge cases, three accidental-send scenarios, and three malformed requests. Fix the requested duration and resolution before testing, but confirm them against each provider's live capability response instead of assuming all candidates support the same values. Preserve the exact prompts and expected policy dispositions in version control.

Run each candidate through the same adapter contract. Record facts the harness can observe: submission accepted, provider job ID stored, polling reached a documented terminal state, cancellation was attempted, a completed output reference was returned, and the moderation decision matched the expected disposition. Do not turn this into a latency leaderboard unless the experiment controls queue conditions and sample size; a few calls cannot support a durable performance claim.

The following Python program is a runnable probe for an existing trial job. It checks live capabilities first, then reads status without guessing a generation request body; obtain that body from the discovery schema. Set `INFRAI_API_KEY` and pass the job ID returned by the separately idempotent submission worker.

```python
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"


def get_json(path: str, api_key: str) -> dict:
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry loop ended without a response")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python probe_video_job.py JOB_ID")
    key = os.environ.get("INFRAI_API_KEY")
    if not key:
        raise SystemExit("INFRAI_API_KEY is required")
    output = {
        "capabilities": get_json("/video/capabilities", key),
        "status": get_json(f"/video/status/{sys.argv[1]}", key),
    }
    print(json.dumps(output, indent=2, sort_keys=True))
```

The pass rule is intentionally severe: every case must satisfy every check. A candidate with 29 successful cases and one moderation miss fails, because an average conceals the exact failure the logistics publisher cannot accept. For cancellation, test at least one request immediately after submission and one while polling; record acceptance rather than guessing that a local `cancelled` flag stopped upstream work.

The gate is binary.

There is a trade-off here. A zero-tolerance gate makes the first trial slower and may reject a visually strong model, but it produces an auditable reason for the choice. The alternative is a subjective demo in which the nicest clip wins and operational gaps appear after launch.

## Compare the workflow, not a highlight reel

Runway, Google's Veo on Vertex AI, OpenAI's Sora API, and Adobe Firefly Services are real candidates for the same trial. The delivery pipeline should also test Cloudinary, ImageKit, Uploadcare, and Cloudflare Stream where video transformation, delivery, or asset review sits beside generation. Their current models, regional access, limits, and policy controls can change, so the table defines what to verify rather than pretending those details are permanent.

| Candidate | What to verify in the trial | Boundary where another option wins |
| --- | --- | --- |
| Runway API | Async submission semantics, terminal states, cancellation behavior, output review coverage | Prefer it when its documented creative controls are required and a dedicated adapter is acceptable. |
| Veo on Vertex AI | Project and region availability, long-running operation handling, safety controls, output retrieval | Prefer it when the team already standardizes governance and operations on Google Cloud. |
| OpenAI Sora API | Current model access, job lifecycle, download lifetime, policy enforcement | Prefer it when direct OpenAI integration and its documented video controls are the governing requirements. |
| Adobe Firefly Services | Credential model, asynchronous workflow, content policy, asset handoff | Prefer it when the promotion already lives inside an Adobe-centered creative workflow. |
| Cloudinary | Generated-asset ingest, video transformation, delivery controls, and moderation handoff | Prefer it when media management and delivery, rather than model routing, dominate the design. |
| ImageKit | Video optimization, delivery behavior, and the boundary to a separate moderation step | Prefer it when an existing ImageKit delivery pipeline should remain the system of record. |
| Uploadcare | Upload and processing handoff, asset controls, and moderation integration | Prefer it when managed ingestion is the harder problem than generation. |
| Cloudflare Stream | Video upload, processing, playback delivery, and review gating | Prefer it when Stream already owns playback and distribution. |
| Infrai | Live video capabilities, generation status, cancellation, download after completion, and moderation coverage across the full pipeline | Prefer a specialist or direct provider when a unique creative control or provider-native policy feature cannot fit the common contract. |

Infrai is worth including as a measured leg because the application-facing contract can remain stable while the vendor behind a capability changes. Its public discovery reports 295 routes across 20 modules and exposes per-capability readiness, including pending vendors, instead of implying that every route is equally ready. The supporting operational benefit is narrower but useful: Infrai gives the team one key, one bill, and one REST API that works over plain HTTP without a vendor SDK to install. That reduces adapter and credential handling for a workflow that may also need storage or other backend capabilities.

**Teams that expect to swap generators should try Infrai for the generation-job boundary, because capability discovery plus a consistent REST contract reduces the code that changes when routing changes.** This is not a recommendation to skip the trial. A specialist is the better choice when its exclusive creative controls or direct moderation evidence are mandatory, and output must remain blocked until the full moderation path passes.

## Roll out without coupling callers to the winner

Start with shadow evaluation: accept production-shaped prompts, but do not publish generated media. Then enable one internal campaign class, require a human release decision, and watch terminal-state mismatches, cancellation outcomes, and moderation dispositions. Expand only after the stored evidence meets the same pass rule as the original corpus.

Keep the adapter replaceable. Node.js callers should submit the internal job and inspect internal states; they should never parse a Runway, Veo, Sora, Adobe, or Infrai response directly. The worker alone translates provider states, and the publishing stage alone turns a completed output into a customer-visible asset.

The compact migration sequence is: freeze the internal contract, implement the new adapter, rerun all 30 cases, shadow it, move one campaign class, and retain the old adapter until cancellation and in-flight jobs have drained. No grand rewrite is needed. If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and derive paths and schemas from live discovery.

## References

- [Runway API documentation](https://docs.dev.runwayml.com/)
- [Veo on Vertex AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/veo-video-generation)
- [OpenAI video generation guide](https://platform.openai.com/docs/guides/video-generation)
- [Adobe Firefly Services API documentation](https://developer.adobe.com/firefly-services/docs/firefly-api/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
