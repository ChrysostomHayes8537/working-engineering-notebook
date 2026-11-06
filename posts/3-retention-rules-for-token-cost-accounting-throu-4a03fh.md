# 3 Retention Rules for Token Cost Accounting Through One API Key Router

The bill for reviewing a game's code changes through one API key includes every inference attempt, the router's charges if applicable, and the evidence retained afterward. For a startup app, pricing comparisons need that entire workload boundary. A useful accounting identity is total inference and routing charges plus storage and retrieval charges; dividing that total by accepted reviews exposes the expense of malformed findings and repeated attempts. Which term dominates requires workload measurements. For retention specifically, an illustrative workload of 10,000 reviews per day, each retaining a 200 KB request-and-response bundle for 30 days, holds about 60 GB before replicas, indexes, and backups. These are planning assumptions, not measured traffic or published prices.

TL;DR: Put structured finding validation ahead of route selection, and give source bundles, accepted findings, and accounting records separate retention rules. One application-facing API key can simplify authentication, but it doesn't establish that different models produce interchangeable review results. For a gaming team, compare routes by their ability to return findings attached to the correct code revision, then account for all attempts required to obtain those findings. Token price alone leaves too much out.

The retention lever in that example is immediate: reducing full-bundle retention from 30 days to 3 days reduces its live footprint from roughly 60 GB to 6 GB under the same assumptions. Smaller accepted records can outlive the bundles. That change buys less historical evidence, and the rest of the design has to make that loss explicit.

## How should one API key router compare token cost across reviews?

A parsed object proves very little. A finding about an inventory update might contain the expected keys and still point at a line from an earlier revision, misclassify a harmless change as item duplication, or quote code that never appeared in the submitted diff. Structured output correctness therefore has several boundaries: syntax, schema, attachment to the reviewed revision, and the substance of the review. A storage-minded design keeps those boundaries separate because a durable record of an incorrect attachment is still incorrect.

Pin the input.

The review request should identify an immutable code revision and the exact changed lines supplied to the model. Bind the returned findings to that request in application code, using the revision stored with the request rather than trusting a revision string generated in the response. Validate filenames and line numbers against that pinned input before publishing anything. This is a proposed application contract, not a claim that an inference service verifies repository state.

There is a useful distinction in the structured-output guide listed below: schema-constrained output and a mode that returns valid JSON provide different guarantees. The guide also documents supported-schema restrictions and a refusal path. Those boundaries matter at an adapter: a response that the application cannot turn into its findings contract must remain an explicit outcome, rather than becoming an empty list that falsely tells the team there were no issues. The application must check what each candidate route actually supports; sharing credentials or a request envelope supplies no evidence of schema equivalence.

Even a schema-valid result needs review-quality testing.

For a proposed gaming fixture, a change to inventory persistence could require an expected finding about a missing ownership check, while an unrelated rendering rename should produce no such finding. These are test cases to build and label, not reported defects or claims about model accuracy. Keep the labels under version control and inspect disagreements before using the acceptance rate to compare routes.

## Three retention boundaries with different failure costs

Retaining everything indefinitely is an expensive substitute for deciding what an investigation requires. Retaining only the final prose is equally unsatisfying: a reviewer disputing an item-duplication warning needs enough context to distinguish a bad inference from a finding attached to the wrong change. The three boundaries below separate the short-lived evidence required for that investigation from the longer-lived records needed to explain what the system accepted and what work it performed.

| Retained material | Proposed retention rule | What it enables | What expiration costs |
| --- | --- | --- | --- |
| Exact submitted code and raw response | Short, explicitly configured investigation window | Inspection of truncation, malformed output, and disputed context | Exact input inspection becomes unavailable unless retained elsewhere |
| Accepted findings with revision, validator version, and request identity | Align with the team's code-review record policy | Tracing a published finding to a pinned change | Deletion removes that historical review evidence |
| Attempt outcomes, usage records, route identity, and object references | Align with accounting and evaluation needs | Counting retries and tracing the sequence of attempts | Summaries cannot reconstruct deleted source or responses |

For the illustrative workload, shrinking the full-bundle window changes retained byte-days by a factor of ten. It does not promise the same change in the invoice: storage classes, request charges, deletion behavior, replication, and retrieval charges would require separate evidence from the chosen storage contract. Nor does it reduce inference tokens already consumed. Keep that distinction in the budget review.

An object identifier also isn't a durability guarantee. Specify which write acknowledgment the review publisher requires, when readers must be able to retrieve the accepted record, and how expiry interacts with retained replicas and backups. Verify those properties against the selected storage system before relying on them. The design's consistency requirement is concrete: publishing a finding must leave a retrievable accepted record tied to the same review request; if that condition cannot be established, the workflow should expose an incomplete publication state for reconciliation.

Deletion needs references too.

An expired bundle should have an explicit expiration marker in its accounting record, so an investigator can distinguish intentional removal from an unexplained missing object. A digest can help check the identity of bytes that are still available.

It cannot recover deleted evidence.

## Validate before committing the accepted record

The following Python example implements a small proposed findings contract. Its input represents the complete response text after the adapter has separately checked completion and refusal status. The changed-line map comes from the application's pinned review input. The line limit and severity vocabulary are application choices; they aren't advertised limits of any model.

```python
import json


def reject_duplicate_keys(pairs):
    result = {}
    for key, value in pairs:
        if key in result:
            raise ValueError("duplicate object key")
        result[key] = value
    return result


def reject_constant(value):
    raise ValueError("non-JSON numeric constant")


def accepted_review(raw, *, request_id, revision, changed_lines):
    payload = json.loads(
        raw,
        object_pairs_hook=reject_duplicate_keys,
        parse_constant=reject_constant,
    )
    if type(payload) is not dict or set(payload) != {"findings"}:
        raise ValueError("unexpected response shape")
    findings = payload["findings"]
    if type(findings) is not list:
        raise ValueError("findings must be a list")
    for finding in findings:
        if type(finding) is not dict:
            raise ValueError("finding must be an object")
        if set(finding) != {"path", "line", "severity", "message"}:
            raise ValueError("unexpected finding fields")
        path, line = finding["path"], finding["line"]
        if type(path) is not str or type(line) is not int:
            raise ValueError("invalid location types")
        if line not in changed_lines.get(path, set()):
            raise ValueError("location outside pinned changed lines")
        if finding["severity"] not in ("warning", "error"):
            raise ValueError("unknown severity")
        message = finding["message"]
        if type(message) is not str or not message.strip():
            raise ValueError("empty finding message")
    return {
        "request_id": request_id,
        "revision": revision,
        "validator_version": "1",
        "findings": findings,
    }
```

This deliberately strict location rule excludes comments on unchanged lines, which can suppress useful observations about surrounding inventory logic. That is a review-policy trade-off: expand the allowed location map when the workflow supports context comments, and test the expanded policy before deployment. Passing this function still says nothing about whether an alleged ownership bypass is real.

Commit acceptance separately from individual attempts.

Assign a stable review-request identity, keep each retry as a distinct attempt, and publish only through an application operation that prevents duplicate acceptance for that request. A fallback triggered by schema rejection still belongs in the accounting record even if the next attempt succeeds. Otherwise the route with the most hidden retries can appear deceptively efficient.

Streaming adds another boundary. The server-sent events documentation describes event framing and reconnect behavior; it does not turn partial event text into an accepted findings document. Buffer the response until the adapter has established a complete result, then validate it. Test truncated streams and reconnects explicitly, including the case where completion arrives but the publishing acknowledgment is lost. Transport completion and durable publication deserve separate states.

## Measure the evidence you keep and the evidence you lose

Evaluate candidate routes on the same labeled gaming changes, schema, and validator version. Record schema rejection, refusal, interruption, location rejection, duplicate publication attempts, review latency, and total usage across attempts. Treat a valid empty findings list as its own outcome. Its accuracy depends on the labeled change, so counting it as a successful review merely because it parses would reward a route that misses every defect.

For deployment, run candidate changes against retained evaluation fixtures before allowing them to publish review comments. A change to the adapter, prompt, model selection, or location policy can change the accepted population; record those versions with the outcomes so a later comparison has a defined denominator. This adds adapter and fixture maintenance for the team, but it makes a routing decision inspectable instead of delegating correctness to a shared API surface.

Do not silently drop failures from that denominator.

An operational dashboard should connect accepted-review counts to attempt counts and retained bytes. These measurements answer different questions: a rising retry ratio suggests additional inference work, while growing bundle storage can indicate that retention is no longer following the intended policy. Neither observation alone identifies the cause. Preserve enough request identity to trace the relevant state transitions without copying code into every log entry.

The deliberate stopping point is full request-and-response retention after the investigation window. Keep accepted findings and their provenance according to the review policy, retain accounting outcomes for the chosen reporting period, and stop keeping expired source bundles merely because storage is convenient. If an old review is disputed after expiration, the team may be able to inspect the accepted finding and recover the repository revision while still being unable to reconstruct the exact prompt, omitted context, or raw response. Repeating inference would be a new attempt, not proof of what happened earlier.

That is the retention decision to approve: a bounded investigation window with a known loss of evidence. A single API key is useful only inside a workflow that preserves structured correctness and makes that loss visible.

## References

The structured-output guide supports the distinctions between JSON validity, schema adherence, supported-schema boundaries, and refusals. The event-stream documentation supports the discussion of framing and reconnection. The retention arithmetic, acceptance contract, and lifecycle rules above are proposed design choices with explicitly stated assumptions.

## Further reading

- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- OpenAI, Structured Outputs guide: https://platform.openai.com/docs/guides/structured-outputs
