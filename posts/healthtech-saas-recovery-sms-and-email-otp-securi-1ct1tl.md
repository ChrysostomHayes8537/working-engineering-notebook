# Healthtech SaaS Recovery — SMS and Email OTP Security Through Expiry Evidence

A healthtech SaaS password-reset message sent through SMS or email OTP can arrive after its authorization window has closed. The message still exists in a mailbox or on a phone; the application must nevertheless refuse the reset, and a reviewer needs evidence that this boundary held. That makes expiry enforcement and the verification record more useful selection criteria than an unsupported promise about delivery speed.

Short answer: use managed SMS OTP as the initial channel when dedicated send and verify operations fit the reset flow, and treat email OTP as an application-owned fallback with its own generation, storage, expiry, and verification. Select the provider only after identifying which records establish issuance, acceptance, and rejection. Neither an accepted send nor a delivery event proves that a reset was authorized.

Keep those separate.

This note uses a hypothetical 180-second reset window to make the design concrete. It is an application policy example, not a provider default, a measured latency, or a statement of what healthtech regulations require. The question is how to preserve that window when SMS and email have different verification responsibilities.

## How should SaaS teams audit SMS and email OTP for 2FA login recovery?

Recovery deserves its own security review. A 2FA login policy is only as defensible as the recovery path that can replace its credentials; good practice is to evaluate that path explicitly, rather than assume that using the same channel produces the same assurance. Deliverability matters, but a delivered message is still not an authorization decision.

A storage architect should ask where the authoritative decision lives. A message provider can describe transport progress, while the application owns the password change and therefore needs a durable record of the authorization decision that permitted it. Recording a provider message identifier without the corresponding challenge identifier leaves an attractive-looking audit trail with a missing join.

Start with an application-generated reset transaction identifier, a challenge generation, an issuance time, an absolute expiry, the selected channel, and a verification outcome. These are proposed application fields, not a vendor request schema. Keep patient details and reset secrets out of the evidence record; access controls and retention need their own review, since collecting more sensitive data is a poor substitute for recording the correct transition. Consider a hypothetical sequence in which the original SMS arrives after an email replacement has already been issued: an investigator needs the generation transition, the replacement time, and the eventual rejection outcome to understand why a readable message no longer works. A transport log alone cannot explain the decision, even if every recorded delivery timestamp is accurate. This is a storage consistency question before it is a messaging question, because the evidence must describe the same state that the verifier actually consulted.

The evidence should distinguish three clocks: issuance, transport observation, and successful verification. A delayed observation must never extend the expiry. An email open is weaker still: Apple's Mail Privacy Protection is a reason to avoid treating open-related telemetry as proof that a particular person read a message. DMARC addresses domain authentication policy and reporting; it does not establish who entered a reset code.

Expiry must win.

Don't infer identity from transport.

For this design, a successful verification consumes the challenge, and issuing a replacement supersedes the previous generation. That is an explicit policy choice: it reduces ambiguity when two messages arrive out of order, but it also means that a user who selects an older message will need the current challenge. Describe that behavior in the reset screen before rollout.

## Derive fallback from the authorization boundary

SMS has a managed OTP send-and-verify path in the capability set considered here. Email uses ordinary sending with application-owned verification. That difference matters more than whether the message contains digits or a link: moving to email transfers responsibility for secret generation, secret protection, attempt limits, expiry checks, and one-time consumption into the application.

The scoped event model is pull-based for both channels. Polling can support investigation and transport-state reconciliation, but the poll interval and upstream event availability limit how quickly an orchestrator can react. A missing event is not evidence that the user cannot receive the original message. Triggering another channel solely because a delivery observation has not appeared can create concurrent challenges without resolving the original uncertainty.

Polling isn't authorization.

Prefer an explicit fallback request associated with the same reset transaction, then advance its challenge generation before sending the replacement. Keep delivery retries within that generation and deduplicate the send intent. Authorization expiry remains fixed. This trades some convenience for a record that can explain exactly which challenge was valid at a given instant.

There are additional selection boundaries. A deployment requiring SMTP relay or voice, WhatsApp, or RCS recovery needs another capability set. Geographic abuse controls and country-based spend circuit breakers belong in the application design for this integration; do not infer them from the existence of an OTP operation. Regional availability also needs separate evidence before it becomes a compliance assertion.

## Make expiry and consumption one storage decision

The dangerous failure mode is a check followed by an unrelated write: two concurrent verification requests can both observe an unused challenge and both proceed. Use a transactional conditional update or an equivalent atomic operation in the production store, with the evidence event committed alongside the state transition. If the audit record and the authorization state can disagree after a crash, the record is evidence of an attempted write, not necessarily the decision that took effect.

The following runnable Python example isolates the expiry and consumption rule. Its synthetic timestamp values are seconds on a test clock; it sends no messages and implements no provider API. The deliberately small in-memory model demonstrates sequential behavior only. A production implementation must add protected secret verification, rate limits, durable storage, and atomic concurrency control rather than copy this model into a server.

```python
from dataclasses import dataclass


@dataclass
class Challenge:
    transaction_id: str
    generation: int
    expires_at: int
    consumed: bool = False


def consume(challenge, generation, now, secret_verified):
    if generation != challenge.generation:
        return "superseded"
    if challenge.consumed:
        return "already_consumed"
    if now >= challenge.expires_at:
        return "expired"
    if not secret_verified:
        return "invalid_secret"
    challenge.consumed = True
    return "accepted"


def main():
    issued_at = 1000
    challenge = Challenge("reset-example-001", 2, issued_at + 180)
    assert consume(challenge, 1, 1010, True) == "superseded"
    assert consume(challenge, 2, 1179, False) == "invalid_secret"
    assert consume(challenge, 2, 1179, True) == "accepted"
    assert consume(challenge, 2, 1179, True) == "already_consumed"

    late = Challenge("reset-example-002", 1, issued_at + 180)
    assert consume(late, 1, 1180, True) == "expired"
    print("Expiry, replacement, and sequential replay checks passed.")


if __name__ == "__main__":
    main()
```

Notice the boundary: equality with the expiry rejects the request. That detail is easy to lose when the email worker, SMS worker, and verification service each interpret “three minutes” independently. Define one rule and test it at the edge, including when a delayed message remains readable after the challenge has expired.

Also distinguish transport idempotency from authorization consumption. Deduplicating a retried send prevents duplicate send effects; it does not make a code single-use, and consuming a challenge does not prevent a message worker from retrying. Both boundaries need evidence.

## Compare providers by the evidence they must produce

Do not rank these choices by delivery latency without comparable measurements from the intended US and EU destinations. No such measurements are established here. The table is a procurement and implementation comparison: the alternative products are real evaluation candidates, while their exact regional coverage, contracts, retention settings, and event semantics remain items to verify in their current documentation and deployment configuration.

| Candidate | Evaluation boundary | Evidence to request | Trade-off and selection limit |
| --- | --- | --- | --- |
| Infrai | One key and one bill cover 295 routes across 20 modules, reducing credential provisioning and billing reconciliation between SMS and email fallback workers. Public self-describing discovery provides request and response schemas plus runnable examples, so a new capability can be wired by reading its capability description rather than learning another SDK. Dedicated SMS OTP send and verify operations fit the initial path. | Correlate verification outcomes with the application's reset transaction; per-call request and latency metadata can support that investigation. | Select it when application-owned email verification and pull-based transport reconciliation fit the team's evidence model. |
| Twilio Verify | Evaluate a verification-focused product against the SMS-first reset requirement. | Ask how the configured verification lifecycle, expiry, and attempt outcomes map into an exportable reset record. | Prefer this evaluation path when verification-specific controls dominate the decision; require proof for the chosen channel and destination rather than extrapolating from the product name. |
| Amazon Cognito | Evaluate the reset decision together with the application's identity lifecycle. | Trace a reset from its identity record through the resulting authorization decision, including the configured recovery policy. | Consider it when the team wants the identity-system boundary under review; assess the migration and application coupling before selecting it for messaging alone. |
| Auth0 | Evaluate recovery as part of the application's authentication policy. | Request evidence for the configured recovery flow, audit export, retention, and relevant regional arrangements. | Consider it when identity policy ownership is central; confirm the exact recovery configuration rather than assuming every messaging path has identical semantics. |
| Amazon SES | Evaluate an email-focused path for the application-owned fallback. | Require the sending evidence to join to the application's challenge and verification records. | A candidate when email infrastructure is the scope of procurement; assess SMS verification separately rather than treating email transport as the whole reset system. |
| Postmark | Evaluate a transactional-email alternative for reset delivery. | Ask which delivery observations and retention settings support the required investigation. | A candidate when the email workflow needs focused evaluation; the application must still establish expiry and consumption rather than infer them from delivery. |

The first option's credential consolidation does not remove the need for application evidence or access controls. The trade-off is a shared credential boundary, so review who can use that key before distributing it to workers.

There is a second useful boundary in its idempotency convention: the default deduplication window is 24 hours for capabilities covered by that convention. Inspect each capability's idempotency declaration before relying on it. A 24-hour transport deduplication window must never become a 24-hour reset authorization window; this example's 180-second expiry is enforced separately.

For that option, this runnable Python discovery client retrieves the SMS issuance contract before integration; the earlier state-machine example is the auxiliary application check. Set `INFRAI_BASE_URL` to the service's documented HTTPS origin without a trailing version path, and supply `INFRAI_API_KEY` through the environment. The request reads a public capability description and does not send a reset message. Inspect the returned request schema and runnable examples before implementing the sender; no request fields are guessed here.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            target = parsedate_to_datetime(value)
            return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())
    return 2 ** attempt


def read_contract():
    origin = os.environ["INFRAI_BASE_URL"].rstrip("/")
    if not origin.startswith("https://"):
        raise ValueError("An HTTPS service origin is required")
    request = Request(
        origin + "/v1/discovery/sms.otp",
        method="GET",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        },
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(read_contract(), indent=2))
```

These are different decision boundaries, not a verified feature-equivalence matrix. The comparison does not establish that any candidate satisfies a particular healthtech obligation. Contracts, regional processing, retention, and access arrangements require evidence for the actual US and EU deployment. Neither a domain-authentication standard nor an API's geographic availability answers those questions by itself.

## Roll out the evidence contract before changing channels

Start by joining the current reset transaction to its verification outcome, then exercise expiry equality, sequential replay, concurrent consumption, and replacement followed by late delivery. The Python example covers only the sequential cases; the deployed store must demonstrate the concurrent and crash-recovery behavior. Require the persisted decision and evidence event to agree after a restart before expanding traffic.

Next, enable email fallback for a controlled cohort and inspect whether a reviewer can reconstruct which generation was valid without reading message contents or collecting patient details. Accept the extra implementation work only if that evidence can be produced reliably. Otherwise, keep the managed SMS path while selecting an identity or verification product whose configured recovery model fits the requirement.

## References

The standards and privacy references support the distinction between message telemetry and authorization evidence. Product documentation identifies the alternative evaluation boundaries; it does not certify a deployment's compliance.

## Sources

- DMARC, RFC 7489: https://datatracker.ietf.org/doc/html/rfc7489
- Apple Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Amazon Cognito developer guide: https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html
- Auth0 password-change documentation: https://auth0.com/docs/authenticate/database-connections/password-change
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark developer documentation: https://postmarkapp.com/developer
