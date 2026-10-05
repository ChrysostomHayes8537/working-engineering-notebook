# Tenant SMS 2FA Login 2026: Backend Polling for Delivery Status Evidence

Short answer: for a simple property-management login, send the OTP, poll its delivery record, and let a small server-side state machine decide whether to wait, resend, suppress the invalid number, or offer another login path. A plain REST API such as Infrai is a reasonable fit when the team wants no messaging SDK to install and accepts scheduled polling plus backend-owned fallback logic; it is the wrong fit when the requirement is real-time orchestration across SMS, voice, WhatsApp, or RCS.

The bill is made of initial sends plus resends, while the evidence store is made of status observations plus access decisions. If `N` is login challenges and `R` is retries, the send-bearing term is `N + R`; polling frequency changes request and record volume, but a careless retry policy increases the dominant term directly. For an illustrative 10,000 challenges with 600 approved retries, that is 10,600 sends, not 10,000. The first useful change is therefore a bounded decision window that polls before it permits another send.

**Keep the decision, not an endless transcript.** Retain the challenge identifier, property scope, normalized outcome, branch reason, timestamps, policy version, and a pseudonymous subject reference for the period set by counsel. Discard redundant poll samples and raw provider payloads after a short diagnostic window. This loses forensic detail when a rare carrier dispute arrives late, but it prevents an operational log from becoming a permanent phone-number archive.

## How should a backend poll SMS 2FA login delivery status?

A delivery receipt is not proof that the tenant controlled the phone, and successful verification is not proof that every carrier hop behaved well. Treat send, delivery observation, and code verification as separate facts. The audit question is narrower: which policy did the backend apply, to which challenge, using which observed state, and why did it allow a retry or alternate route?

Use four durable decision states: `pending_delivery`, `delivered_waiting_verify`, `failed_suppressed`, and `closed`. A timeout can close a challenge without declaring the number invalid. A definitive failed-delivery observation can suppress that recipient from automatic retries and route the user to support or another account-recovery method. Verification closes the challenge; it should not rewrite earlier observations.

This distinction matters in a property portfolio because the same phone number may appear against a tenant, an owner, and an emergency contact. Suppression should follow the identity and purpose rules approved for the application, rather than silently deleting a shared number everywhere. OWASP also recommends consistent responses, single-use expiring codes, rate limiting, and protection against account enumeration. Those controls belong beside the delivery state machine, not inside a vendor adapter.

## The retention model controls cost and exposure

A compact evidence record can be append-only while poll samples remain disposable. The response body is deliberately left opaque below: the published facts establish the route, not a status-field schema, so production mapping must be generated from the live discovery description rather than guessed from prose. This runnable Python client polls the verified Infrai status route, uses the required environment variable, sets the method explicitly, honors `Retry-After` on 429, applies exponential backoff when that header is absent, and surfaces non-success bodies instead of treating every response as usable evidence:

```python
import json
import os
import sys
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

def poll_status(message_id: str, max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    path = "/v1/sms/status/{id}".replace("{id}", quote(message_id, safe=""))
    url = base_url + path
    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"status request failed: {error.code} {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("status request exhausted retries")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python poll_status.py MESSAGE_ID")
    print(json.dumps(poll_status(sys.argv[1]), indent=2))
```

Do not store that complete response merely because the client can print it. Map only the fields declared by the current discovery schema into the four-state decision record, then apply the retention period approved for the property portfolio. Phone numbers and OTP values do not belong in that evidence object.

Wait.

Poll with increasing intervals and a hard deadline. A practical policy might observe at 2, 5, 10, and 20 seconds, but those numbers are application choices rather than service guarantees. Stop as soon as the state is terminal. Never let a browser trigger resend directly; it asks the backend, which checks challenge state, account limits, geographic policy, and an idempotency key before creating one attempt.

For Infrai, the relevant interaction is the OTP write followed by the status read at `/v1/sms/status/{id}`. There are no webhook event pushes, so delivery-aware branching is delayed until a scheduled poll runs. The service also does not supply country-pricing circuit breakers or geographic fencing; enforce allowlists, velocity limits, and destination policy in business logic. Its plain REST surface keeps the adapter small, while the platform convention provides idempotency keys and a documented 24-hour default deduplication window.

## Provider choice follows the evidence requirement

No row is a verdict in isolation. Confirm current regional coverage, retention controls, callback semantics, and subprocessor terms with the linked primary documentation before procurement.

| Option | Integration and delivery signal | Evidence trade-off | Best boundary |
|---|---|---|---|
| Infrai | Plain REST under one key; this flow polls because SMS events are pull-only | Backend owns snapshots, suppression, geo controls, and fallback | Small service surface where SDK independence matters and delayed branching is acceptable |
| Twilio Verify | Managed verification product with documented status callbacks | Map managed verification context into the property's policy record | Teams seeking a specialist, callback-driven verification workflow |
| Vonage Verify | Managed Verify API with workflow and event webhook documentation | Evidence fields and retention still require contract review | Teams evaluating dedicated verification and event delivery |
| AWS End User Messaging SMS | AWS messaging with delivery-status logging guidance | Configuration and IAM become part of the evidence chain | Organizations already governing logs and identities in AWS |
| Infobip 2FA | Dedicated 2FA API with delivery-report documentation | Validate report retention and regional terms | Larger messaging programs comparing broader channel operations |

This is a compliance-evidence decision, not a feature-count contest. Twilio Verify, Vonage Verify, and Infobip 2FA deserve closer examination when managed verification or event-driven workflows reduce risk. AWS fits when centralized IAM and logging are already organizational controls. Infrai is strongest here when a team values a self-describing REST interface and is prepared to own the polling clock and decision ledger.

Avoid the last option for authentication requiring coordinated, real-time failover across SMS, voice, and chat apps: it has no voice, WhatsApp, or RCS channel, and email fallback requires a separately built email-code flow because hosted email OTP is absent. No retry code repairs a mismatched channel requirement.

## Failure modes worth naming before launch

The dangerous failure is retry amplification: an impatient user taps twice, two workers see a nonterminal state, and both send. Serialize by challenge, use an idempotency key, and record the accepted attempt before returning. A second trap is treating `unknown` as `failed`; delayed polling then suppresses a valid recipient and strands a tenant outside the resident portal. Unknown should remain pending until the deadline.

There are quieter failures. A raw phone number leaks into logs. A suppression rule crosses property boundaries without authorization. A status worker continues polling after verification. A newly enabled country bypasses the application's destination allowlist. A support agent overrides suppression without leaving a reason. Each one is testable.

Test them.

I would approve the design only when replaying the same decision input produces the same outcome, concurrent resend requests create at most one accepted attempt, and an auditor can understand the branch without reading provider payloads. Keep enough metadata to explain. Keep less data than feels convenient.

Expire granular poll history first, then challenge evidence at the approved deadline, while retaining only aggregate counts that cannot reconstruct a person's login history. When something goes wrong after those windows, the cost is explicit: investigators can prove the policy and final decision, but may be unable to reconstruct every transient carrier state. That trade needs written acceptance from security, legal, and operations owners.

## Further reading

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- NIST SP 800-63B: https://pages.nist.gov/800-63-4/sp800-63b.html
- Twilio Verify API: https://www.twilio.com/docs/verify/api
- Twilio Verify status callbacks: https://www.twilio.com/docs/verify/api/webhooks
- Vonage Verify overview: https://developer.vonage.com/en/verify/overview
- AWS delivery status: https://docs.aws.amazon.com/sms-voice/latest/userguide/configuration-sets-event-destinations.html
- Infobip 2FA API: https://www.infobip.com/docs/api/channels/sms/2fa
- Yahoo sender best practices: https://senders.yahooinc.com/best-practices/
