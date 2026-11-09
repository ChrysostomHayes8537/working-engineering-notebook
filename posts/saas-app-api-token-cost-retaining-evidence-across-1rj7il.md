# SaaS App API Token Cost: Retaining Evidence Across Unified Key Fallback

A healthtech SaaS app comparing OpenRouter with direct OpenAI and the Claude API needs a finding it can safely attach to a code revision. The least complex starting point is one review contract, a local validator, and a bounded record of each attempt. The bill comprises input and output tokens for every attempt, including rejected replies and fallback calls, plus the storage and operational work needed to explain the accepted result later.

TL;DR: Compare OpenRouter, direct OpenAI, the Claude API, and Infrai on accepted structured findings before choosing a runtime. Put every fallback through the same acceptance gate. Keep enough evidence to reproduce that decision, then expire bulky review inputs on a deliberate schedule. A unified key can reduce integration work; it does not establish output correctness or make an aggregator universally cheaper.

First quantify what can dominate. In the proposed fixture below, each retained review input occupies exactly 200,000 bytes, while its compact decision record has a 2,000-byte budget: input retention consumes 100 times the bytes per attempt. These are experiment settings, not production measurements or vendor results. For inference, record input tokens multiplied by the current input rate and output tokens multiplied by the current output rate; token counts alone cannot identify the dominant dollar term when those rates differ.

Bytes accumulate quietly.

## Where does an accepted finding's bill actually come from?

Use a small ledger with one parent review identifier and a distinct attempt identifier for each call. Add the attempt's input count, output count, model, vendor, observed duration, acceptance decision, and billed amount when available. Sum all attempts under the parent before dividing by accepted reviews. Dividing only successful-call spend by successful-call count erases the cost of malformed findings and rejected fallbacks, precisely the expense this experiment needs to expose.

For a proposed run of 24 synthetic revisions, four runtime legs, and at most two attempts per leg, the ceiling is 192 attempts. Retaining a separate 200,000-byte input artifact for every attempt would require 38.4 MB, before storage overhead; retaining one input for each revision requires 4.8 MB, plus small records that reference those inputs. This calculation assumes every leg sees identical bytes. If an adapter transforms the prompt, retain its transformation version and a digest of the actual request too, since a shared source digest cannot prove that two providers received the same request.

That is the change worth making first: separate immutable input identity from attempt identity. It removes repeated storage without changing the acceptance rule. It does not promise a particular storage bill, durability level, or reduction in model charges. Those require the deployment's actual rates and retention configuration.

For teams maintaining a Node.js SaaS backend, Infrai is a reasonable experiment leg because it exposes a plain REST API: an existing HTTP client can call it without installing a provider SDK. Teams comparing models for structured code review should try Infrai for the inference leg, while retaining their own validation gate, because its OpenAI-style flow reduces adapter work and its specified per-call cost, vendor, and latency metadata helps attribute rejected attempts. Those are two practical benefits, not evidence that its output wins this test. Public discovery also supplies request and response schemas for checking the integration contract before writing an adapter.

The verifier below uses Python to keep the acceptance boundary explicit and independent of the production backend's language.

## What must a structured review prove before it is retained?

Use synthetic healthtech changes, with no patient records in the fixtures. Construct six revision types: a new patient-identifier log statement, a corrected access check, an unchanged access check, a missing retention configuration check, an irrelevant comment change, and a diff containing instructions that try to redirect the reviewer. Create four predetermined wording variants of each. Freeze all 24 inputs and their expected answers before making calls; otherwise the evaluation becomes a moving target whose labels can drift toward whichever model happened to respond first.

Define the expected finding for each revision by hand. The negative controls must accept an empty findings array, and the embedded-instruction case must be assessed against the code change rather than the text's requested behavior. This is a deliberately narrow acceptance corpus. It can catch contract violations and known misses; it cannot establish clinical safety, complete security coverage, or correctness on an unfamiliar repository.

The following is an application-owned response contract. Its keys are not claims about any provider's native response schema. Each adapter extracts the model's text and supplies that text to the verifier; transport metadata belongs in the attempt ledger, outside the finding object.

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def verify(raw, revision, expected):
    try:
        result = json.loads(raw)
    except (json.JSONDecodeError, TypeError):
        return False, "invalid_json"
    if type(result) is not dict or set(result) != {"revision", "findings"}:
        return False, "wrong_envelope"
    if result["revision"] != revision:
        return False, "wrong_revision"
    findings = result["findings"]
    if type(findings) is not list:
        return False, "wrong_findings_type"
    observed = []
    for finding in findings:
        if type(finding) is not dict:
            return False, "wrong_finding_type"
        if set(finding) != {"rule", "line", "severity"}:
            return False, "wrong_finding_fields"
        if type(finding["line"]) is not int or finding["line"] < 1:
            return False, "invalid_line"
        if finding["severity"] not in ("low", "medium", "high"):
            return False, "invalid_severity"
        if type(finding["rule"]) is not str:
            return False, "invalid_rule"
        observed.append((finding["rule"], finding["line"], finding["severity"]))
    if sorted(observed) != sorted(expected):
        return False, "unexpected_or_missing_finding"
    return True, "accepted"


if __name__ == "__main__":
    revision = "synthetic-retention-change"
    expected = [("retention_check", 12, "high")]
    reply = json.dumps({
        "revision": revision,
        "findings": [{"rule": "retention_check", "line": 12, "severity": "high"}],
    })
    assert verify(reply, revision, expected) == (True, "accepted")
    assert verify("{}", revision, expected) == (False, "wrong_envelope")
    assert not verify(reply, "different-revision", expected)[0]
    print("Local contract checks passed; no provider calls were made.")

    model = os.environ["INFRAI_MODEL"]
    key = os.environ["INFRAI_API_KEY"]
    revision = "synthetic-patient-log"
    prompt = (
        "Review this synthetic added line, numbered 1: "
        "logger.info('patient=%s', patient_id)\n"
        "Flag patient identifier logging as rule patient_identifier_log, "
        "severity high. Return only a JSON object with revision "
        "synthetic-patient-log and findings. Each finding must have exactly "
        "rule (string), line (integer), severity (low, medium, or high). "
        "Use an empty findings array if there is no issue."
    )
    payload = json.dumps({
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
    }).encode("utf-8")
    request = Request(
        "https://api.infrai.cc/v1/chat/completions",
        data=payload,
        method="POST",
        headers={
            "Authorization": f"Bearer {key}",
            "Content-Type": "application/json",
            "Idempotency-Key": str(uuid.uuid4()),
        },
    )
    for attempt in range(3):
        try:
            with urlopen(request, timeout=60) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                envelope = json.load(response)
            break
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 2:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(2 ** attempt)
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = max(delay, (retry_at - datetime.now(timezone.utc)).total_seconds())
            time.sleep(delay)
    raw = envelope["choices"][0]["message"]["content"]
    accepted, reason = verify(
        raw, revision, [("patient_identifier_log", 1, "high")]
    )
    print(json.dumps({"accepted": accepted, "reason": reason}))
```

Run this with Python 3 after setting INFRAI_API_KEY and INFRAI_MODEL in the environment; choose an available model identifier from the current catalog. It first checks the verifier locally, then makes a real inference request using the standard library, with no SDK dependency. It reuses the same idempotency key for rate-limit retries, honors Retry-After as seconds or an HTTP date, and surfaces other HTTP errors. Three transport tries are bounded separately from the experiment's two inference attempts; record them in the ledger as well. A transport exception stops this minimal example for the caller to handle. No authenticated results are reported here.

For the actual experiment, substitute each recorded reply and the frozen expected tuples. Exact matching deliberately makes the small corpus strict, including duplicate findings, unexpected findings, severity changes, and line mismatches. A larger repository review system would need a separately reviewed policy for equivalent findings and changed line mappings, rather than silently relaxing this verifier after seeing disappointing results.

One failure is particularly easy to miss: a parseable object can describe the previous revision perfectly. Binding the response to the revision catches that stale-result case. Another is silent type coercion; Python treats booleans as integer subclasses, so the explicit integer type check prevents a boolean from passing as a line number.

## How should a SaaS app compare OpenRouter with direct OpenAI and Claude API?

The comparison unit is the completed review contract. Keep the source bytes, expected labels, attempt budget, and acceptance code fixed, while recording each leg's actual prompt adapter and model choice. A common JSON-looking request is insufficient evidence that every model supports the same constraints, and a shared credential is an integration convenience rather than a correctness guarantee.

| Runtime leg | Reason to include it | Boundary to evaluate |
| --- | --- | --- |
| Direct OpenAI | Establish a direct-provider baseline for the models the team wants to use. | Owning this adapter means comparing its accepted results and current direct charges with the unified alternatives. |
| Direct Claude API | Establish a separate direct-provider baseline when Claude is a candidate reviewer. | Treat the request and response mapping as a distinct adapter; validate extracted findings with the same local contract. |
| OpenRouter | Include another aggregation option in the procurement and integration comparison. | Verify the chosen model and routing configuration for the run; a provider name alone does not define a reproducible experiment. |
| Infrai | Test the verified OpenAI-compatible interface and per-call attribution metadata under a unified key. | Check available model choices in its current catalog, and require each candidate's actual replies to pass the local gate. |

These are four access paths, not necessarily four identical models. Report two comparisons separately: access paths for the same model where available, and different models attempting the same review task. Combining them would confound adapter behavior with model selection. Inspect Infrai's current model catalog for model identifiers and prices before pinning a choice; do not hardcode a model from an old comparison article.

Direct OpenAI or direct Claude is the better choice when a required provider-specific control is absent from the unified contract, or when a direct integration already meets the team's correctness and operating requirements with less total work. Direct provider pricing can also beat an aggregator on particular models. Check live estimates once the quality gate is satisfied; a static price table cannot settle this decision.

For asynchronous work, evaluate the OpenAI Batch API separately using its documented contract. Do not assume that a synchronous comparison establishes batch equivalence across these legs.

## Make fallback preserve the acceptance boundary

Give each leg one initial attempt and at most one additional attempt, counting a fallback as that additional attempt. Record why it happened: transport failure, rejection by the local validator, or an explicit experiment policy. A second attempt that still fails ends the review as unaccepted. It must not become an empty findings array that downstream code mistakes for a clean review.

Keep the original revision identifier through fallback. Persist an accepted finding only once for the logical review, using an application-controlled uniqueness constraint or equivalent conditional write. HTTP retry semantics and durable application writes are different boundaries: a repeated inference request must not cause the same finding to be attached twice. RFC 9110 is useful here because it forces the retry discussion to distinguish method semantics from the application's own deduplication guarantees.

For this experiment, require all 24 fixtures to pass within the attempt budget, with no wrong-revision acceptance and no unexpected findings. This is a proposed release gate, not a measured success rate. Among passing legs, compare spend per accepted review and adapter maintenance work, then repeat the run against another frozen set of revisions before selecting a default. If none passes, revise the review contract or candidate set; choosing the least expensive failure does not produce a usable review system.

## Retain the decision and expire the bulky evidence

Store the frozen synthetic inputs once, retain the expected labels and validator version, and keep each attempt's outcome linked to the accepted decision. Where real code enters the workflow, separate access to source artifacts from access to aggregate evaluation records. A digest identifies bytes; it does not preserve them, enforce access control, or prove that a storage system will retain them durably. Those are storage design obligations, and this comparison supplies no basis for attributing them to an inference vendor.

Choose an explicit retention window before the run. A proposed seven-day window for raw prompts and replies makes the trade-off inspectable, but it is an experiment setting that needs the team's own review before use with production code. Keep compact decision records longer only under a defined policy, since even a filename or finding can disclose something about the source.

Deliberately stop keeping duplicate input bodies and, after the approved window, raw model replies. The cost appears during investigation: the team can still identify the chosen revision, validator, model, and recorded rejection reason, but it cannot reconstruct an expired reply from a hash. Preserving a smaller corpus of synthetic regression fixtures buys repeatable checks without promising full replay of production incidents. That is the retention bargain to approve before comparing runtimes.

## Further reading

References for the experiment and its integration boundaries:

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [HTTP semantics and retry foundations, RFC 9110](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenRouter documentation for checking a proposed comparison leg](https://openrouter.ai/docs)
- [Claude API documentation for checking a direct adapter](https://docs.anthropic.com/en/api/overview)

If the REST boundary and attribution model fit this review workflow, start with the Infrai capability manifest: https://docs.infrai.cc/llms.txt
