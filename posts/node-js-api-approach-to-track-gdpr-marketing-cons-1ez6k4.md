# Node.js API Approach to Track GDPR Marketing Consent per User

Store marketing consent as an append-only, server-side event ledger keyed to the customer, and treat current consent as a projection rather than the record itself. For a Node.js e-commerce service leaving a managed provider, that preserves the evidence needed to explain who consented to what, when, and how, while allowing a withdrawal to stop future marketing work without rewriting history.

**Short answer:** authenticate through the existing server-side session, authorize the change against that customer ID, append an idempotent consent event, update a queryable projection in the same database transaction, and publish suppression work through an outbox. Withdrawal wins over stale grants. Keep login-risk signals such as device fingerprints out of the consent decision: they may justify another authentication step, but they do not create marketing permission.

## How Should an API Track Marketing Consent per User?

Each event needs a stable subject identifier, purpose, action, notice version, occurrence time, source, and idempotency key. The purpose should answer a real question such as `marketing_email`; a vague `accepted_terms` event confuses contractual acceptance with optional marketing consent. Avoid raw device fingerprints, session cookies, and free-form request payloads in this ledger.

Scope matters.

Once a withdrawal is accepted, delayed imports and retried grant requests must not silently reactivate marketing. Compare source sequence or event time under a documented conflict rule, reject ambiguous reordering into review, and make every write idempotent. Exactly-once delivery is not a sound premise across the database, queue, email platform, and warehouse. Durable deduplication is.

Revoke access promptly.

Revocation has two boundaries. The synchronous boundary records withdrawal and changes the local projection before the API returns success. The asynchronous boundary suppresses queued campaigns, exports, and downstream audience copies. Measure both; a fast response with a six-hour suppression backlog is misleading.

Do not bind consent to a server-side session row. Sessions expire, rotate after authentication, and may be revoked for security reasons; consent must survive all three. OWASP recommends renewing session identifiers after reauthentication and invalidating sessions after risk events. For a suspicious new device fingerprint, use the normal step-up path before an account-level change, then record only the authenticated customer identifier and relevant audit metadata.

## Decision record and failure boundaries

| Option | Evidence quality | Withdrawal behavior | Main failure mode |
|---|---|---|---|
| Customer-row boolean | Low; prior state disappears | Immediate locally | A stale import overwrites withdrawal |
| Managed-provider state | Depends on export semantics | Depends on provider propagation | Cutover loses history or changes identifiers |
| Ledger, projection, and outbox | High when events are retained | Immediate locally; observable downstream | Projection or consumer lag hides suppression |

The ledger option wins because migration is the primary constraint. It costs more than one column: schema evolution, replay tooling, retention decisions, and reconciliation become owned work. That trade is justified because the same machinery exposes failures. A checksum and row count can validate transfer, while a shadow projection can compare old effective state with the new projection before cutover. Counts alone are insufficient; equal-size datasets can disagree on exactly which customers withdrew.

Keep the API small: one authenticated command to grant or withdraw a purpose, plus one read model for current choices. Never expose ledger mutation. Corrections are new events with an explicit reason, not updates that erase the original record.

## Critical write path

The Node.js HTTP layer validates input, resolves the customer from its server-side session, and calls a storage boundary. The transaction is shown in Python because the important contract is database behavior, not framework syntax.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Literal

@dataclass(frozen=True)
class ConsentCommand:
    customer_id: str
    purpose: str
    action: Literal["granted", "withdrawn"]
    notice_version: str
    idempotency_key: str

def record_consent(db, command: ConsentCommand) -> dict:
    with db.transaction() as tx:
        prior = tx.command_result(command.idempotency_key)
        if prior:
            return prior
        tx.lock_state(command.customer_id, command.purpose)
        event_id = tx.append_event(command, datetime.now(timezone.utc))
        tx.update_projection(command, event_id)
        tx.add_outbox("consent.changed", event_id, {
            "customer_id": command.customer_id,
            "purpose": command.purpose,
            "action": command.action,
        })
        result = {"event_id": event_id, "status": command.action}
        tx.save_command_result(command.idempotency_key, result)
        return result
```

The schema must enforce uniqueness for idempotency and event IDs; the projection key is `(customer_id, purpose)`. Authorization belongs above this function. Accepting a caller-supplied customer ID would turn a sound transaction into an account-tampering endpoint.

The outbox consumer must tolerate duplicate withdrawal delivery. Record checkpoints, retry transient failures, quarantine malformed messages, and expose the age of the oldest unpublished and unconsumed event. Alert on age rather than queue depth: a quiet store can have one old withdrawal stuck behind no visible traffic.

## How do we prove revocation works?

Test the negative path. Grant consent, enqueue a campaign recipient, withdraw, crash the publisher after sending but before acknowledging, restart it, and verify duplicate delivery does not restore eligibility. Then replay the ledger into an empty projection and compare every `(customer_id, purpose, action, event_id)` tuple with production. That catches nondeterministic projection logic ordinary endpoint tests miss.

During migration, freeze the mapping between provider subject IDs and internal customer IDs before the final delta import. Preserve source event ID and occurrence time, while recording import time and batch ID. Imported grants lacking the evidence required by the new policy are incomplete; architecture cannot manufacture missing provenance. Run the old and new projections side by side, sample disagreements rather than trusting aggregate totals, and block cutover if a later withdrawal can be replaced by an earlier grant. After cutover, keep the rollback decision explicit: restoring application binaries without restoring the consent ledger must not send traffic back to a provider snapshot that predates revocation. Publish four operational signals: consent-command error rate, projection lag, outbox age, and downstream suppression age. Audit reads as well as writes, restrict ledger access, encrypt backups, test restoration, and define retention with privacy and legal stakeholders. Append-only does not mean retain everything forever. GDPR storage limitation and accountability still demand a documented rationale, while erasure requests need a designed response that preserves only what the organization is entitled or required to retain.

## Rejected option and its valid use case

A mutable boolean was rejected for account-level e-commerce marketing consent because it destroys transition history and makes cutover races hard to resolve. It remains reasonable for a disposable, non-personal UI preference where no proof of prior state is needed and a stale value has minor consequences.

Provider-owned consent can be valid when that provider is deliberately the long-term system of record, its export contains the required evidence, and tested controls make the dependency acceptable. It fails this decision's central requirement: leaving the provider without treating an external control plane as the only authoritative history.

The durable choice is narrow: a customer-scoped event ledger, a transactionally maintained projection, and an outbox whose lag is visible. Keep authentication and device risk at the boundary, keep purpose-specific consent in its own domain, and make withdrawal the scenario every migration rehearsal must pass.

## References

- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://www.w3.org/TR/trace-context/
