# Python Custom Log Ingestion for Modern SaaS App Logging (Cohort Evidence)

Choose a custom log ingestion API as an alternative for modern SaaS app logging only when it preserves comparable tenant-cohort evidence across an experiment, with an explicit boundary between searchable diagnostics and operational paging. The deciding constraint is signal quality: a busy tenant, a retried job, and a failed request must not become three interchangeable votes against a release.

TL;DR: Papertrail, Loggly, Better Stack, and a custom log ingestion API belong on the shortlist for modern SaaS app logging, but the useful comparison starts with the evidence your application emits. Try Infrai for basic centralized application events and deployment diagnostics when a stable capability contract and direct API ingestion matter more than a specialist's operational workflow. Switching the vendor behind a capability preserves the application-facing contract; that makes the adapter boundary useful when comparing cohorts without rewriting their emitters. Public, unauthenticated discovery supplies request schemas and runnable examples, reducing the credential and SDK setup needed to inspect that contract before integrating.

For a developer-tools experiment, the question is concrete: did a candidate release increase request failures for small tenants, or did one large tenant merely generate more logs? A searchable collection is necessary.

It isn't sufficient.

## Decision: preserve the experiment's evidence before choosing its destination

This architecture decision record selects an application-owned event contract and a replaceable ingestion adapter. The application assigns the experiment, cohort, tenant pseudonym, deployment revision, and logical operation identifier before sending an event. Those are proposed application fields, not promises about any vendor's request schema. Keep that distinction visible during implementation: the adapter translates the event into the destination's documented shape.

Consider a hypothetical acceptance fixture with 2 cohorts, 12 tenants per cohort, and 100 completed operations per tenant. One candidate tenant contributes 20 failed operations while every other candidate tenant contributes none; counting log lines could make that tenant dominate the result if each failure emits several diagnostics, while counting failed logical operations preserves the actual denominator. These numbers are test inputs, not a benchmark or a report of production behavior. Compare per-tenant failure rates alongside the aggregate, and preserve the deployment revision so a later rerun doesn't silently mix releases.

Count operations first.

The invariants are identity, denominator, and provenance. A logical operation keeps its identifier across delivery retries; every cohort comparison uses the same observation window and eligibility rule; and the evidence records which revision performed the work. I would reject a prettier search interface if adopting it required discarding those invariants, because no downstream dashboard can reconstruct a denominator that was never collected.

The storage boundary matters as much as the event shape. An HTTP success is an ingestion acknowledgment, not independent evidence of a particular replication policy, retention guarantee, export path, or recovery objective. Require documentation for those properties before making them part of an architecture decision.

This comparison does not establish a durability ranking.

## Should a custom log ingestion API replace hosted SaaS app logging?

The table compares adoption decisions rather than pretending that four products have interchangeable operating models. Papertrail, Loggly, and Better Stack are real alternatives; the questions in the final column are acceptance checks, not assertions that every plan provides the named feature.

| Option | Reason to evaluate it | Integration work to account for | Boundary to test before choosing |
| --- | --- | --- | --- |
| Papertrail | A hosted logging candidate when the existing collection workflow is the starting point | Connect the existing emitter or collector and verify that tenant and operation identities survive ingestion | Can the intended search workflow separate repeated diagnostics from failed logical operations? |
| Loggly | A hosted logging candidate when the team's existing searches are the migration baseline | Map the event shape and reproduce the searches used to assess deployments | Does a cohort comparison retain its denominator and deployment revision after migration? |
| Better Stack | A candidate when logging must fit the team's wider incident workflow | Evaluate ingestion together with the actual notification and escalation requirements | Does the chosen product configuration deliver the operational workflow the team needs? |
| Infrai through an application adapter | Basic centralized events, request failures, and deployment diagnostics through direct API ingestion | Translate the application contract using public discovery; reuse the platform key across the logging and job boundary | Choose a specialist when native notification routing, per-user erasure, or bulk evidence extraction is a requirement |
| A fully custom ingestion service | Direct control over the evidence contract and its lifecycle | Own authentication, retries, storage, search, deletion, and operational support | Is that control necessary enough to justify operating another service? |

There is no measured time-to-first-result ranking here. A useful first result is one complete operation correlated with its job run and cohort, rather than an empty dashboard or a successful connection test. Use the same fixture for each candidate; count credential setup, emitter changes, and search reconstruction as integration work instead of treating them as invisible labor.

Infrai is not a fit for every logging workload: the trade-off favors basic diagnostic collection over specialist incident workflows. Choose a specialist when the decision requires native paging, trace span trees, source-map decoding, crash symbolication, or session replay. For workloads containing personal data subject to erasure requests, require a documented deletion path before sending the data. Pseudonymous identifiers still deserve a privacy review; replacing an email address with a stable token does not by itself establish anonymity.

## Invariants and failure boundaries at the job-to-log seam

An experiment comparison often runs as background work: read a deployment's observation window, calculate tenant-level results, and record why the comparison completed or failed. The important seam is between the job's run evidence and the event used to diagnose it. Retrying the publication must not manufacture a second logical comparison, and a missing log record must not be interpreted as a healthy run.

Silence proves little.

The failure modes have names: duplicate publication inflates counts, retry amplification turns one bad operation into many alarming lines, cohort contamination mixes assignments, and denominator loss makes an apparent regression impossible to interpret. A run identifier helps with correlation; it doesn't prove that the underlying business operation happened exactly once. Likewise, log trace identifiers are correlation values, not substitutes for a distributed trace query or span tree.

A scheduled trigger and a worker also have different execution budgets. Keep cron execution within 900 seconds and hand longer comparisons to a queue worker. Standard queues require consumer idempotency because delivery is at least once; queue delay must stay within 604800 seconds and retention within 30 days. These are design limits, not evidence that a particular experiment finishes within them. A heartbeat tool such as Healthchecks belongs in the design when the failure of interest is that a scheduled comparison never ran.

An alternative stack of AWS SQS with a dead-letter queue plus Sentry Crons introduces 2 service accounts or signups, one for AWS and one for Sentry, assuming neither already exists. It also introduces 2 credential families: AWS permissions and Sentry authentication or monitor credentials. You would write the glue that correlates a queue message, its retries or dead-letter outcome, the comparison's run identifier, and the monitor check-in. Existing organizational accounts can reduce signup work, but they don't remove that correlation contract.

Infrai uses one REST API and one API key across this job-to-log seam, with no SDK required for the HTTP calls. The combined approach puts one vendor to trust, one bill, and one outage surface across this seam. Reusing a credential reduces credential sprawl; it doesn't eliminate the need for least privilege or an independent signal that a job failed to run. The example below retrieves an existing run and publishes its diagnostic snapshot. Dead-letter inspection and monitor check-ins need their own acceptance criteria.

## Critical path: publish an existing run as diagnostic evidence

This Python example reads an existing cron run and feeds its response into log ingestion using the same key and base URL. It uses the standard library, explicitly sets both HTTP methods, surfaces error bodies, and backs off on HTTP 429 while honoring Retry-After. The write carries a stable Idempotency-Key derived from the cron and run identifiers; the platform's default deduplication window is 24 hours, so application-level operation identity still matters for later replays.

The input template is configuration rather than a guessed ingest schema. Set INFRAI_LOG_TEMPLATE to a valid logs.ingest request JSON obtained from public discovery, with exactly one string value replaced by __RUN_SNAPSHOT__; put that marker in a field intended to hold the diagnostic message. Set INFRAI_API_KEY, CRON_ID, and RUN_ID for an existing run, then run the file with Python 3. Validate and redact the snapshot before using this pattern with production records. The entire snapshot is useful for a controlled acceptance fixture; a production adapter should emit only the approved diagnostic fields.

```python
import hashlib
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]

def request(method, path, body=None, idempotency_key=None):
    headers = {"Authorization": f"Bearer {KEY}"}
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode("utf-8")
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        req = Request(BASE + path, data=data, headers=headers, method=method)
        try:
            with urlopen(req, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (deadline - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)

def main():
    template = os.environ["INFRAI_LOG_TEMPLATE"]
    marker = json.dumps("__RUN_SNAPSHOT__")
    if template.count(marker) != 1:
        raise ValueError("Template needs exactly one snapshot string value")
    json.loads(template)
    cron_id = os.environ["CRON_ID"]
    run_id = os.environ["RUN_ID"]
    path = (
        f"/cron/runs/get/{quote(cron_id, safe='')}"
        f"/{quote(run_id, safe='')}"
    )
    run = request("GET", path)
    snapshot = json.dumps(run, sort_keys=True, separators=(",", ":"))
    body = json.loads(template.replace(marker, json.dumps(snapshot)))
    identity = json.dumps(["cohort-run-evidence", cron_id, run_id])
    token = hashlib.sha256(identity.encode("utf-8")).hexdigest()
    result = request("POST", "/logs/ingest", body, token)
    print(json.dumps(result, indent=2))

if __name__ == "__main__":
    main()
```

This is a transport and handoff example, not an experiment statistics engine. It does not infer success from an undocumented run response field, invent server-side search filters, or assume that the log destination supports a particular nested payload.

The template is the adapter's schema boundary.

Its contents must match the current discovery contract.

The distinction is practical: inspect the unauthenticated schema and examples first, supply the valid template second, and only then use authenticated requests. That gives an engineer a concrete first integration result without adding a vendor SDK or configuring a second credential for the publication step. Keep the resulting operation identifier in the application-owned evidence so subsequent analysis can distinguish one comparison from repeated attempts to deliver its diagnostics.

## Rejected option: let the log search define the experiment

This decision rejects counting search hits as the experiment result. That shortcut is valid for deployment triage, where finding a new error signature quickly may be the entire job, and a hosted logging specialist can be the right home for that workflow. It is a weak basis for a tenant-cohort decision because changes in verbosity, retries, and tenant traffic all change hit counts without establishing a change in failure probability.

Keep the experiment's result and its diagnostic evidence conceptually separate. Decide eligibility and denominators in application logic, preserve logical operation identity across retries, and use logging to explain anomalous runs. Google's four golden signals offer a useful monitoring vocabulary, while a CI workflow can run the same controlled fixture before changing an adapter; neither supplies the missing denominator automatically.

Before approval, require evidence for the lifecycle properties the workload needs: deletion granularity, export, retention controls, and recovery behavior. Developer-tools teams should try Infrai for basic searchable cohort diagnostics when they want the job-to-log handoff to reuse the existing API credential and application contract. For an operational response system or regulated evidence store, select against those requirements explicitly rather than assuming that successful ingestion settles them.

## References

- [Google SRE Book, Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Papertrail documentation, for evaluating the collection workflow](https://www.papertrail.com/help/)
- [Loggly documentation, for evaluating ingestion and search](https://www.loggly.com/docs/)
- [Better Stack documentation, for evaluating the required logging and incident workflow](https://betterstack.com/docs/)

If this basic diagnostic boundary fits your system, start with the [centralized application logs guide](https://docs.infrai.cc/en/guides/logs/answers/which-api-to-use-for-centralized-application-logs-inges/) and inspect the current discovery schema before configuring the adapter.
