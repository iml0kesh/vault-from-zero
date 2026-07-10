# AppRole Authentication

AppRole is an authentication method used by **machines**, **applications**, **scripts**, and **CI/CD pipelines**.

> Not intended for human users.
>
> Humans generally use methods like LDAP, Userpass, OIDC, GitHub, etc.

---

# AppRole Authentication Flow

1. Enable the AppRole auth method.
2. Create a policy.
3. Create an AppRole and attach the policy.
4. Retrieve the Role ID.
5. Generate a Secret ID.
6. Securely deliver the Role ID and Secret ID to the application.
7. The application authenticates using Role ID + Secret ID.
8. Vault verifies the credentials.
9. Vault generates a **new Vault Token**.
10. The application uses the Vault Token to access secrets.

```
                Login

Application
     │
     │
Role ID + Secret ID
     │
     ▼
+--------------------+
|      AppRole       |
+--------------------+
     │
     ▼
Verifies Credentials
     │
     ▼
Generates Vault Token
     │
     ▼
Application uses Token
to access Vault
```

---

# Enable AppRole

```bash
vault auth enable approle
```

Verify:

```bash
vault auth list
```

---

# Create an AppRole

```bash
vault write auth/approle/role/inventory-app \
token_policies=inventory-policy \
token_ttl=1h
```

---

# Get the Role ID

```bash
vault read auth/approle/role/inventory-app/role-id
```

---

# Generate a Secret ID

```bash
vault write -f auth/approle/role/inventory-app/secret-id
```

---

# Authenticate using AppRole

```bash
vault write auth/approle/login \
role_id=<ROLE_ID> \
secret_id=<SECRET_ID>
```

Vault returns:

```text
client_token
lease_duration
renewable
...
```

The application stores the **Vault Token** and uses it for future requests.

---

# Important Concept

AppRole **only verifies the credentials**.

It checks:

- Is the Role ID valid?
- Is the Secret ID valid?

If both are valid, Generate a **new Vault Token**.

That's it.

AppRole does **not** care:

- Which server?
- Which VM?
- Which Pod?
- Which IP?
- Whether someone has already logged in.

Its only responsibility is authenticating the credentials.

---

# Identity vs Session

This is the most important concept.

## AppRole = Identity

```
inventory-app
```

Represents the application.

There is only **one identity**.

---

## Vault Token = Session

Every successful login creates a **new session**.

```
Login #1
↓

Token A

Login #2
↓

Token B

Login #3
↓

Token C
```

One AppRole can generate **thousands of Vault Tokens**.

---

# One AppRole Across Multiple Servers

One AppRole is enough even if the application runs on many servers.

```
                inventory-app

             Role ID
             Secret ID

        ┌────────┼────────┐
        │        │        │
     Server1  Server2  Server3
```

Every server uses the same AppRole credentials. Each server authenticates independently.

---

# Does every server receive the same Vault Token?

No.

Example:

```
Server1

Role ID
Secret ID
     │
     ▼
Vault
     │
     ▼
Token A
```

```
Server2

Role ID
Secret ID
     │
     ▼
Vault
     │
     ▼
Token B
```

Even though the Role ID and Secret ID are the same, Vault creates a **new token for every successful login**.

---

# Why doesn't Vault return the same token?

A Vault Token represents a **login session**, not an application.

Imagine Gmail.

```
Laptop
↓

Session A

Phone
↓

Session B
```

Same account.

Different sessions.

Vault follows the same idea.

One AppRole.

Many sessions (tokens).

---

# Does Vault know which server is logging in?

No.

Vault does **not** identify Server 1 or Server 2.

It simply receives an authentication request.

```
Role ID

Secret ID

↓

Valid?

↓

YES

↓

Generate Token
```

Vault treats every successful authentication request as a **new login**.

---

# What happens if 100 servers authenticate simultaneously?

Suppose:

```
secret_id_num_uses = 0
```

(unlimited)

```
100 Login Requests

↓

100 Successful Authentications

↓

100 Vault Tokens
```

Vault is designed to handle concurrent authentication requests.

---

# What if secret_id_num_uses = 1 ?

```
100 Login Requests

↓

Only one succeeds

↓

99 fail
```

After the first successful login, the Secret ID is marked as used.

---

# Frequently Asked Questions

## Q1. Do I need one AppRole per server?

**No.**

One AppRole represents the application, not the server.

---

## Q2. Do I need a new Role ID for every server?

**No.**

The Role ID belongs to the AppRole.

All servers can use the same Role ID.

---

## Q3. Do all servers receive the same Vault Token?

**No.**

Every successful login generates a new Vault Token.

---

## Q4. What does the application use after login?

The application no longer uses the Role ID and Secret ID.

Instead, it uses the **Vault Token** for all future API requests.

---

## Q5. What happens after the Vault Token expires?

The application must either:

- Renew the token (if renewable), or
- Authenticate again using Role ID + Secret ID.

---

# Company Scenario

## Requirement

ABC Retail has an application called **inventory-app**.

The application runs on:

- 100 Servers
- Auto Scaling enabled
- Same permissions for every server

## Vault Design

- Enable AppRole
- Create one AppRole
- Attach one policy
- Generate one Role ID
- Generate one Secret ID
- Securely distribute the credentials
- Every server authenticates independently
- Every successful login receives a different Vault Token

```
                    Vault

             AppRole: inventory-app

             Role ID
             Secret ID

      ┌──────────┼──────────┐
      │          │          │
   Server1    Server2    Server3
      │          │          │
      ▼          ▼          ▼
   Token A    Token B    Token C
```

---

# Key Takeaways

- AppRole is for machines, not humans.
- AppRole represents an application identity.
- Role ID belongs to the AppRole.
- Secret ID acts like a secret credential for authentication.
- AppRole only verifies credentials.
- Every successful login creates a new Vault Token.
- Vault Tokens represent sessions.
- One AppRole can be used by many servers or pods.
- The application uses the Vault Token after authentication.