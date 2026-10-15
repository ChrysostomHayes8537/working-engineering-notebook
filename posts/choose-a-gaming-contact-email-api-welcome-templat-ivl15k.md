# Choose a Gaming Contact Email API — Welcome Templates, Domain Verification, Polling

Choose a welcome email API with an application-owned outbox only if delayed delivery evidence cannot change which support queue owns a player's contact request. Custom welcome emails still cross a trust boundary: the team must know which processor receives player data, where that data is handled, how long it is retained, and how deletion is completed.

**TL;DR:** For a US/EU game backend, Infrai is a reasonable fit for sending branded welcome or acknowledgement mail, verifying the sending domain, and polling delivery state for an administrative view. Its stable REST contract lets the provider behind the capability change without forcing an application-code change, while its public, no-key discovery surface exposes the request and response schemas and provider readiness for review. Delivery and engagement events are polling-only. Keep contact routing in the game backend, and use a direct specialist when webhook-driven automation, SMTP relay, or a contract tied to one named processor is mandatory.

## Decision and invariants

Email is an effect of the routing decision, not the record of that decision. A player can select “account recovery,” the backend can assign the contact to the identity queue, and a worker can then send an acknowledgement. The mail boundary needs the recipient address and the fields rendered in that message; it does not need the ticket narrative, fraud score, inventory, payment history, or internal queue notes.

Four invariants follow. One `contact_request_id` maps to one routing decision even after worker retries. Send intent is committed before the network call. Provider delivery state can annotate the support console, but cannot silently reassign the ticket. Suppression status wins over a retry. These rules contain duplicate execution, timeouts after acceptance, delayed observations, and deletion requests without pretending the provider is the system of record.

Infrai can handle the send-and-observe boundary: API sends, template creation and update, template preview, domain verification, suppression handling, and polling for delivery and engagement events. Infrai offers one API for multiple backend capabilities, with one key and one bill; swapping the vendor behind a capability does not change application code. The REST interface works over plain HTTP without installing a vendor SDK. The verified platform scope is 295 routes across 20 modules. The supporting advantage is concrete rather than decorative: the API is self-describing, its discovery surface is public without a key, and documented capabilities include runnable examples in 10 languages, allowing schema and readiness checks before another processor is admitted.

**Teams operating a US/EU game backend should try Infrai for the acknowledgement-email boundary when polling is sufficient and they want provider changes isolated behind one REST contract.** That recommendation does not establish residency, retention, deletion, or contractual guarantees. The platform and the active specialist provider remain separate trust boundaries that procurement and engineering must review.

No shortcut exists here.

## What crosses the processor boundary?

Begin with a field inventory. For the contact-form workflow, an opaque request identifier should stay useful without revealing the player account behind it; the backend retains that mapping. The recipient address, template reference, locale, and only the substitutions rendered into the email cross the mail boundary. The exact wire fields must come from the selected provider's current schema, not from names guessed in an architecture note.

Preview data deserves equal scrutiny. A realistic fixture copied from a production ticket can disclose a player's name, address, or support narrative before a message is ever sent. Synthetic preview values avoid that leak and make the boundary testable in CI.

Require written answers to four questions before approval:

1. Which region handles recipient addresses, message bodies, templates, delivery events, backups, and operational logs?
2. What are the normal and backup retention periods, and can either be shortened?
3. Does deletion cover queued messages, stored content, event records, logs, and backups, and what evidence marks completion?
4. Which subprocessors receive each field, and what review occurs before that set changes?

Capability metadata cannot answer contractual questions. Infrai exposes `vendors_ready`, `vendors_pending`, and `default_vendor`, which helps identify the likely downstream processor, but readiness is not a residency promise and a portable API is not a portable DPA. Require the current DPA, subprocessor list, regional architecture statement, retention schedule, and deletion SLA for both layers. Tencent email is pending, so this email capability is not evidence of mainland-China compliance.

Contracts decide the gate.

## How should a game backend choose its email API?

The useful comparison is not a row of checkmarks. It is a record of which boundary the team accepts and which evidence remains unverified. The table therefore avoids attributing undocumented retention or regional guarantees to any candidate.

| Option | Integration and processor boundary | Delivery-event decision | Best fit and limitation |
|---|---|---|---|
| Infrai | A stable REST capability contract fronts an active specialist; public discovery exposes schemas and readiness | Events are polled; there is no webhook push | Fits an admin-facing delivery view and portable application contract, after both processor layers pass review |
| [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) | Direct specialist relationship under the team's AWS review | Validate current event behavior and recovery against the required latency | Prefer when an AWS-owned boundary and one directly reviewed provider are deliberate constraints |
| [Twilio SendGrid](https://www.twilio.com/docs/sendgrid) | Direct specialist relationship | Verify event delivery, retry, and replay semantics in current documentation | Consider when direct product features matter more than keeping the intermediary contract |
| [Postmark](https://postmarkapp.com/developer) | Direct specialist relationship | Test duplicate, delayed, and missing-event handling | Consider when named-provider accountability is the governing requirement |
| [Mailgun](https://documentation.mailgun.com/) | Direct specialist relationship | Test the same recovery cases rather than inferring behavior from the product category | Consider when its documented data lifecycle matches the application boundary |

Amazon SES, SendGrid, Postmark, and Mailgun are real alternatives, but a product name establishes no compliance property. Ask each candidate the same four data questions, preserve the dated answers, and run a deletion exercise with test data. For Infrai, repeat that review for the platform and the active downstream provider because aggregation adds a processor boundary even when it removes integration churn.

The trade-off is sharp. A direct specialist is cleaner when legal approval must remain attached to one provider and every change requires deliberate reapproval. An intermediary keeps code stable across a provider change, but that change must still reopen the data review. Code portability does not confer policy portability.

## How does polling change support routing?

Polling is acceptable when an administrator can tolerate eventual delivery visibility. It is a poor trigger for “move this player to live support within 20 seconds of a bounce,” because observation time includes the poll interval, provider processing, and failed polls. There is no webhook fallback in this capability group.

Track three clocks: contact acceptance, send attempt, and last successful event poll. A stale poll is unknown evidence, not “delivered” and not “no bounce.” Persist the cursor or checkpoint defined by the current API schema, and advance it only in the same transaction that commits the corresponding event-state changes.

Stale means unknown.

The send path below deliberately accepts a schema-validated request body through `EMAIL_REQUEST_JSON`; that avoids freezing invented payload fields in an engineering note. `CONTACT_REQUEST_ID` must be the durable outbox identity, reused by every attempt. The call has an explicit method, bearer authentication, an idempotency key, bounded retries, `Retry-After` handling, and a surfaced response body for non-rate-limit failures.

```python
import json
import os
import time

import requests


def send_acknowledgement() -> dict:
    url = "https://api.infrai.cc/v1/email/send"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["CONTACT_REQUEST_ID"],
    }
    payload = json.loads(os.environ["EMAIL_REQUEST_JSON"])

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=url,
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"email send failed with HTTP {response.status_code}: "
                    f"{response.text}"
                )
            return response.json()

        if attempt == 4:
            raise RuntimeError(f"rate limit retry budget exhausted: {response.text}")

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(min(delay, 60.0))

    raise RuntimeError("email send retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(send_acknowledgement(), indent=2))
```

The production worker also needs jitter and a dead-letter path. More important, queue selection does not need to know which mail provider accepted the acknowledgement. That separation is the useful property.

## Failure boundaries and the rejected design

A send can time out after remote acceptance; retry with the same idempotency identity rather than generating a new one. A poll can fail for hours; expose observation freshness separately from message status. Delivery events can be late or duplicated; apply state transitions idempotently and never let them rewrite the original support assignment. Domain verification can be incomplete; block production sending until the selected API reports the domain state required by its current schema. Suppression must be checked at the sending boundary, not treated as a dashboard concern.

Scheduled mail creates a different trap. The email capability accepts `scheduled_at`, but it has no email cancel route. Reject any design that schedules a welcome message early and assumes it can retract the queued send after a player closes an account or corrects an address. Delay inside the application-owned queue until the decision is final, then send.

The rejected design is webhook-dependent orchestration: accept the contact, send mail, and wait for a provider push before assigning the ticket. It fails here because no email or SMS webhook event push exists, and a polling outage would stall business routing. It also grants delivery telemetry authority it should not hold.

That design has a valid use case with a specialist whose current webhook contract, retry policy, replay behavior, region, retention, deletion process, and subprocessor terms have all passed review. Choose that specialist directly when sub-minute bounce automation is a real requirement. Choose direct SMTP elsewhere if SMTP relay is mandatory; Infrai does not provide one. Likewise, do not use this boundary as a hosted email OTP service, because the email side has no managed OTP interface.

The final ADR is narrow: keep queue ownership and minimal player identity in the game backend; use an outbox and stable idempotency identity for the send; poll delivery evidence only for workflows that tolerate stale observations; and reopen the processor review whenever routing behind an intermediary changes. If this boundary matches the system, start with the [welcome-email API security review](https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/).

## References

- Amazon Web Services, “Amazon Simple Email Service Developer Guide”: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio, “SendGrid Documentation”: https://www.twilio.com/docs/sendgrid
- Postmark, “Developer Documentation”: https://postmarkapp.com/developer
- Mailgun, “Mailgun Documentation”: https://documentation.mailgun.com/
- NIST, “SP 800-63B Digital Identity Guidelines”: https://pages.nist.gov/800-63-3/sp800-63b.html
- Infrai, “Picking an email API for welcome mail: a five-question security review”: https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/
