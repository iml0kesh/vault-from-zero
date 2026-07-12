# Vault Policies

A Vault Policy is a set of rules that determines **what a token is allowed to do** inside Vault.

Authentication tells Vault **who you are**.

Policies tell Vault **what you are allowed to access**.

---

# Vault Authentication Flow

```text
Application / User
        │
        ▼
Authentication
(AppRole, LDAP, Userpass, AWS, etc.)
        │
        ▼
Vault verifies credentials
        │
        ▼
Vault generates Token
        │
        ▼
Policy attached to Token
        │
        ▼
Vault checks Policy
        │
        ▼
Allow / Deny Request
```

---

# Why do Policies exist?

Imagine a company has three applications.

```text
Inventory App

Payment App

HR App
```

Vault stores:

```text
secret/data/inventory/*
secret/data/payment/*
secret/data/hr/*
```

Question:

Should the Inventory Application read:

```text
secret/data/payment/db-password
```

Answer: **No**.

The Inventory Application should only access the secrets required for its work.

This follows the **Principle of Least Privilege (PoLP).**

---

# Principle of Least Privilege (PoLP)

Give only the **minimum permissions** required to perform the job.

Never grant more permissions than necessary.

Bad:

```text
Inventory App

↓

Read Everything
```

Good:

```text
Inventory App

↓

Read only

secret/data/inventory/*
```

If one application is compromised, the attacker can only access that application's secrets.

---

# Basic Policy Syntax

Example:

```hcl
path "secret/data/inventory/*" {

  capabilities = ["read"]

}
```

---

# path

```hcl
path "secret/data/inventory/*"
```

Defines where the policy applies.

The wildcard (*) means:

```text
secret/data/inventory/db-password

secret/data/inventory/api-key

secret/data/inventory/config
```

are all covered.

But not:

```text
secret/data/payment/*
```

or

```text
secret/data/hr/*
```

Vault is **path-based**.

It does not care which application is asking.

It only checks:

- Which Token?
- Which Policy?
- Which Path?

---

# capabilities

Capabilities define **what operations are allowed** on a path.

Example:

```hcl
capabilities = ["read"]
```

Means:

The token can only read secrets.

---

# Common Capabilities

| Capability | Purpose |
|------------|---------|
| create | Create a new secret |
| read | Read a secret |
| update | Modify an existing secret |
| delete | Delete a secret |
| list | List available secret names |
| patch | Partially update a secret |
| sudo | Administrative access for specific system endpoints |

---

# read Capability

Example:

```hcl
path "secret/data/inventory/*" {

  capabilities = ["read"]

}
```

Allows:

```text
Read

secret/data/inventory/db-password
```

Returns:

```text
username

password

apikey
```

The application can read the secret values.

---

# list Capability

Example:

```hcl
path "secret/metadata/inventory/*" {

  capabilities = ["list"]

}
```

Allows:

```text
inventory/

db-password

api-key

redis-password
```

Notice:

The application only knows the **secret names**.

It **cannot** see their values.

---

# Difference between read and list

## read

Question:

> What's inside the secret?

Example:

```text
db-password

↓

username = inventory

password = P@ssw0rd
```

---

## list

Question:

> Which secrets exist?

Example:

```text
inventory/

db-password

api-key

config
```

No values are returned.

Only names.

---

# Why would someone need list without read?

This is a common production requirement.

Example:

The Vault UI needs to display folders.

```text
KV Secrets

inventory

payment

hr
```

It only needs to know what exists.

It does not need permission to read the secret values.

Another example:

A developer wants to verify whether a secret exists.

Listing is sufficient.

Reading is unnecessary.

---

# KV Version 2 Paths

KV Version 2 uses different endpoints for metadata and data.

## Secret Values

```text
secret/data/*
```

Contains:

```text
username

password

apikey
```

Actual secret values.

Reading happens here.

---

## Secret Metadata

```text
secret/metadata/*
```

Contains information about the secret.

Examples:

- Secret name
- Versions
- Created Time
- Deletion Status

Listing happens here.

---

# Why separate data and metadata?

Think of Windows Explorer.

You can:

```text
Open Folder

↓

See file names
```

without opening the files.

Opening the file is different.

Vault follows the same principle.

```text
Metadata

↓

What exists?
```

```text
Data

↓

What's inside?
```

Separating them allows administrators to grant:

- list

without granting

- read

---

# Real Production Scenario

Vault contains:

```text
secret/data/inventory/*
secret/data/payment/*
secret/data/hr/*
```

Inventory Application receives:

```hcl
path "secret/data/inventory/*" {

    capabilities = ["read"]

}
```

Result:

Can read:

```text
secret/data/inventory/db-password
```

Cannot read:

```text
secret/data/payment/db-password
```

Vault checks the path against the attached policy.

If the path is not allowed:

```text
403 Permission Denied
```

---

# Authentication vs Authorization

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

Vault Flow:

```text
Authentication

↓

Generate Token

↓

Attach Policy

↓

Read Policy

↓

Allow / Deny Request
```

---

# Interview Questions

## Q1

What is a Vault Policy?

**Answer:**

A Vault Policy is a set of rules attached to a token that defines what paths and operations are allowed inside Vault.

---

## Q2

What is the Principle of Least Privilege?

**Answer:**

Grant only the minimum permissions required for an application or user to perform its job.

---

## Q3

What is the difference between read and list?

**Answer:**

- read returns the secret values.
- list returns only the available secret names.

---

## Q4

Why does KV v2 have both `secret/data/*` and `secret/metadata/*`?

**Answer:**

Vault separates secret values from metadata so administrators can grant listing permissions without exposing secret values.

---

## Q5

How does Vault decide whether to allow a request?

Vault checks:

1. The Token.
2. The Policies attached to the Token.
3. Whether the requested path and operation are allowed.

If allowed:

```text
200 OK
```

Otherwise:

```text
403 Permission Denied
```

---

# Key Takeaways

- Policies define authorization.
- Authentication generates a token.
- Tokens have one or more policies attached.
- Vault is path-based.
- `read` returns secret values.
- `list` returns only secret names.
- KV v2 separates data and metadata.
- Always follow the Principle of Least Privilege.