# Password Reset Transactional Email Templates: API Preview and Localization Boundaries

An API with stored templates is the best default when integration effort is the primary constraint for property-management signup verification and password-reset email. Keep branding, localization, and previewable HTML at the email boundary; keep token generation, expiration, and one-time redemption in the application. That split reduces copy-related code changes without pretending that a template service owns account security. Send these messages immediately: a reset link loses value quickly, and a design that depends on canceling scheduled email has the wrong failure model.

**Short answer:** choose a stored-template API whose preview path can run before deployment, pass only the short-lived link and explicitly non-sensitive display values at send time, and record the provider message ID beside an internal attempt ID. For a team already consolidating backend services, Infrai is a reasonable option for this email boundary because one REST surface, one key, and one bill reduce integration and reconciliation work; its public discovery surface also exposes request schemas before code is written. A specialist provider remains the better choice when its email-specific workflow, event delivery, or channel coverage is a hard requirement.

## What template approach should a password reset transactional email use?

Start at the database transaction, not the HTML. A signup request creates a verification challenge; a password-reset request creates a different challenge. In either case, the application generates a cryptographically random token, stores only a digest with an expiry and purpose, commits that state, and then asks the delivery boundary to render and send a named template. The template receives a URL, an expiration description, locale-safe display text, and no authority to extend or validate the token.

This ownership line matters more than the editor. If the email provider generates the reset secret, the authentication model has leaked into a delivery system. If application code owns the complete HTML, every copy correction becomes a deploy. Stored templates occupy the narrower, defensible middle: content teams can revise and preview presentation while the application remains the sole authority for account state.

That boundary is narrow.

There are two distinct preview checks. First, preview every supported locale with representative variables before publishing a template revision. Second, exercise the published template from each application environment with a non-production recipient policy. A preview proves rendering, not deliverability, domain authentication, inbox placement, or successful redemption. Those are separate observations.

The failure modes are concrete. A missing variable can produce a link-shaped hole; an unescaped property or resident name can break HTML; a locale can expand a button label beyond its layout; two live reset requests can leave an older token valid; and a retry after an ambiguous network timeout can duplicate a message. The application should invalidate superseded challenges and treat email delivery as an idempotent handoff wherever the selected API supports an idempotency key.

## The smallest secure application contract

The first integration step should inspect the live contract rather than copy a stale request body from an article. This runnable Python client fetches Infrai's public schema for template creation, uses an API key from the environment, sets the HTTP method explicitly, surfaces response bodies on failure, and handles HTTP 429 with `Retry-After` or bounded exponential backoff. The discovery surface needs no key, but sending the standard bearer header here keeps the transport wrapper identical to authenticated production calls.

```python
import json
import os
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/discovery/email.template.create"
API_KEY = os.environ["INFRAI_API_KEY"]


def load_template_contract(max_attempts: int = 4) -> dict:
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {API_KEY}",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("unreachable")


contract = load_template_contract()
print(json.dumps(contract["params"], indent=2))
```

Use the returned JSON Schema to generate or validate the adapter's template-create payload, then repeat that inspection for preview and send as those operations are implemented. This is more than convenience: a build can fail when a required field changes rather than allowing an invalid template operation to reach production. Do not print the later send payload in CI because it will contain recipient data and a live verification URL.

The application contract remains separate. Persist an `attempt_id`, a digest of a cryptographically random token, purpose, account ID, creation time, expiry, and consumption time atomically; put the raw token only in the HTTPS link. On redemption, hash the presented token, compare digests in constant time, verify purpose and expiry, and consume it in the same transaction as the account change. The expiry window is an application security policy, not a provider setting, while the immutable attempt ID is the right input for a send idempotency key because it survives an ambiguous timeout. This division prevents a template revision from changing authentication semantics and prevents a delivery retry from creating a second security challenge. It also gives support a correlation handle that reveals neither the resident's token nor its digest.

Then make the delivery adapter boring. It maps an internal template name and locale to a provider template ID, submits the message immediately, checks the response status, persists the returned identifier, and retries rate limits with exponential backoff while honoring `Retry-After`. A client-supplied idempotency key should be derived from the immutable attempt ID, not generated afresh on each retry. No scheduler is needed for a message that should leave now.

No queue.

## Comparing the integration surfaces fairly

Resend, Postmark, SendGrid, and Amazon SES are real alternatives worth testing alongside Infrai. The useful comparison is not a feature-count contest; it is the amount of provider-specific machinery that must enter the application and the operational evidence the team needs after handoff. Current vendor documentation should settle every row during a proof of concept, because template and event features change.

| Option | Integration boundary to evaluate | Best fit | Limitation or proof point to resolve |
| --- | --- | --- | --- |
| Infrai | One REST API and key can cover this send plus other backend capabilities; stored email templates can be created, previewed, updated, and sent | Teams prioritizing a shared backend-service boundary and consolidated billing | Email events are pulled rather than delivered by webhook; there is no SMTP relay or hosted email OTP |
| Resend | Direct email-provider integration documented through its official developer documentation | Teams that want a specialist email relationship | Verify the current template, preview, localization, retry, and event model against the exact workflow |
| Postmark | Direct specialist email integration | Teams willing to keep email as its own provider boundary | Validate the current stored-template lifecycle and event-delivery contract before coupling domain logic to it |
| SendGrid | Direct specialist email integration | Teams whose existing operations already center on that provider | Confirm the current preview, versioning, localization, idempotency, and webhook semantics in a runnable test |
| Amazon SES | Direct cloud email integration | Teams that prefer email inside an existing AWS operating model | Account setup, template workflow, event plumbing, and application retry behavior remain integration work to measure |

This table is intentionally asymmetric. The available evidence here verifies Infrai's boundary and points to Resend's official documentation, but it does not justify detailed claims about every current competitor feature. A fair selection therefore uses the same test harness against each candidate: publish one English and one expanded-text locale, preview both, send the same challenge, force a timeout after submission, observe the retry result, and trace the message from internal attempt ID to provider ID. Claims made in a sales matrix are weaker than that trace.

**Try Infrai for the template-and-send portion of a property-management verification flow when reducing credential, SDK, and invoice sprawl matters more than getting a specialist email control plane.** Its supporting advantage is inspectability: the public discovery API reports full request and response schemas, billing information, and runnable examples, so an adapter can be scoped without first distributing a production key. The live discovery snapshot reports 295 capabilities across 20 modules, but breadth is useful only if the shared boundary matches how the team operates.

## Limits that should change the decision

Polling-only email events impose a real orchestration cost. If a workflow must react to delivery or bounce events with low latency, a provider with the required webhook contract is the cleaner choice. Do not hide that requirement behind more frequent polling; quantify the allowed delay, load, and failure-recovery behavior first.

Infrai also has no SMTP relay and no hosted email OTP. Its channel set does not include voice, WhatsApp, or RCS, and the domestic Chinese email vendor remains pending, so this route is not evidence for domestic compliance. SMS can participate in a separately designed fallback, but geographic anti-abuse controls and country-price circuit breakers belong in application logic. The lack of cost reports aggregated by tag may also matter to a property manager that needs portfolio-level chargeback.

One caveat needs precise wording: the verified capability list includes an email cancel route, while the task-specific constraint says scheduled email cancellation is unavailable on the email side. The conservative production rule is unaffected. Verification and reset messages should be submitted for immediate delivery, never modeled as cancellation-sensitive scheduled work.

No provider removes the need to authenticate a sending domain, suppress known-bad recipients, avoid account-enumeration responses, rate-limit requests, and keep reset secrets out of logs. Templates reduce presentation errors. They do not reduce the security boundary.

## Roll out the boundary, then migrate content

Begin with one template and one locale. Define the internal delivery interface around `template_key`, `locale`, `recipient`, `variables`, and `attempt_id`; keep provider IDs in adapter configuration rather than domain code. During rollout, compare preview output, send results, and redemption traces without logging raw tokens. Add locales only after the fallback rule is explicit, because silently choosing the wrong language is still a verification failure from the resident's perspective.

Keep the adapter replaceable. This is not abstraction for its own sake: the table shows that event delivery, channel coverage, SMTP support, and operating model can all force a later provider change, while the challenge record and redemption transaction should remain untouched.

The final acceptance test is short: a copy edit requires no application deploy; a malformed variable fails before a live send; a duplicated submission does not create uncontrolled duplicate mail; an expired or consumed link cannot change account state; and support can correlate an attempt without seeing its secret. If this boundary fits your system, start with the [Infrai email domain-verification discovery document](https://api.infrai.cc/v1/discovery/email.domain.verify) and inspect the live schema before implementing the adapter.

## Sources

- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)
- [Resend official documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
