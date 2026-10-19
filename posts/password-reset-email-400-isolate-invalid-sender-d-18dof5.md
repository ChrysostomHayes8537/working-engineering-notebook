# Password Reset Email 400: Isolate Invalid Sender Domains from Suppression State

A password reset request that receives HTTP 400 with an invalid or unverified From domain has failed before delivery, so retrying the same payload cannot improve reliability. **Treat it as a control-plane configuration rejection, verify the exact sender domain and DKIM state, and keep it out of the recipient bounce pipeline.** For a B2B SaaS system, that separation matters: a malformed sender must not cause a valid user to be suppressed, while a genuine hard bounce should still prevent repeated mail to an invalid recipient.

TL;DR: capture the response and correlation ID, compare the envelope's From domain with the authenticated domain, verify public DNS from outside the deployment environment, and retry only after authentication is active. Do not mint another reset token on a blind retry. A 400 belongs to request/configuration handling; a later delivery event belongs to recipient-state handling.

## What should handle a password reset email 400 bad request?

The failures occur on opposite sides of message acceptance. An HTTP 400 says the submission request was rejected. No accepted message means there can be no trustworthy downstream delivery outcome for that attempt. A bounce is a delivery-status failure associated with a message that entered the mail path; SMTP enhanced status codes provide machine-readable classes and subjects for those reports.

That distinction should survive storage. If one `failed` flag covers API validation errors, DNS authentication problems, hard bounces, mailbox-full responses, and complaints, the system will eventually suppress the wrong people or retry requests that cannot succeed. A tenant configuration error can affect thousands of password reset attempts, yet none of those recipient addresses has become invalid.

| Observation | Scope | Durable effect | Retry condition |
|---|---|---|---|
| HTTP 400, invalid From domain | Tenant sender configuration | Block that sender | Configuration changes |
| HTTP 429 or transient server response | Submission attempt | Preserve attempt history | Backoff permits |
| SMTP 5.x.x recipient failure | Recipient and message | Suppress by policy | Evidence is reviewed |
| SMTP 4.x.x temporary failure | Message attempt | Record transient state | Queue policy permits |
| Complaint event | Recipient and channel | Suppress applicable mail | Policy-defined reversal |

This is a classification model, not a claim that every service exposes identical fields. Normalize only evidence actually received, retain the original payload for audit, and make event processing idempotent because webhook delivery can be repeated.

Three durable records are enough for this boundary: sender configuration, submission attempt, and recipient suppression.

## Trace sender identity before touching retries

Inspect the exact serialized request, with secrets and reset tokens redacted. Read the full From mailbox, not the display name. In `Support <reset@accounts.example.test>`, the domain portion supplies the visible author domain, while a separately configured return path can participate in SPF and bounce handling. DKIM signs selected headers and a body hash; its `d=` domain is the identity evaluated for DKIM alignment under DMARC. SPF and DKIM answer different questions, and an SPF pass does not prove that a sending service has authorized the From domain.

Then query authoritative public DNS from outside the application cluster. This avoids mistaking split-horizon DNS, a stale resolver, or an unpublished change for global readiness. Compare the selector and record value character for character with the required value. DNS publication is necessary, but the submission service may also maintain an explicit verification state; wait for that state to become active before replaying the rejected request.

Check organizational alignment too. DMARC passes when at least one supported authentication mechanism passes with an aligned authenticated domain; relaxed and strict alignment differ. A DKIM record can exist and validate cryptographically while the visible From identity still fails the intended policy.

Be precise. A `400` alone does not prove DKIM caused the rejection. The response body may instead identify invalid JSON, a missing field, or a malformed mailbox. Preserve the error code and message, map only known codes to `sender_not_verified`, and leave unknown validation errors unknown until inspected.

A narrow diagnostic can validate address construction without pretending to certify deliverability:

```python
from dataclasses import dataclass
from email.headerregistry import Address


@dataclass(frozen=True)
class SenderCheck:
    mailbox: str
    domain: str
    expected_dkim_name: str


def build_sender_check(local_part: str, domain: str, selector: str) -> SenderCheck:
    address = Address(username=local_part, domain=domain)
    clean_domain = domain.lower().rstrip(".")
    return SenderCheck(
        mailbox=str(address),
        domain=clean_domain,
        expected_dkim_name=f"{selector}._domainkey.{clean_domain}",
    )
```

This deliberately stops short of declaring a domain verified. A TXT lookup establishes what DNS publishes; only the submission service establishes whether its configuration accepts that identity. Conflating those checks produces brittle automation.

## Keep suppression state independent

A suppression record deserves storage semantics closer to an authorization decision than to a log line. Give it a stable recipient key, tenant scope, channel, reason, source event ID, observed time, and policy version. Store the message identifier when one exists. Do not key it only by a mutable reset token or request ID.

This separation has a cost. It introduces more states, migrations, and reconciliation work than a single delivery-status column, so it is not suitable for a prototype that deliberately accepts manual recovery and has no durable suppression requirement. Once multiple tenants, asynchronous events, or compliance-driven suppressions enter the design, the simpler schema's operational ambiguity becomes the larger risk; the trade-off then favors explicit records even though engineers must own their retention and transition rules.

**Sender failures change sender readiness; recipient evidence changes recipient eligibility.** Enforce that invariant in the write path. An API rejection without an accepted message ID cannot create a hard-bounce suppression.

Webhooks can arrive late or more than once, and two workers may process the same event concurrently. Put a uniqueness constraint on the event identity where available, record the raw event before deriving state, and use a monotonic policy for irreversible states such as a complaint. A later temporary event must not silently make an address eligible after a hard bounce.

Retention is separate. Raw payloads can contain addresses and diagnostic detail, so their access, encryption, and deletion rules should match privacy obligations. The normalized decision may need a different lifetime. Keeping both forever because storage is cheap is an unbounded policy commitment.

Password resets add another boundary. Token expiry and one-time-use semantics must not depend on callbacks, which can be delayed or absent. When replay is justified after fixing authentication, check whether the existing token remains valid. Generating several live tokens during retries complicates user experience and incident analysis.

## Failure handling that protects delivery

Build a small state machine instead of a catch-all retry loop. Known authentication rejections move the tenant sender to `blocked_config`. Transient submission failures enter a bounded schedule with jitter. Accepted messages wait for delivery events, and recipient suppressions are checked before any new submission.

Do not retry fast.

One retry queue cannot express every failure policy.

Count failures along dimensions that lead to action: tenant, sender domain, error class, and message purpose. Avoid addresses in metric labels because they create high cardinality and expose personal data. A rise in `sender_not_verified` for one tenant belongs to its configuration owner; a rise in hard bounces calls for recipient-quality review.

Logs should join a reset workflow ID to an attempt ID and, after acceptance, to a message ID. Redact tokens and authorization headers. The useful timeline is short: reset requested, eligibility checked, submission attempted, request rejected or accepted, event received, normalized state committed. Acceptance is not delivery.

Test negative paths deliberately. In a non-production tenant, confirm that an unauthorized identity is rejected without writing recipient suppression; duplicate a signed event and observe one transition; deliver events out of order; and verify that a suppressed address stops before queueing. Verify signatures against raw request bytes, because parsing and re-serializing a signed body can change those bytes.

## Roll out the boundary

First, add sender, attempt, and recipient-state fields while continuing to read the old status. Backfill only states supported by preserved evidence; mark ambiguous rows `unknown` rather than guessing.

Second, shadow the classifier and compare its decisions with the existing path. Investigate disagreements by error class. Do not write suppressions from shadow results.

Third, enforce sender readiness before queueing password reset mail, then enable idempotent event writes and suppression checks. Finally, remove generic retries after every failure class has an owner and bounded action. The desired result is plain: a bad domain blocks one tenant's sender configuration, a hard bounce affects the evidenced recipient, and neither corrupts the other state.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc3463
- https://www.rfc-editor.org/rfc/rfc5321
- https://resend.com/docs/introduction
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
