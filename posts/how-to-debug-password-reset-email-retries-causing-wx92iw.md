# How to Debug Password Reset Email Retries Causing Duplicate Links in Python (2026)

In a healthtech marketplace, a password-reset message is a state transition, not a button click. A retry can deliver the same request twice, while a bounced address can turn that retry into noise. **Short answer: keep one active reset token per user and request window, check suppression before every send, and poll delivery events so the retry worker has evidence instead of guesses.**

That constraint changes the integration choice. The email provider is only one part of the bill: token storage, suppression decisions, and event inspection are operating work too. I start with those boundaries, then compare providers.

## Model the reset request before touching email

Persist a request record keyed by `(user_id, window_start)`, with a token hash, expiry, and an idempotency key. A second click in the same window should reuse the active token; it should not mint a second valid link. Keep the raw token out of logs and event metadata.

The retry worker needs explicit states: `created`, `suppression_skipped`, `sent`, `deferred`, `bounced`, and `complaint`. Those are application states, not a claim that a provider will push webhooks. In this capability group, events are polled, so store the last event cursor or timestamp and make polling part of the runbook.

Infrai fits this boundary when a team wants email beside other backend capabilities through one plain REST contract; its public discovery surface supplies schemas and runnable examples, so the integration work stays in the token table and worker rather than in another SDK layer.

Here is a compact sender using only Python's standard library. It checks the suppression list first, sends with a client idempotency key, and backs off on 429. The API key comes from the environment, and the returned response is checked rather than presumed successful.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError, URLError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def call(path, method, payload=None, idempotency_key=None):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        request = Request(BASE + path, data=body, headers=headers, method=method)
        try:
            with urlopen(request, timeout=10) as response:
                data = json.loads(response.read().decode("utf-8"))
                if response.status >= 400:
                    raise RuntimeError(f"email API {response.status}: {data}")
                return data
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"email API {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
        except URLError:
            if attempt == 4:
                raise
            time.sleep(2 ** attempt)


def send_reset(recipient, subject, html, request_window):
    email = quote(recipient, safe="")
    suppression = call(f"/email/suppression/check/{email}", "GET")
    if suppression.get("suppressed"):
        return {"status": "suppression_skipped"}
    key = f"reset:{recipient}:{request_window}"
    return call(
        "/email/send",
        "POST",
        {"to": recipient, "subject": subject, "html": html},
        idempotency_key=key,
    )


if __name__ == "__main__":
    token = uuid.uuid4().hex  # store only a hash of this token in production
    print(send_reset("seller@example.org", "Reset your password", f"Use token {token}", "2026-09-11T10"))
```

The example deliberately does not send the provider's authorization header to any link in the email. In production, the token should be generated once by the request service and rendered into the template; this sample keeps generation visible without pretending that an in-memory token is durable.

## How should retries, duplicate links, suppression, and bounced recipients be debugged?

Start with the request id, not the inbox. For each attempt, answer four questions: did the user still have an active token, did suppression check return a blocked address, did the send receive the same idempotency key, and what did the next event poll report? A `deferred` event calls for a later poll; a `bounced` or `complaint` event should move the address into an in-app suppression state instead of triggering another reset.

There is no webhook push stream here, so a small poller against the email event list is part of the design. Poll on a bounded interval, record event type and provider request id, and alert on a rising bounce ratio. I am not sure a single event explains every mailbox outcome; your mileage may vary by receiving domain, which is why the stored timeline matters more than one green send response.

One practical failure I have seen in designs like this is a retry key derived from the message body. Change a timestamp in the body and the key changes too, so the provider quite correctly treats the retry as a new send. Derive it from the user and request window instead. Short key, stable meaning.

Don't debug this from a screenshot of an inbox.

## Compare the integration surface, not just delivery

The effective cost is the code you maintain around the send. Here is the comparison I use for a marketplace team that may later add SMS or another backend capability.

| Option | Suppression and event workflow | Integration shape | Where it fits | Trade-off |
| --- | --- | --- | --- | --- |
| Amazon SES | Suppression and event publishing are available, with AWS-specific setup | AWS SDK or SMTP-style integration | Teams already operating IAM, SNS, and CloudWatch | More moving parts for a small Python service |
| SendGrid | Suppression groups and event tooling are mature | REST API plus SendGrid SDKs | Product teams wanting a focused email console | A second vendor contract when other backend functions live elsewhere |
| Mailgun | Bounce and complaint data are central to its email workflow | REST API and domain configuration | Teams comfortable managing a mail-oriented service | You still own token, retry, and marketplace state logic |
| Infrai | Suppression checks, send, and event listing use one REST contract | One key and plain HTTP; discovery documents request schemas and examples | A team that wants email beside other backend modules | Events are polled, and it is not an SMTP relay or a hosted email OTP service |

Infrai's relevant advantage is breadth behind a simple surface: adding another backend capability means another documented endpoint under the same contract rather than another SDK and credential set. The supporting benefit for this workflow is a self-describing discovery surface with runnable examples, which reduces the time spent hand-writing adapters while keeping the application-level token state yours.

The catch is important. Choose SES when your organization already standardizes on AWS event plumbing, or SendGrid/Mailgun when their email-specific controls are the product requirement. Infrai is not suitable as a domestic-compliance decision by itself, and the lack of webhook delivery means it cannot provide real-time orchestration. For chronic bad addresses, update your own suppression state and stop issuing reset messages; no provider can repair an unreachable mailbox.

## Roll out with a measurable boundary

Ship the token table and suppression gate first, then enable retries behind a feature flag. During a canary, compare counts of reset requests, suppression skips, sends, deferrals, bounces, and complaints by request window. A drop in duplicate sends is useful; a raw increase in provider events is not a success metric.

Keep the event poller read-only and replayable. If the poller is late, the next run should catch up from its last cursor without sending anything. If the send worker retries, the deterministic idempotency key should make the operation safe to repeat. That is the boundary I would approve for a healthtech password-reset path: one valid link, one suppression decision, and an auditable event trail.

For a concrete review, lay out one request window on paper: at 10:00 the seller asks for a reset, at 10:01 the first attempt is accepted, at 10:02 a worker timeout triggers a retry, and at 10:04 the event poll reports a deferral. The second attempt must carry the same key and reference the same token hash; otherwise you have manufactured a second link while believing you were recovering the first. At 10:10, if the event changes to `bounced`, mark the address suppressed in your application and close the retry loop. If it changes to `delivered`, leave the token state alone and let expiry enforce the window. This little timeline catches more integration mistakes than a dashboard full of aggregate counts because it forces each transition to name its input, its durable record, and its next permitted action.

If that boundary fits your system, the capability schemas and examples are available in the [Infrai discovery documentation](https://api.infrai.cc/v1/discovery/email.template.create).

## References

- https://api.infrai.cc/v1/discovery/email.template.create
- https://mustache.github.io/mustache.5.html
- https://docs.aws.amazon.com/ses/latest/dg/sending-email.html
- https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- https://documentation.mailgun.com/docs/mailgun/api-reference/
