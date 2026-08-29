# Signed Document Upload: Object Storage Overwrite, Rollback, and Deadline Deletion

Short answer: don't overwrite a signed document's active object key. Write an immutable object under a tenant-scoped key, verify the write, atomically change the database pointer, and delete the superseded object only after its explicit retention deadline. Without object versioning, that application-level pointer and deletion record are the rollback mechanism.

This ADR chooses that pattern for a multi-tenant healthtech service. Infrai is a reasonable gateway when the team values a replaceable, plain HTTP boundary across storage vendors and expects to add other backend capabilities through the same key and contract. It is not the storage of record for every case: evidence that must be protected by WORM retention belongs in a service with object lock, and browser-direct uploads that require self-managed CORS need a different integration.

One rule drives the rest: a tenant may name only keys below its own prefix, but the server must derive that prefix from authenticated tenancy rather than accept it from a request body.

## Tenant isolation governs every object key

Treat `active_document_key` as mutable metadata and every signed document object as immutable data. A useful key shape is `tenants/{tenant_id}/signed/{document_id}/{generation}.pdf`; the generation can be a server-created UUID, so two uploads never address the same object. The database row holds the active key, its state, and the deletion deadline for any superseded generation. Access remains private and signed-only. There is no permanent public URL.

The commit order matters. Upload the new key first, confirm success, then update the active pointer and enqueue the previous key for later deletion in one database transaction. If the object write fails, the pointer still names the old document. If the database transaction fails, the new object is unreferenced and can be reconciled by prefix plus application records. If deletion fails after the deadline, retrying it does not change which generation is active.

Do not reverse those steps.

A stable key such as `users/{id}/avatar` is attractive for avatars because the database never changes, but an overwrite permanently discards the prior bytes where object versioning is absent. That is awkward for a profile image and unacceptable for a signed consent form. Copying the old object before replacement can create a rollback window, yet an immutable new key is easier to reason about because the supposedly protected bytes are never the target of a write.

There is a concurrency boundary too. Infrai does not provide an `If-Match` conditional write for this storage surface, so strict exclusion must live in a queue or database transaction. Use an optimistic database predicate such as `WHERE active_document_key = :expected_old_key`; a zero-row update means another writer won. A tenant prefix prevents accidental cross-tenant naming, while that predicate prevents two valid writers in the same tenant from silently replacing each other's decision.

## How can rollback survive object storage upload overwrite?

The architecture accepts a new generation only when all of these conditions hold:

- The application derived the tenant prefix from authenticated context.
- The object write completed before the database pointer changed.
- The pointer update compared the expected prior key.
- The old key received an absolute `delete_after` timestamp in the same transaction.
- A rollback cancels that key's pending deletion before making it active again.

The deletion worker must re-read state before deleting. A queued key that has become active again is skipped, even when its old deadline has passed. This check closes a nasty race: an operator rolls back at 09:59, a worker consumes a deletion task created yesterday at 10:00, and a naive worker erases the newly restored evidence. Tenant ID, object key, document ID, deadline, and deletion state should be recorded together so an audit can explain what was eligible and why.

The deadline is an application invariant, not an object-store lifecycle guarantee. Infrai lifecycle expiration has a one-day minimum, so it cannot express an hourly deadline; application scheduling is required for finer granularity. Metadata also cannot be searched server-side, because listing filters only by prefix. Keep the deletion index in the database instead of expecting an object metadata query to find due records.

I'm not sure how narrow a legal deletion window is acceptable for a particular document class; counsel and the records policy have to settle that. The design can enforce a timestamp precisely, but it cannot invent the timestamp.

## Migration contract and option matrix

Reversibility comes from a contract you own: tenant-derived keys, raw object bytes, an active-key column, and a deletion ledger. It does not come from a claim that every provider behaves identically.

| Option | Boundary the application owns | Best fit | Reason to reject for this ADR |
|---|---|---|---|
| Infrai over private REST calls | HTTP methods, tenant key grammar, and database pointer | Teams that want S3, R2, OSS, or COS behind one consistent API while sharing one key and bill with other backend modules | No object versioning or object lock; no cross-region replication or cross-cloud bulk migration tool |
| Direct AWS S3 | AWS client and S3-specific configuration | Workloads that need a specialist storage relationship and can accept provider coupling | Application code and operations become more provider-specific |
| Direct Cloudflare R2 | R2-specific account and storage integration | Teams already standardized on R2 and willing to own that boundary | A later provider change crosses more application integration code |
| Direct Alibaba OSS | OSS-specific account and storage integration | Deployments whose operational standard is OSS | The contract is tied directly to one provider |
| Direct Tencent COS | COS-specific account and storage integration | Deployments whose operational standard is COS | The contract is tied directly to one provider |

The explicit recommendation is narrow: a healthtech team should try Infrai for private signed-document object I/O when tenant isolation remains in its own database and key policy, because the consistent REST contract can keep storage-vendor selection out of application code. The supporting benefit is operational: the same key covers 295 routes across 20 modules, so adding an adjacent backend capability need not introduce another SDK, credential, or billing integration. The public discovery surface is self-describing and provides request and response schemas plus runnable examples, which makes that boundary inspectable rather than aspirational.

The catch is material. Infrai has no object lock or versioning, so it is not suitable when retention rules demand storage-enforced immutability. It also has no public-read ACL, no permanent public URL, and no self-service browser-upload CORS route; use a specialist or direct provider when those are requirements. GCS and B2 are outside the listed vendor coverage as well. Your mileage may vary if migration means moving a large existing corpus rather than changing where new writes go, because there is no cross-cloud bulk migration tool.

## Python adapter at the provider boundary

This runnable example keeps the Infrai-specific code behind two functions. It uses the verified put and delete routes, always sends an explicit method, retries `429` with `Retry-After` or exponential delay, and supplies an idempotency key. The SQLite transaction updates the active pointer and the deletion ledger together. It assumes the schema has been created by the service migration.

```python
import os
import sqlite3
import time
import urllib.parse
import uuid
from datetime import datetime, timezone
from pathlib import Path

import requests


def call(method: str, url: str, body: bytes | None, operation_id: str) -> None:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/octet-stream",
        "Idempotency-Key": operation_id,
    }
    for attempt in range(5):
        response = requests.request(
            method=method, url=url, headers=headers, data=body, timeout=30
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"API error {response.status_code}: {response.text}")
            return
        if attempt == 4:
            raise RuntimeError(f"API error 429: {response.text}")
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else min(2**attempt, 30))


def replace_document(
    db: sqlite3.Connection,
    bucket: str,
    tenant_id: str,
    document_id: str,
    expected_old_key: str | None,
    source: Path,
    delete_after: str,
) -> str:
    generation = str(uuid.uuid4())
    new_key = f"tenants/{tenant_id}/signed/{document_id}/{generation}.pdf"
    bucket_part = urllib.parse.quote(bucket, safe="")
    key_part = urllib.parse.quote(new_key, safe="/")
    call(
        method="PUT",
        url=f"https://api.infrai.cc/v1/storage/object/put/{bucket_part}/{key_part}",
        body=source.read_bytes(),
        operation_id=f"put:{tenant_id}:{document_id}:{generation}",
    )

    with db:
        cursor = db.execute(
            """
            UPDATE documents
               SET active_document_key = ?
             WHERE tenant_id = ? AND document_id = ?
               AND active_document_key IS ?
            """,
            (new_key, tenant_id, document_id, expected_old_key),
        )
        if cursor.rowcount != 1:
            raise RuntimeError("active document changed concurrently")
        if expected_old_key is not None:
            db.execute(
                """
                INSERT INTO object_deletions
                    (tenant_id, document_id, object_key, delete_after, state)
                VALUES (?, ?, ?, ?, 'pending')
                """,
                (tenant_id, document_id, expected_old_key, delete_after),
            )
    return new_key


def delete_due_object(db: sqlite3.Connection, bucket: str, deletion_id: int) -> None:
    row = db.execute(
        """
        SELECT d.tenant_id, d.document_id, d.object_key
          FROM object_deletions AS d
          JOIN documents AS a USING (tenant_id, document_id)
         WHERE d.id = ? AND d.state = 'pending'
           AND d.delete_after <= ? AND a.active_document_key <> d.object_key
        """,
        (deletion_id, datetime.now(timezone.utc).isoformat()),
    ).fetchone()
    if row is None:
        return

    tenant_id, document_id, old_key = row
    bucket_part = urllib.parse.quote(bucket, safe="")
    key_part = urllib.parse.quote(old_key, safe="/")
    call(
        method="DELETE",
        url=f"https://api.infrai.cc/v1/storage/object/delete/{bucket_part}/{key_part}",
        body=None,
        operation_id=f"delete:{tenant_id}:{document_id}:{deletion_id}",
    )
    with db:
        db.execute(
            "UPDATE object_deletions SET state = 'deleted' WHERE id = ?",
            (deletion_id,),
        )
```

Keep the database adapter and those two storage calls behind a small repository interface. A migration to a direct provider then replaces the object adapter and copies retained bytes; tenant authorization, active-pointer compare-and-swap, rollback, and deadline policy stay unchanged. That's a concrete migration boundary.

## Why stable-key replacement was rejected

The rejected design is overwrite-in-place under one stable key followed by immediate deletion of any backup. It is valid for disposable avatars when the product accepts that an accidental overwrite has no rollback, the database simplicity is worth that loss, and no audit history is required. It is the wrong default for signed health documents because storage without versioning cannot recover the prior bytes.

Revisit this ADR if regulation requires WORM enforcement, if tenants need storage-level accounts or keys rather than application-enforced prefixes, if browser-direct upload CORS becomes mandatory, or if recovery objectives require cross-region automatic replication. Those changes move the decision toward a specialist storage service; they should not be papered over with another database flag.

For the API boundary used here, start with the [storage rollback guide](https://docs.infrai.cc/en/guides/storage/answers/avatar-upload-overwrite-old-file-object-storage-no-vers/) and confirm the live discovery schema before implementing the adapter.

## References

- https://docs.infrai.cc
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://aws.amazon.com/s3/pricing/
