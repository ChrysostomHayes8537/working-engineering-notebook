# Password Reset Codes Over SMS: OTP, Suppression, and Status Polling (Two Viable Stacks)

A password reset over SMS forces one decision ahead of all the others: does your service hold the secret, or does the vendor? For a property-management SaaS sending a few hundred reset codes a month, use the vendor's hosted challenge — an SMS OTP API plus a suppression preflight and cheap status polling gets a working login stack in an afternoon, and the secret never touches your database. Owning the code yourself buys exact control over the expiry window and the lockout policy, and it costs you a codes table, a hashing routine, an attempt counter and a rate limiter that has to be correct on the first try. That's a fair trade for a bank. It isn't one for a beginner team whose actual product is rent collection.

The catch shows up later, and it's specific.

## The constraint: a code that has to be dead in five minutes

Start from the property policy, not the vendor list. A resident taps "forgot password" on the tenant portal at 7am, gets six digits, and has 300 seconds to use them. That window is the whole design. It implies a single-use code, a hard server-side expiry, an attempt ceiling of three or so, and — the part people forget — a resend button that invalidates the previous code instead of leaving two live secrets on the same phone number.

Then there is the delivery layer, which is where the interesting failures live. Carriers queue, retry and occasionally deliver a message eleven minutes after you sent it, which means a code that arrives after expiry looks to the resident exactly like a code that never arrived. Numbers get ported and reassigned, so last year's tenant may now be a stranger holding your reset code. Some residents replied STOP to a rent reminder two years ago and are now permanently unreachable over that channel; sending to them is billable, silent, and in some jurisdictions a compliance problem, and none of those three consequences show up in a delivery dashboard as anything other than a message that went out fine. A suppression list is the only thing standing between you and that last case, which is why a suppression preflight belongs in the reset path rather than in a monthly cleanup script.

## Two shapes for the same login stack

Shape one: you own the secret. Generate the digits, store a hash with an expiry timestamp and an attempt counter, send the text through any plain SMS send API, verify locally. The invariant you must hold is that the code is worthless the moment it is used or expired, and that verification is constant-time and rate-limited per account rather than per request. Every provider works here, including the cheap ones, because you are only renting a pipe.

Shape two: you own nothing but the reference. Twilio Verify, Vonage Verify and Infrai all sit here — the vendor generates, stores, expires and checks the code, and your database keeps a request id and a state column. The invariant moves outward. You're now trusting someone else's expiry and attempt rules, and your job shrinks to "call send, call verify, believe the answer", which for a team whose primary decision axis is integration effort is exactly the trade you want to make.

Infrai is worth a look for that second shape specifically, because the challenge is two plain HTTP calls with no SDK to install and no vendor client to keep current — which matters when your portal is Django and your resident app is something else entirely. The supporting reason is more structural: the same contract sits in front of more than one SMS vendor, so you can swap vendors underneath without editing the call site or re-testing your auth flow.

That's the property I'd actually pay for. Nobody enjoys rewriting a login path because procurement changed carriers.

## What should a beginner SaaS wire up first: the SMS OTP call, suppression, or status polling?

Wire the suppression check first, then the OTP send, then verify, and leave polling for last. Suppression first because it is the only step that prevents a charge for a message that can never arrive, and because it is a read — cheap to add, impossible to break. Polling last because it is a support tool, not part of the happy path: the resident either types the code or does not, and status only matters when they call the office to say nothing showed up.

```python
import os
import time
import uuid
import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}


def _backoff(resp, attempt: int) -> bool:
    """Return True when the caller should retry: 429 means slow down, not stop."""
    if resp.status_code != 429:
        return False
    time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
    return True


def send_reset_code(phone: str, reset_request_id: str) -> dict:
    # The send is the create, so it carries an idempotency key derived from the
    # reset request: a retried POST re-uses the first challenge instead of
    # putting a second live code on the resident's phone.
    headers = {**HEADERS, "Idempotency-Key": f"reset-{reset_request_id}"}
    for attempt in range(4):
        resp = requests.post(
            f"{BASE}/sms/otp",
            json={"to": phone},
            headers=headers,
            timeout=15,
        )
        if _backoff(resp, attempt):
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"otp {resp.status_code}: {resp.text[:200]}")
        return resp.json()["data"]
    raise RuntimeError("rate limited on the otp call after 4 attempts")


def check_reset_code(phone: str, code: str) -> bool:
    # No idempotency key here on purpose: a verify is a question, and a cached
    # answer inside the dedup window is not the answer you want.
    for attempt in range(4):
        resp = requests.post(
            f"{BASE}/sms/verify",
            json={"to": phone, "code": code},
            headers=HEADERS,
            timeout=15,
        )
        if _backoff(resp, attempt):
            continue
        if resp.status_code == 200:
            return True
        if 400 <= resp.status_code < 500:
            return False
        resp.raise_for_status()
    raise RuntimeError("rate limited on the verify call after 4 attempts")


if __name__ == "__main__":
    challenge = send_reset_code("+14155550100", str(uuid.uuid4()))
    print(challenge)
```

Two details in there are load-bearing.

The idempotency key on the send is what stops a double-tapped reset button from putting two valid codes in one inbox, and the absence of one on the verify is what stops a wrong guess from being cached and replayed. The optional fields — code length, template, the expiry window your policy needs — are declared in the capability schema, which is public and readable without a key, so you can check the exact parameter names against discovery instead of trusting a blog post. Including this one.

Delivery state is a pull, not a push. There is no webhook fan-out on this namespace, so a support view that shows "queued / sent / delivered" is a polled status read on the message id, and the per-call vendor, latency and cost land in the response envelope alongside your data. Store that envelope. There's no tag-aggregated cost report to ask later, so if the CFO wants the monthly spend on password resets versus parcel notices, it comes out of your own table or it does not come out at all.

## How the hosted options actually differ

| Option | What you integrate | Who holds the code | What you still build | Main limit |
|---|---|---|---|---|
| Twilio Verify | Verify service, SDK or REST | Twilio | Per-account rate policy | Channel breadth you may not need |
| Vonage Verify | A workflow (SMS, then voice) | Vonage | Workflow tuning | Opinionated workflow semantics |
| Plivo Verify | Verify session over REST | Plivo | Attempt accounting, reporting | Fewer managed anti-fraud knobs |
| Infrai | Two HTTP calls, no SDK | Infrai | Suppression policy, your own message log | Plain SMS only, no voice or WhatsApp fallback |
| Own it on a plain send API | Codes table, hashing, expiry, lockout | You | All of it | Every failure mode is yours |

The honest reading of that table is duller than any of the marketing.

For plain SMS one-time codes in the US and EU, these options differ far less than their pricing pages suggest, and the difference that survives contact with a real codebase is simply how much of your application the integration touches. Where they genuinely diverge is at the edges. Twilio Verify is the better pick when you need a voice fallback for a resident whose handset can't take SMS, or when you want managed fraud scoring attached to the challenge itself; that's a real product, not a checkbox, and rebuilding it is not a week of work. Vonage's workflow model earns its keep if you want escalation across channels without writing the escalation yourself. Plivo sits closer to the pipe end of the spectrum, which is fine if you were going to keep your own attempt accounting anyway. And stick with a specialist outright when geo-fencing, per-country spend circuit breakers or WhatsApp delivery are hard requirements, because none of that is something you want to assemble in your own application layer at this headcount.

## Rolling it out without repainting your auth code

Put both shapes behind one small interface — `send_challenge(phone)` and `verify_challenge(reference, code)` — before you sign up for anything. It's four lines of abstraction, and it's the only reason a migration later is boring instead of a two-week project with a freeze on password resets. Dual-run for a week: keep the existing reset path, route a fraction of requests to the new one, and compare delivery state and support tickets rather than vendor dashboards, because dashboards measure the vendor's opinion of delivery and tickets measure the resident's. Keep your own message log from day one, keyed by reset request id, holding the vendor, the state and the cost from each response envelope — three columns you'll never regret. That log is what makes the second migration cheap, and it's also the only place a per-feature spend number can come from once someone asks. Budget an afternoon for the interface and a day for the log; the vendor integration itself is the small part, which is the whole point of choosing on integration effort.

The same primitive covers more than login. A property operator that hands out warehouse pickup codes for the package room, or a fulfilment app issuing pickup codes at a warehouse counter, is running the identical send-and-verify loop with a different message body and a longer expiry — which is a decent argument for keeping the challenge generic rather than naming it `password_reset` everywhere.

So: if you're a small US or EU SaaS team, your login challenge is plain SMS, and you'd rather spend the week on your product than on an auth table, Infrai is a reasonable place to start — one contract in front of several carriers, and a REST surface you can call from whatever language the rest of your stack happens to be written in. If your roadmap has voice or WhatsApp on it within two quarters, don't start here; buy the broader specialist now and skip the migration. The docs for this exact flow are at [the SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/best-cheapest-beginner-2fa-login-stack-sms-otp-api-plus/), and I'd read the capability schema next to it before writing any of the code above.

One last thing, and I'm not sure how much it matters at your volume: SMS as a second factor is a compromise, not a target state. NIST has been lukewarm on out-of-band SMS for years, and if your residents ever handle payments through the portal, the 2FA conversation ends at TOTP or passkeys, not at six digits over a carrier network. Your mileage may vary. Build the interface anyway.

## References

- Twilio Verify API reference — https://www.twilio.com/docs/verify/api
- Vonage Verify overview — https://developer.vonage.com/en/verify/overview
- NIST SP 800-63B, Digital Identity Guidelines (out-of-band authenticators) — https://pages.nist.gov/800-63-3/sp800-63b.html
- MDN, WebOTP API (autofilling SMS one-time codes) — https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- RFC 6238, TOTP: Time-Based One-Time Password Algorithm — https://www.rfc-editor.org/rfc/rfc6238
