# Vault Tokens

A Vault Token is the **result of a successful authentication**.

Every authentication method in Vault (AppRole, Userpass, LDAP, AWS, Kubernetes, etc.) eventually generates a **Vault Token**.

After authentication, the application **does not use the original credentials anymore**. It only uses the Vault Token until it expires or is revoked.

---

# Authentication Flow

```text
Application
      │
Role ID + Secret ID
(or Username + Password)
      │
      ▼
Vault
      │
Verify Credentials
      │
      ▼
Generate Vault Token
      │
      ├────────────► Store Token
      │              (Storage Backend)
      │
      └────────────► Return Token
                     to Application
```

From this point onward, the application only sends the Vault Token.

---

# Why do we need a Token?

Imagine an application reading secrets every few seconds.

Without tokens:

```text
Read Secret

↓

Role ID

Secret ID
```

Every request would require authentication.

Instead:

```text
Login Once

↓

Receive Vault Token

↓

Use Token for every request
```

Authentication happens only once.

---

# Vault Token = Session

Think of a Vault Token as a login session.

Example:

```text
Login

↓

Vault Token Generated

↓

Use Token

↓

Token Expires

↓

Login Again
```

The token represents the authenticated session.

It does **not** represent:

- the user
- the application
- the server

It represents **one authenticated session**.

---

# Vault Token vs Cookie

A good analogy is a website login.

Website:

```text
Username

Password

↓

Session Created

↓

Cookie
```

Vault:

```text
Role ID

Secret ID

↓

Authentication

↓

Vault Token
```

Instead of repeatedly sending credentials, the client sends the Vault Token.

---

# Where is the Token Stored?

For **Service Tokens**, Vault stores the token in the configured **Storage Backend**.

Example:

```text
Storage Backend

↓

Raft
File
Consul
Integrated Storage
```

Vault also returns the token to the application.

```text
Vault

knows Token A

Application

knows Token A
```

Both now know the same token.

---

# Does Vault keep Tokens only in Memory?

No.

Service Tokens are stored in the Storage Backend.

Vault may cache frequently used information in memory for performance, but the source of truth is the Storage Backend.

---

# How does Token Expiry Work?

A token is stored together with metadata.

Example:

```text
Token

Creation Time

TTL = 1 Hour

Expire Time
```

Whenever a request arrives:

```text
Request

↓

Find Token

↓

Expired?

↓

YES → Reject

NO → Continue
```

The token doesn't disappear simply because it expires. Vault checks its expiration before serving requests.

---

# What happens after Authentication?

After successful login, Vault no longer checks the original credentials.

Example:

```text
Application

↓

Role ID

Secret ID

↓

Vault

↓

Token Generated
```

Future requests become:

```http
GET secret/data/database

X-Vault-Token: hvs.xxxxxxxxx
```

Vault validates only the Vault Token.

Role ID and Secret ID are **not sent again**.

---

# Authentication vs Authorization

These are two different things.

## Authentication

Question:

> Who are you?

Example:

```text
Role ID

Secret ID

↓

Valid?

↓

Generate Token
```

---

## Authorization

Question:

> What are you allowed to access?

Vault uses the Token to determine:

- attached policies
- allowed paths
- allowed operations

---

# Policy Changes

Suppose a token has:

```text
inventory-policy
```

Five minutes later, an administrator modifies that policy. Does the application need to login again?

**No.**

The token remains valid. During every request, Vault checks the **current version** of the attached policy.

Example:

```text
Token

↓

inventory-policy

↓

Read Current Policy

↓

Allow / Deny
```

Changing a policy immediately affects existing tokens.

---

# Does changing a Policy invalidate the Token?

No. The token remains valid. Only its permissions change.

Example:

```text
Token Valid

↓

Policy Changed

↓

Next Request

↓

Permission Denied
```

The token expires only when:

- TTL expires
- Administrator revokes it
- Parent token is revoked (covered later)

---

# Inspecting a Token

Command:

```bash
vault token lookup
```

Example Output:

```text
accessor
creation_time
creation_ttl
display_name
entity_id
expire_time
explicit_max_ttl
id
meta
num_uses
orphan
path
policies
ttl
type
```

---

# Important Fields

## display_name

Human-readable name showing how the token was created.

Examples:

```text
display_name = root
```

```text
display_name = approle
```

```text
display_name = userpass-lokesh
```

---

## path

Shows which authentication endpoint created the token.

Examples:

Root Token:

```text
auth/token/root
```

AppRole Login:

```text
auth/approle/login
```

Userpass Login:

```text
auth/userpass/login/lokesh
```

> **Note**
>
> This is **NOT** the AppRole configuration path.
>
> Example:
>
> ```text
> auth/approle/role/inventory-app
> ```
>
> This path stores the AppRole configuration.
>
> Authentication happens through:
>
> ```text
> auth/approle/login
> ```

---

## policies

The token stores **references** to policies.

Example:

```text
[root]
```

```text
[inventory-policy]
```

Vault does **not** copy the policy contents into the token. During every request, Vault reads the latest version of the policy.

---

## ttl

Shows how long the token is valid.

Example:

```text
ttl = 1h
```

Root Token:

```text
ttl = 0s
```

This does **NOT** mean expired.

It means the Root Token has **no automatic expiration**.

---

## creation_ttl

The TTL assigned when the token was created.

Example:

```text
creation_ttl = 1h
```

Root Token:

```text
creation_ttl = 0s
```

Meaning: No expiration configured.

### Difference between creation_ttl and ttl?

**creation_ttl:** The initial TTL assigned to the token at creation time. It is historical information and does not count down.

**ttl:** The remaining lifetime of the token at the moment you inspect it. It decreases over time and may increase again if the token is renewed.

**creation_ttl** is the initial lifetime assigned to the token when it was created. It never changes. **ttl** is the token's remaining lifetime at the current moment. It counts down over time and can increase if the token is renewed.

---


## type

Shows what kind of token it is. Two common types are:

### Service Token

The default token type.

Characteristics:

- Stored in Storage Backend
- Renewable
- Can be revoked
- Supports parent-child relationships
- Used by most applications

Examples:

- AppRole
- Userpass
- LDAP
- Kubernetes
- AWS

---

### Batch Token

Designed for short-lived workloads.

Characteristics:

- Not stored in Storage Backend
- Not renewable
- Cannot create child tokens
- Lightweight
- Lower management overhead

Examples:

- CI/CD Pipelines
- AWS Lambda
- Short-lived Kubernetes Jobs

---

# Service vs Batch Token

| Feature | Service | Batch |
|----------|----------|-------|
| Stored in Storage Backend | ✅ | ❌ |
| Renewable | ✅ | ❌ |
| Revocable | ✅ | Naturally expires |
| Parent / Child | ✅ | ❌ |
| Best For | Long-running applications | Short-lived jobs |

---

# Company Scenario

## Inventory Application

Runs 24x7. Needs to read secrets continuously.

Use:

```text
Service Token
```

Reason:

- Renewable
- Managed by Vault
- Suitable for long-running services

---

## Jenkins Pipeline

```text
Start

↓

Read Secrets

↓

Deploy

↓

Exit
```

Pipeline lasts 20 seconds.

Use:

```text
Batch Token
```

Reason: No need to maintain token state after the pipeline completes.

---

# Key Takeaways

- Every authentication method eventually generates a Vault Token.
- A Vault Token represents an authenticated session.
- After login, applications use only the Vault Token.
- Service Tokens are stored in the Storage Backend.
- Vault validates the token for every request.
- Vault checks the latest version of attached policies during authorization.
- Changing a policy immediately affects existing tokens.
- Tokens are not invalidated simply because a policy changes.
- Service Tokens are suitable for long-running applications.
- Batch Tokens are designed for short-lived workloads.