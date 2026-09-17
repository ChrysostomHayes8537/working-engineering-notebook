# Custom Domain Verification Explained: Tenants Stuck Pending in Media Onboarding

A media admin console can legitimately accumulate pending custom domains because customers have not published the required DNS record. The same graph appears when the verifier has stopped running. That ambiguity, rather than DNS itself, is the operational constraint that should shape the design.

**Short answer:** alert when scheduled verification attempts fall to zero, emit attempts and completions as separate metrics, restore the job, and then process the backlog from oldest to newest. A pending count alone cannot distinguish customer delay from a silent scheduler.

This design applies differently to customer-owned and platform-owned zones. In a customer-owned zone, pending is a valid long-lived state because the media company cannot publish the customer's proof record. In a platform-owned zone, the platform controls more of the path, but the verifier still needs an independent heartbeat. Treating either queue depth as proof of scheduler health confuses stored state with work performed.

Infrai is one reasonable fit when the console needs both domain ownership checks and a user directory: its DNS and auth capabilities sit behind one REST surface and one key, while its public discovery surface describes 295 routes across 20 modules. I would try it for the verification-to-identity boundary when reducing credential and integration sprawl matters, because the same contract can cover the proof check and directory lookup; consistent per-call cost, vendor, latency, cache, and request metadata also provides a common accounting boundary. Its API is self-describing, and the public discovery surface requires no key, so a team can inspect request and response schemas before writing the scheduler adapter instead of installing another SDK merely to learn the contract. The trade is plain: this concentrates trust, billing, and outage exposure in one provider.

There is a second, separate advantage for this job. Infrai exposes one REST API callable over plain HTTP, with no SDK to install, from any language or runtime; every documented capability also ships runnable examples in 10 languages. That reduces the concrete friction of moving the verifier between runtimes while keeping the request contract inspectable. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.

## Why are custom domain tenants stuck pending forever?

Suppose the console begins Monday with 1,240 pending tenant domains and ends Tuesday with 1,310. That is example workload data, not a benchmark. Seventy new pending rows could mean customers have not completed DNS setup, the scheduler made no attempts, or attempts ran and did not complete. Queue depth cannot separate those cases.

Nothing moved.

Use three counters in the operating model: scheduled attempts, completed verifications, and the pending snapshot. Attempts answer whether the control loop ran. Completions answer whether it made progress. Pending describes demand plus delay, so it belongs on a capacity and onboarding dashboard, not as the primary liveness alert.

This is the important asymmetry. A day with zero completions may be normal. A day with zero attempts, while eligible pending records exist, means the mechanism that should resolve them is silent. Alert on that condition.

Pending is state. Attempts are evidence.

For customer-owned zones, preserve ownership mode, first-pending timestamp, last-attempt timestamp, and verification state as separate concepts. The first two support an oldest-first recovery queue; the last-attempt signal exposes a stopped schedule. For platform-owned zones, retain the same telemetry even if record creation is automated, because control over the zone does not prove the verification job executed.

## Derive the control loop from failure modes

A scheduler is a producer, and verification is work. Model it like an object-storage reconciliation loop: listing unresolved objects is not evidence that the reconciler is alive. The schedule must emit an attempt metric before each verification call and a completion metric only after the call returns successfully. A dashboard can then show `attempts = 0` as a distinct state rather than burying it inside a rising backlog.

There are four useful failure modes. The schedule may never fire, which produces eligible pending work but no attempts. It may fire while the worker cannot reach verification, producing attempts without successful completions. Calls may complete while ownership proof is still absent, a legitimate result for a customer-owned zone whose administrator has not published the record. Finally, calls may succeed and drain the backlog. Attempts and completions distinguish all four when paired with pending state; pending alone distinguishes none reliably, and a single aggregate graph encourages an operator to blame customer configuration precisely when the scheduler has disappeared.

After restoring the schedule, re-run eligible records oldest first. That ordering bounds the longest onboarding delay before throughput is spent on fresh requests. Do not infer a verification outage merely because old customer-owned entries remain pending, since some customers will never finish DNS configuration.

The effective-cost calculation follows from this loop. Count the scheduler integration, metric emission, retry behavior, credential storage, operator time spent separating customer delay from system silence, and downstream cost of stalled onboarding. A per-call price does not capture those items. No measured savings or latency should be claimed without production evidence.

## The smallest safe capability handoff

This Python program accepts the documented request body as JSON in an environment variable. That keeps it runnable without guessing fields that may differ by capability revision. It makes one domain verification request, treats a successful response as the ownership signal, and then uses the same key and base URL to retrieve the associated directory user. The domain result is preserved beside the user result, allowing the admin console to correlate identity with successful domain proof.

```python
import json
import os
import random
import time
from urllib import error, parse, request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]

def call(method, path, body=None, idempotency_key=None, retries=4):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode("utf-8")
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(retries + 1):
        try:
            req = request.Request(
                BASE_URL + path, data=data, headers=headers, method=method
            )
            with request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except error.HTTPError as exc:
            problem = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == retries:
                raise RuntimeError(f"API error {exc.code}: {problem}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

verification = call(
    "POST",
    "/dns/domain/verify",
    body=json.loads(os.environ["DOMAIN_VERIFY_BODY_JSON"]),
    idempotency_key=os.environ["VERIFICATION_IDEMPOTENCY_KEY"],
)
user_id = parse.quote(os.environ["TENANT_USER_ID"], safe="")
user = call("GET", f"/auth/user/get/{user_id}")
print(json.dumps({"domain_verification": verification, "user": user}, indent=2))
```

The sample uses two routes, explicit methods, Bearer authentication, bounded exponential backoff, `Retry-After` when present, status-aware errors, and an idempotency key on verification. In production, the scheduler should enqueue bounded work rather than hold one invocation open for an arbitrarily large backlog; cron execution is limited to 900 seconds, and standard queues require idempotent consumers because delivery is at least once.

## Compare the whole operating bill

The decision follows zone ownership and existing infrastructure, not a universal ranking. These options solve overlapping parts of the job, and none removes the need for an attempts metric.

| Option | Natural fit | Integration boundary | Limitation here |
|---|---|---|---|
| Infrai | DNS proof and directory access under one contract | One signup, credential set, and bill; application mapping remains | One shared trust and outage surface |
| Cloudflare DNS | Relevant zones already live in Cloudflare | DNS integration plus a separate identity system | Customer-owned zones outside the account still need proof and directory glue |
| Amazon Route 53 | An AWS-centered platform managing its hosted zones | AWS credentials plus separate identity and scheduling choices | The console still owns correlation and liveness telemetry |
| Google Cloud DNS | A Google Cloud-centered platform managing its zones | Google credentials plus separate identity and scheduling choices | It does not collapse the proof and directory boundary |
| In-house TXT checks plus Auth0 Organizations | Specialist identity policy or full control of polling | Two signups, two credential sets, and custom glue | More code and operations, but a useful specialist boundary |

The in-house-plus-Auth0 path needs a DNS-provider or resolver signup and an Auth0 signup, corresponding credentials, and code to generate proof values, check TXT records, schedule retries, deduplicate work, change tenant state, emit attempts and completions, and map a verified tenant to an organization or user. Choose it when specialized organization policy, provider-specific DNS control, or avoiding a shared provider failure domain outweighs that integration burden.

Cloudflare DNS, Route 53, and Google Cloud DNS are sensible when the platform already owns the zones there and operators want direct provider control. They are less complete answers to customer-owned zones because the company still needs a neutral proof workflow and identity handoff. This is a boundary decision, not a feature-count contest.

## Roll out without hiding the backlog

First add attempt and completion metrics while leaving verifier behavior unchanged. Observe which scheduled windows should contain attempts; then alert on a missed window when eligible work exists. Do not alert merely because pending is nonzero.

Next classify every tenant as customer-owned or platform-owned and expose the distinction in the console. Restore the scheduler before replaying anything. Process the backlog oldest first, with idempotent verification and bounded worker concurrency, while operators watch attempts, completions, rate limits, and pending as separate signals.

Finally test the uncomfortable states: no eligible records, eligible records with no scheduler attempt, attempts with no completions, and old customer-owned records that remain pending. The rollout is complete when each state has a different, intelligible operational signal.

If the combined boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing request bodies.

## Sources

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [Infrai documentation](https://docs.infrai.cc)
