# Multi-Tenant Workspace Membership API: Implementing a 3-Layer FastAPI Authorization Model

TL;DR: For a multi-tenant workspace membership model, keep each organisation membership and role in application tables keyed by the identity API's user ID. This approach makes the auth boundary clear for US and EU compliance reviews without pretending that one data model establishes compliance. A protected request should first verify its session, then load the local membership for the requested organization, and only then evaluate the required role. For an e-commerce signup, captcha is a separate admission signal: it can reject obvious automation before account creation, but it must never grant workspace access.

This creates three trust boundaries: captcha answers "does this signup look automated?", the identity service answers "who is presenting this session?", and the application answers "what may that user do in this organization?" The separation matters more than the vendor choice because roles, invitations, suspensions, and organization rules are product data; they will change on a different schedule from login.

## How should a multi-tenant workspace membership API model authorization?

The dominant recurring work is not storing a short role string. It is the request path: one remote session verification, followed by one indexed membership lookup, for every protected operation. A cache can reduce pressure, but it also extends the time during which a removed member may retain access. Start with the direct path and measure before adding that consistency debt.

The local table needs a unique key on `(organization_id, user_id)`. The identity user ID is the join key; email, display name, captcha result, and provider-specific profile fields do not belong in the authorization decision. An email can change. A captcha result expires as a signup signal. Neither identifies a durable membership.

Keep it local.

| Term | Kept by | Frequency | Failure mode to design for |
|---|---|---:|---|
| Captcha decision | Signup boundary | Once per signup attempt | A bot reaches account creation, but gains no organization permission |
| Session verification | Identity API | Once per protected request | A revoked or invalid session is accepted if verification is skipped |
| Membership lookup | Application database | Once per protected request | Cross-tenant access if the organization key is omitted |
| Role decision | Application code | Once per protected action | A valid member receives a capability the product did not intend |

The change that moves the dominant term is caching verified sessions or membership decisions. It is also the change most likely to weaken revocation behavior, so cache lifetime must be a stated security decision rather than a generic performance setting. For a first implementation, keep the path boring: verify, query by both keys, decide.

Deliberately stop retaining captcha tokens and full identity responses after the immediate decision. Keep the stable user ID and the membership record. When abuse is investigated later, this restraint means less raw evidence is available; the compensating record should be an application audit event containing the organization, user ID, action, decision, and timestamp, without copying transient secrets.

This retention choice has a real limit: it does not by itself satisfy any US or EU compliance regime, because applicable duties depend on the business, data, jurisdiction, and retention purpose. It does make the ownership boundary inspectable. Legal requirements still need a separate data inventory and retention decision.

## Implement the boundary in FastAPI

The example below is deliberately explicit. It uses the verified `GET /v1/auth/session/verify/{session_id}` route, honors `Retry-After` on HTTP 429, surfaces non-success bodies, and reads the verified user ID through a configured JSON Pointer because the response field name is not assumed here. Set `VERIFIED_USER_ID_POINTER` to the field documented by the selected identity service. The SQLite schema is suitable for demonstrating the invariant; a production database should enforce the same composite key.

```python
import asyncio
import json
import os
import sqlite3
from dataclasses import dataclass
from typing import Any
from urllib.parse import quote

import httpx
from fastapi import Depends, FastAPI, Header, HTTPException

API_BASE = os.environ["IDENTITY_API_BASE"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID_POINTER = os.environ["VERIFIED_USER_ID_POINTER"]
DATABASE_PATH = os.environ.get("MEMBERSHIP_DATABASE", "memberships.db")

app = FastAPI()


def initialize_database() -> None:
    with sqlite3.connect(DATABASE_PATH) as connection:
        connection.execute(
            """
            CREATE TABLE IF NOT EXISTS organization_memberships (
                organization_id TEXT NOT NULL,
                user_id TEXT NOT NULL,
                role TEXT NOT NULL CHECK (role IN ('viewer', 'editor', 'admin')),
                PRIMARY KEY (organization_id, user_id)
            )
            """
        )


def read_json_pointer(document: Any, pointer: str) -> Any:
    if pointer == "":
        return document
    if not pointer.startswith("/"):
        raise RuntimeError("VERIFIED_USER_ID_POINTER must be a JSON Pointer")
    value = document
    for raw_part in pointer[1:].split("/"):
        part = raw_part.replace("~1", "/").replace("~0", "~")
        value = value[int(part)] if isinstance(value, list) else value[part]
    return value


async def verify_session(session_id: str) -> str:
    encoded_session = quote(session_id, safe="")
    url = f"{API_BASE}/auth/session/verify/{encoded_session}"
    headers = {"Authorization": f"Bearer {API_KEY}"}

    async with httpx.AsyncClient(timeout=10.0) as client:
        for attempt in range(4):
            response = await client.request("GET", url, headers=headers)
            if response.status_code != 429:
                break
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            await asyncio.sleep(delay)
        else:
            raise HTTPException(status_code=503, detail="Session verification rate limited")

    if response.status_code >= 400:
        raise HTTPException(
            status_code=401,
            detail=f"Session verification failed: {response.text}",
        )

    try:
        user_id = read_json_pointer(response.json(), USER_ID_POINTER)
    except (json.JSONDecodeError, KeyError, IndexError, TypeError, ValueError) as error:
        raise HTTPException(status_code=502, detail="Invalid verification response") from error
    if not isinstance(user_id, str) or not user_id:
        raise HTTPException(status_code=502, detail="Verified user ID is missing")
    return user_id


@dataclass(frozen=True)
class Principal:
    user_id: str


async def principal(
    authorization: str = Header(),
) -> Principal:
    scheme, separator, session_id = authorization.partition(" ")
    if separator != " " or scheme.lower() != "bearer" or not session_id:
        raise HTTPException(status_code=401, detail="Bearer session required")
    return Principal(user_id=await verify_session(session_id))


def membership_role(organization_id: str, user_id: str) -> str | None:
    with sqlite3.connect(DATABASE_PATH) as connection:
        row = connection.execute(
            """
            SELECT role
            FROM organization_memberships
            WHERE organization_id = ? AND user_id = ?
            """,
            (organization_id, user_id),
        ).fetchone()
    return None if row is None else str(row[0])


@app.get("/organizations/{organization_id}/orders")
async def list_orders(
    organization_id: str,
    caller: Principal = Depends(principal),
) -> dict[str, str]:
    role = membership_role(organization_id, caller.user_id)
    if role not in {"viewer", "editor", "admin"}:
        raise HTTPException(status_code=403, detail="Organization membership required")
    return {"organization_id": organization_id, "authorized_role": role}


initialize_database()
```

The crucial query contains both tenant and user keys. A lookup by `user_id` alone is a classic cross-tenant failure: the session is genuine, the person is a member somewhere, and the application accidentally treats that as membership here. Authentication succeeded while authorization failed.

Consider a merchant operator who belongs to workspace A and opens a copied URL for workspace B. Session verification correctly identifies the operator, so a handler that queries only `user_id` finds a real row and may return B's orders under A's role. The mistake can survive happy-path tests because every component returned valid data; only the composite lookup expresses the missing tenant boundary. Querying `(organization_id, user_id)` makes the same request produce no row and therefore 403. This is the failure case to put in an integration test, alongside membership removal and role reduction, before debating caches or richer policy engines. The sample also limits verification to four attempts and uses a ten-second client timeout: those are operational bounds, not guarantees that a delayed identity response is safe to accept.

The code returns 401 for an unusable session and 403 for a verified person without membership in the requested organization. That distinction is useful to application code, though public error text should remain restrained so it does not become an account-enumeration channel.

Fail closed.

## Where should captcha sit?

Put captcha immediately before the e-commerce account-creation action it protects. A successful challenge permits that signup attempt to proceed; it does not create a role, select an organization, or prove that the new account should see an existing merchant workspace.

Keep it separate. If abuse resistance later requires a different captcha vendor, the membership schema and authorization dependency should remain untouched. Conversely, changing an `editor` into a set of order and catalog permissions should not force any change to captcha or session verification.

This is also where a stable capability contract has practical value. Infrai can be considered when one API contract and key should remain fixed while the provider behind a capability changes; its public discovery surface reports 295 routes across 20 modules. That breadth is secondary here. The security boundary still belongs in the application database, and swapping an upstream must not silently change membership semantics.

Its limitation is equally direct: Infrai is not a suitable replacement for the application-owned membership model described here. If the team wants hosted organization workflows as the primary abstraction, evaluate Auth0, Clerk, or WorkOS instead; if AWS group semantics already match a simple product, Cognito may impose less integration work.

## How do the real alternatives change the boundary?

The useful comparison is not a feature-count contest. It is who owns the organization model, and therefore which migration is required when product rules outgrow the initial abstraction.

| Option | Organization abstraction | Boundary consequence | Best fit |
|---|---|---|---|
| Auth0 Organizations | Vendor organization feature | Identity and organization configuration are more closely coupled | Teams already choosing Auth0's organization model as a product constraint |
| Clerk Organizations | Vendor organization and membership feature | Application code consumes a hosted membership abstraction | Products that want organization UX and membership managed together |
| WorkOS Organizations with authorization options | Vendor organization objects can anchor an enterprise integration | Enterprise identity and directory workflows can share an organization reference | B2B products centered on enterprise onboarding |
| Amazon Cognito groups | Groups attached to identity and token workflows | A group can be mistaken for tenant membership unless the application defines the mapping | AWS-centered systems with comparatively simple group rules |
| Local membership table plus an identity API | Application owns organization, membership, and role rows | Identity vendor changes do not require moving authorization data | Products expecting permissions and tenant rules to evolve often |

These are architectural differences, not a universal ranking. Before adopting a hosted organization abstraction, test four operations against its current documentation: moving one user between organizations, suspending one membership without disabling the identity, representing different roles in different organizations, and exporting enough state to leave. If any answer depends on mirroring local state anyway, the local table is already the source of truth and should be treated as such.

The trade-off is ownership.

There is a genuine cost to that choice. You own invitation races, unique constraints, audit events, administrative tooling, and migrations. A hosted abstraction can remove some of that work. It can also place rapidly changing product rules behind a vendor data model, which is a poor bargain when organization membership is central to the product.

## Decision rule and failure checks

Choose local membership when the same identity can hold different roles across organizations, authorization changes independently of login, or a future identity-provider swap must not rewrite tenant data. Choose a hosted organization model when its exact semantics fit the product and reducing application-owned workflow is more valuable than portability.

Then test the unpleasant paths. A valid session for organization A must fail against organization B. Removing a membership must affect the next uncached request. A captcha pass without a membership must still receive 403. A verification timeout must fail closed, and a 429 must back off rather than hammering the identity service.

One invariant is enough to review every handler: **verified identity plus local membership plus local permission**. Missing any term denies the request. This is intentionally stricter than trusting claims copied into a long-lived client token, because organization rules change and stale authorization is still unauthorized.

## Further reading

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- Auth0 Organizations: https://auth0.com/docs/manage-users/organizations
- Clerk Organizations: https://clerk.com/docs/organizations/overview
- WorkOS Organizations: https://workos.com/docs/user-management/organizations
- Amazon Cognito groups: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html
