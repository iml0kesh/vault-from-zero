# HashiCorp Vault - KV Secrets Engine v2

## What is KV Secrets Engine?

The KV (Key/Value) Secrets Engine is used to securely store secrets such as:

* Database passwords
* API Keys
* Tokens
* Certificates
* Configuration values

Applications authenticate with Vault, receive a token, and use that token to read or write secrets.

---

# KV v1 vs KV v2

| KV v1                       | KV v2                                |
| --------------------------- | ------------------------------------ |
| Overwrites existing secrets | Supports versioning                  |
| No history                  | Maintains secret history             |
| No rollback                 | Can recover previous versions        |
| No metadata                 | Stores metadata for every version    |
| Simple storage              | Advanced secret lifecycle management |

---

# Enable KV Secret Engine

To enable kv secrets engine version 1.
```bash
vault secrets enable -path=mykv1 kv
```

To enable kv secrets engine version 2.
```bash
vault secrets enable -path=mykv2 -version=2 kv
```
# Writing a Secret

```bash
vault kv put mykv2/inventory/db-password \
username=inventory \
password=Inv@123
```

Vault stores the secret under:

```text
mykv2/inventory/db-password
```

The CLI returns metadata similar to:

```text
Version: 1
Created Time
Destroyed: false
Deletion Time: n/a
```

---

# What actually happens?

When you execute:

```bash
vault kv put ...
```

Vault performs the following steps:

```text
Application / User
        │
        ▼
Receives write request
        │
        ▼
Vault encrypts the secret
        │
        ▼
Stores encrypted data in the Storage Backend
(Raft / Consul / Integrated Storage / etc.)
        │
        ▼
Returns metadata to the client
```

Important:

The storage backend **never stores plaintext secrets**.

Only Vault knows how to decrypt them.

---

# Reading a Secret

```bash
vault kv get mykv2/inventory/db-password
```

Output contains two sections:

```text
Metadata

Data
```

---

## Metadata

Metadata describes the secret.

Examples:

* Version
* Created Time
* Deleted?
* Destroyed?
* Custom Metadata

Metadata does **not** contain secret values.

---

## Data

Contains the actual secret.

Example:

```text
username = inventory
password = Inv@123
```

---

# Why separate Metadata and Data?

Think of a movie file.

Metadata:

* Release Date
* Duration
* Resolution
* File Size

Data:

The actual movie.

Vault follows the same principle.

Metadata describes the secret.

Data contains the secret.

---

# Creating a New Version

Initial write:

```bash
vault kv put mykv2/inventory/db-password \
password=Inv@123
```

Creates:

```text
Version 1
```

Updating:

```bash
vault kv put mykv2/inventory/db-password \
password=Inv@456
```

Creates:

```text
Version 2
```

Vault **does not overwrite** Version 1.

Instead:

```text
mykv2/inventory/db-password

├── Version 1
│     password = Inv@123
│
└── Version 2
      password = Inv@456
```

The secret path remains the same.

Only the version changes.

---

# Reading Secrets

Default:

```bash
vault kv get mykv2/inventory/db-password
```

Returns:

Latest version only.

Example:

```text
Version 2

password = Inv@456
```

---

Older version:

```bash
vault kv get -version=1 mykv2/inventory/db-password
```

Returns:

```text
password = Inv@123
```

Vault never returns all versions automatically.

Older versions must be requested explicitly.

---

# Why return only the latest version?

Applications only need the current secret.

Returning every version would:

* Increase response size
* Waste bandwidth
* Slow applications
* Force applications to choose a version

Vault keeps it simple:

Without specifying a version:

```text
Return Latest Version
```

---

# Viewing Metadata

```bash
vault kv metadata get mykv2/inventory/db-password
```

Displays information such as:

* Current Version
* All Versions
* Creation Time
* Deletion Time
* Destroyed Status
* Last Updated By

No secret values are displayed.

---

# Secret Lifecycle

```text
Create

↓

Version 1

↓

Update

↓

Version 2

↓

Delete

↓

Undelete

↓

Destroy
```

---

# Soft Delete

Delete the latest version:

```bash
vault kv delete mykv2/inventory/db-password
```

Result:

```text
Version 2

Deleted

Destroyed = false
```

The secret still exists internally.

Only the latest version is marked as deleted.

---

## Reading after Delete

Running:

```bash
vault kv get mykv2/inventory/db-password
```

Returns:

Metadata only.

Example:

```text
Version = 2

Deletion Time = ...

Destroyed = false
```

The Data section is not returned because the latest version is deleted.

Vault does **not** automatically fall back to Version 1.

Vault never guesses.

---

# Undelete

Recover the deleted version.

```bash
vault kv undelete -versions=2 mykv2/inventory/db-password
```

Result:

```text
Version 2 restored
```

Reading again:

```bash
vault kv get mykv2/inventory/db-password
```

Returns:

```text
password = Inv@456
```

No new version is created.

Vault simply removes the deleted flag.

---

# Destroy

Destroy permanently removes the data for a version.

```bash
vault kv destroy -versions=2 mykv2/inventory/db-password
```

Metadata becomes:

```text
destroyed = true
```

The secret data is permanently removed.

It cannot be recovered.

---

# Undelete after Destroy

Interesting behavior observed during lab:

Running:

```bash
vault kv undelete -versions=2 mykv2/inventory/db-password
```

Returned:

```text
Success!
```

However,

Metadata still showed:

```text
destroyed = true
```

Reading the secret still returned:

* Metadata

But **no Data section**.

Meaning:

Vault accepted the request, but there was no secret data left to recover.

Always verify the resulting state instead of relying only on the success message.

---

# Delete vs Destroy

## Delete

```text
Soft Delete
```

* Recoverable
* Secret data still exists
* Can use Undelete

---

## Destroy

```text
Permanent Delete
```

* Secret data removed
* Cannot be recovered
* Undelete has no effect

---

# Production Example

Inventory Application:

```text
Version 1

Password = Inv@123
```

Administrator rotates the password:

```text
Version 2

Password = Inv@456
```

Application automatically receives Version 2 because Vault always returns the latest version.

If Version 2 is deleted:

Applications cannot read the secret.

Vault does **not** return Version 1 automatically.

If Version 2 is undeleted:

Applications immediately start receiving Version 2 again.

If Version 2 is destroyed:

The secret data is permanently gone.

---

# Commands Used During Lab

Enable KV v2:

```bash
vault secrets enable -path=mykv2 -version=2 kv
```

Store Secret:

```bash
vault kv put mykv2/inventory/db-password \
username=inventory \
password=Inv@123
```

Read Secret:

```bash
vault kv get mykv2/inventory/db-password
```

Read Older Version:

```bash
vault kv get -version=1 mykv2/inventory/db-password
```

View Metadata:

```bash
vault kv metadata get mykv2/inventory/db-password
```

Delete Latest Version:

```bash
vault kv delete mykv2/inventory/db-password
```

Undelete Version:

```bash
vault kv undelete -versions=2 mykv2/inventory/db-password
```

Destroy Version:

```bash
vault kv destroy -versions=2 mykv2/inventory/db-password
```

---

# Interview Questions

## Q1. Difference between KV v1 and KV v2?

KV v2 supports versioning, metadata, delete, undelete, and destroy. KV v1 simply overwrites secrets.

---

## Q2. Does `vault kv put` overwrite the secret?

KV v1: Yes.

KV v2: No. It creates a new version.

---

## Q3. Why does `vault kv get` return only the latest version?

Applications typically need the current secret. Older versions can be retrieved explicitly using the `-version` option.

---

## Q4. Difference between Delete and Destroy?

Delete is a soft delete and can be undone.

Destroy permanently removes the secret data.

---

## Q5. Does Vault automatically fall back to an older version if the latest version is deleted?

No.

Vault always uses the latest version. If that version is deleted, the request does not automatically return an older version.

---

# Key Takeaways

* KV v2 stores multiple versions of a secret.
* Secret values and metadata are separate.
* `vault kv get` returns the latest version by default.
* Older versions require explicit version selection.
* Delete is recoverable.
* Destroy is permanent.
* Vault never guesses or automatically falls back to previous versions.
* Always verify the resulting state after destructive operations.
* Storage backends store encrypted data; Vault performs encryption before storage and decryption when authorized clients read secrets.
