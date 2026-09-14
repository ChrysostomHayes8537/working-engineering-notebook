# Marketplace Outage Control: Estimate Expensive AI Step Cost Before Agent Admission

Decision: admit an expensive AI call only after atomically reserving a conservative estimate from the remaining budget; settle the reservation against actual usage afterward, and refuse, defer, or downgrade the event when the reservation would cross the spending ceiling. In a marketplace outage, that ordering matters more than forecast precision because concurrent workers can each see the same apparent balance.

The unit of control should be a reservation, not a pre-call dashboard check. Estimate the next step from known input, the declared maximum output, and any bounded tool charges, then reserve that amount in the same consistency boundary that stores the budget ledger. This can reject some work that would have finished cheaply. The alternative is worse for a hard ceiling: optimistic admission lets a burst of seller, order, and payout events oversubscribe money that no longer exists.

## Decision record: invariants and failure boundaries

The concrete workload is a marketplace ingesting platform events while a downstream dependency is unavailable. Events keep arriving. Some need an expensive model step to classify operational impact; others can wait or use a deterministic rules path. The business choice is uncomfortable and explicit: preserve the spend ceiling and refuse lower-priority traffic, or accept more traffic while allowing spend to become probabilistic.

Three invariants define the design. First, committed spend plus active reservations must never exceed the ceiling. Second, the event identifier must make admission idempotent, so a delivery retry cannot reserve twice. Third, every reservation must reach one of three terminal states: settled, released, or expired under a documented recovery policy. Those are ledger properties. A model client cannot enforce them by itself. Keep secrets outside event payloads and logs. The OWASP Secrets Management Cheat Sheet recommends centralizing secret handling, applying least privilege, rotating credentials, and auditing their use. A budget record is not a secret, but the credential used to call a model is; mixing it into retryable event data turns a cost-control mechanism into a credential-distribution mechanism. Failure boundaries deserve names because vague diagrams hide them. A duplicate-delivery failure repeats an event after admission. A split-brain admission failure lets two workers reserve the same remaining balance. An orphaned-reservation failure occurs after reserve but before settlement. A price-catalog drift failure estimates with terms different from those used for later accounting. A usage-evidence failure gets a successful result without trustworthy metering fields. Each one needs a state transition, not a hopeful retry.

Stop there for a moment.

A timeout is ambiguous: it does not prove that no remote work occurred. The admission record must therefore survive process loss, and reconciliation must use the provider request identifier and returned usage when those are available. If usage remains genuinely unknown, the conservative policy is to keep the reservation charged until an operator or a bounded reconciliation process resolves it. I'm not sure a single expiry duration is defensible across every model provider; retention, request-status APIs, and billing latency determine that value, so measure those before setting it.

## How should a Python agent loop compare estimated AI cost with remaining budget?

Compare integers in the smallest currency unit, under one atomic ledger operation. Floating-point arithmetic is a needless failure mode for money. Python's `decimal` module provides decimal arithmetic, but an integer micro-unit ledger makes the admission comparison and database constraint straightforward. The estimate itself can be represented as: estimated input units multiplied by the catalog's input rate, plus maximum output units multiplied by its output rate, plus bounded tool charges. Rates and units must carry an effective version; don't silently reinterpret old reservations after a catalog update.

Input length may be exact only if the model's tokenizer and message serialization are known. Otherwise it is an estimate, and your mileage may vary across models. Add a measured safety margin rather than claiming universal tokenizer equivalence. For output, reserve against the configured output cap, not the average response, because admission protects a ceiling. If refusal is costly, create an explicit lower-cost policy: smaller output cap, deterministic classification, or durable deferral. Do not quietly weaken the reservation formula.

| Admission policy | Ceiling behavior | Refused traffic | Operational catch | Appropriate use |
|---|---|---|---|---|
| Atomic worst-case reservation | Hard bound, subject to correct catalog and metering | Highest during bursts | Stranded reservations need reconciliation | Regulated or contractual spend ceiling |
| Atomic percentile reservation | Bounded only to the chosen risk policy | Lower | Tail outputs can overrun the reservation | A soft ceiling with an approved overage buffer |
| Check balance without reservation | No concurrency guarantee | Lowest initially | Parallel workers oversubscribe the same balance | Single-threaded experiments only |
| Queue before admission | Hard bound at execution time | Work is delayed rather than immediately refused | Queue age and storage become product constraints | Events whose value survives delay |

The table separates refused traffic from lost traffic. A refusal response can be retried or routed to durable storage; an acknowledged event that disappears cannot. During an outage, acknowledge only after the event is durably recorded, then perform budget admission asynchronously. HTTP defines `429 Too Many Requests` for rate limiting and permits a `Retry-After` header, but an internal budget refusal does not automatically mean that status is the right public contract. Use a domain result such as `deferred_budget` in the worker and map it at the API boundary according to whether the caller can safely retry.

## Critical path in Python

The example below keeps provider details behind protocols and makes the ledger's atomicity requirement visible. The storage implementation must enforce uniqueness on `(account_id, event_id)` and serialize updates to an account's available balance with a transaction, compare-and-swap, or another mechanism that provides the same invariant. A plain read followed by a write is not equivalent.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Quote:
    input_units: int
    max_output_units: int
    input_rate_micros: int
    output_rate_micros: int
    fixed_tool_micros: int = 0

    @property
    def reserve_micros(self) -> int:
        return (
            self.input_units * self.input_rate_micros
            + self.max_output_units * self.output_rate_micros
            + self.fixed_tool_micros
        )


@dataclass(frozen=True)
class Reservation:
    reservation_id: str
    reserved_micros: int
    already_settled: bool = False


@dataclass(frozen=True)
class ModelResult:
    value: str
    actual_cost_micros: int
    request_id: str


class BudgetLedger(Protocol):
    def reserve(
        self, account_id: str, event_id: str, amount_micros: int
    ) -> Reservation | None:
        ...

    def settle(
        self, reservation_id: str, actual_micros: int, evidence_id: str
    ) -> None:
        ...

    def release(self, reservation_id: str, reason: str) -> None:
        ...


class ModelClient(Protocol):
    def classify(
        self, payload: str, max_output_units: int, idempotency_key: str
    ) -> ModelResult:
        ...


def process_event(
    account_id: str,
    event_id: str,
    payload: str,
    quote: Quote,
    ledger: BudgetLedger,
    model: ModelClient,
) -> str:
    reservation = ledger.reserve(
        account_id=account_id,
        event_id=event_id,
        amount_micros=quote.reserve_micros,
    )
    if reservation is None:
        return "deferred_budget"
    if reservation.already_settled:
        return "duplicate_complete"

    try:
        result = model.classify(
            payload=payload,
            max_output_units=quote.max_output_units,
            idempotency_key=event_id,
        )
    except ValueError:
        ledger.release(reservation.reservation_id, reason="invalid_request")
        raise

    ledger.settle(
        reservation_id=reservation.reservation_id,
        actual_micros=result.actual_cost_micros,
        evidence_id=result.request_id,
    )
    return result.value
```

This intentionally does not release on a network timeout. Doing so would reopen budget before the system knows whether the model accepted the request. The worker should record an `outcome_unknown` state and reconcile it outside this function. By contrast, a locally validated invalid request can release immediately because no expensive call was admitted. The exact exception taxonomy belongs in the model adapter, where transport errors, local validation errors, and authoritative remote rejection can remain distinct.

There is another sharp edge: actual cost can exceed the reservation if the catalog, unit conversion, or provider behavior disagrees with the quote. Treat that as a policy violation and emit a high-cardinality-safe metric keyed by account class and catalog version, while retaining request-level identifiers in trace or audit storage rather than metric labels. The ledger still records actual evidence; hiding the difference makes the next estimate worse. Alert on both reservation overruns and growing orphan age.

Testing should attack interleavings, not just arithmetic. Run two workers against a balance that can fund exactly one reservation and prove only one reaches the model adapter. Replay the same event identifier 100 times and prove there is one reservation. Crash after reserve, after remote acceptance, and before settle; then verify reconciliation preserves the ceiling. Finally, test the boundary values: zero remaining budget, an estimate equal to the balance, a one-micro-unit excess, and a catalog version change between queued and admitted states.

## The rejected option, and when it is still valid

The rejected design is `if estimate <= remaining: call_model()`. It looks reasonable in a code review and can pass every single-worker test. Under concurrency, worker A and worker B can both read a remaining balance of 10 units, each approve a 7-unit call, and commit 14 units of spend. The bug is not arithmetic; it is the missing reservation between observation and action.

Still, the catch is that atomic worst-case reservation is not suitable when refusal itself is more damaging than a bounded overage, or when the accounting source reports usage too late to support a useful hard ceiling. In that case, use a documented soft limit with an overage reserve, percentile-based admission, and a circuit breaker. For offline marketplace enrichment whose value remains after the outage, stick with durable queuing and admit later. For safety-critical moderation that cannot wait, a deterministic fallback may be preferable to either refusal or unbounded model spend.

This design also costs engineering time. It needs a transactional ledger, idempotency records, reconciliation, a versioned rate catalog, and operators who can investigate unknown outcomes. A low-volume, single-process prototype with a disposable spend cap may reasonably choose a serialized in-memory counter instead. Just label that boundary honestly; don't present it as a distributed guarantee.

## Operating the decision during an outage

The useful dashboard is not merely total spend. Show available budget, active reservations, committed spend, refusal count by event priority, oldest orphan age, reconciliation backlog, and queue age. Those measurements expose the real trade: how much marketplace work is being deferred to keep the ceiling intact. A green spend graph beside a silently growing seller-event queue is not healthy.

Define degradation order before the incident. Payout-risk and account-takeover signals might retain the expensive path; catalog cleanup can queue; low-value enrichment can be refused. Those examples are policy categories, not universal priorities. Product, risk, and finance owners must set them because an infrastructure team cannot infer the damage of refusing each event class.

Keep the response boring.

Deploy the ledger change before enabling parallel admission, shadow-compute quotes without calling the model, and compare quoted amounts with recorded usage. Then enforce on a small account cohort while watching overrun and refusal metrics. The go/no-go question is concrete: does every admitted call have exactly one durable reservation, and can every nonterminal reservation be explained? If not, increasing worker count only magnifies uncertainty.

The final rule remains narrow: reserve pessimistically when the ceiling is hard, degrade explicitly when traffic value differs, and reconcile ambiguity instead of pretending a timeout erased spend. That is how an agent loop survives an outage without turning either money or marketplace events into an unbounded variable.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.python.org/3/library/decimal.html
- https://www.rfc-editor.org/rfc/rfc9110.html#name-429-too-many-requests
- https://opentelemetry.io/docs/specs/semconv/general/metrics/
