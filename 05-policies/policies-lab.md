# Vault Policies - Practical Lab

Today we implemented everything we learned about Vault Policies using AppRole authentication and a KV Secrets Engine.

---

# Goal

Create an AppRole that can only read Inventory secrets.

Requirements:

```text
Inventory App

↓

Can Read

mykv1/inventory/*
```

Cannot read:

```text
mykv1/payment/*
```

---

# Lab Architecture

```text
                Vault

      +----------------------+
      |                      |
      |  mykv1              |
      |   │                  |
      |   ├── inventory      |
      |   │     └── db-password
      |   │
      |   └── payment
      |         └── db-password
      |                      |
      +----------------------+

              ▲

        AppRole Login

              ▲

 RoleID + SecretID

              ▲

      Inventory Application
```

---

# Step 1 - Create Policy

Create a policy file.

inventory-policy.hcl

```hcl
path "mykv1/inventory/*" {

    capabilities = ["read"]

}
```

> **Important:** This path is for **KV Version 1**.

If using KV Version 2:

```hcl
path "mykv1/data/inventory/*" {

    capabilities = ["read"]

}
```

---

# Upload Policy

```bash
vault policy write inventory-policy inventory-policy.hcl
```

Verify:

```bash
vault policy read inventory-policy
```

Output:

```hcl
path "mykv1/inventory/*" {

    capabilities = ["read"]

}
```

---

# Step 2 - Attach Policy to AppRole

View AppRole

```bash
vault read auth/approle/role/inventory-app
```

Initially:

```text
token_policies = [default]
```

Update:

```bash
vault write auth/approle/role/inventory-app token_policies=inventory-policy
```

Verify:

```bash
vault read auth/approle/role/inventory-app
```

Output:

```text
token_policies = [inventory-policy]
```

Notice:

The AppRole stores only the custom policy.

The **default policy** is automatically added later during token creation.

---

# Step 3 - Generate Credentials

Role ID

```bash
vault read auth/approle/role/inventory-app/role-id
```

Secret ID

```bash
vault write -f auth/approle/role/inventory-app/secret-id
```

These two credentials are securely provided to the application.

---

# Step 4 - Login

```bash
vault write auth/approle/login \
role_id=<ROLE_ID> \
secret_id=<SECRET_ID>
```

Vault returns:

```text
token

token_accessor

token_duration

token_policies

identity_policies
```

Example:

```text
token_policies

↓

default

inventory-policy
```

---

# Why did "default" appear?

AppRole configuration:

```text
token_policies = inventory-policy

token_no_default_policy = false
```

Since

```text
token_no_default_policy = false
```

Vault automatically adds:

```text
default
```

Generated Token

```text
default

+

inventory-policy
```

If

```text
token_no_default_policy = true
```

then only

```text
inventory-policy
```

would be attached.

---

# Step 5 - Login Using Token

```bash
vault login <TOKEN>
```

Verify:

```bash
vault token lookup
```

Example

```text
display_name

approle

path

auth/approle/login

policies

default

inventory-policy

type

service

meta

role_name = inventory-app
```

---

# Step 6 - Store Secrets

As Root:

Inventory

```bash
vault kv put mykv1/inventory/db-password \
username=inventory \
password=Inv@123
```

Payment

```bash
vault kv put mykv1/payment/db-password \
username=payment \
password=Pay@123
```

---

# Step 7 - Read Inventory Secret

Using AppRole Token

```bash
vault kv get mykv1/inventory/db-password
```

Result

✅ Success

Vault Flow

```text
Application

↓

Token

↓

Policies

↓

inventory-policy

↓

Requested Path

↓

mykv1/inventory/db-password

↓

Path Matches

↓

ALLOW
```

---

# Step 8 - Read Payment Secret

```bash
vault kv get mykv1/payment/db-password
```

Result

```text
403 Permission Denied
```

Reason

```text
Application

↓

Token

↓

Policies

↓

inventory-policy

↓

Requested Path

↓

mykv1/payment/db-password

↓

Path NOT Found

↓

DENY
```

Vault never checks:

- Application Name
- Server Name
- Hostname

Vault only checks:

- Token
- Policies
- Requested Path
- Requested Operation

---

# Debugging a Real Issue

Initially the policy was

```hcl
path "mykv1/data/inventory/*"
```

But the secrets engine was mounted as

```text
KV Version 1
```

Actual request:

```text
mykv1/inventory/db-password
```

Policy:

```text
mykv1/data/inventory/*
```

These paths do not match.

Vault therefore returned

```text
403 Permission Denied
```

---

# How We Debugged

We verified:

Authentication

✅ Successful

Policy Attached

✅ inventory-policy

Token

✅ Generated Successfully

Capabilities

```bash
vault token capabilities mykv1/data/inventory/db-password
```

Output

```text
read
```

Finally we compared

Policy Path

```text
mykv1/data/inventory/*
```

with

Actual Request

```text
mykv1/inventory/db-password
```

and discovered the KV Version mismatch.

---

# KV Version Difference

## KV Version 1

```text
mykv1/inventory/db-password
```

No `/data`

---

## KV Version 2

```text
mykv1/data/inventory/db-password
```

Uses `/data`

Policies must always match the actual API path.

---

# Important Lesson

Vault evaluates policies against the **API path**, not the CLI command.

CLI

```bash
vault kv get mykv1/inventory/db-password
```

Internally Vault checks the API path.

If the policy path does not match, Vault returns:

```text
403 Permission Denied
```

---

# Practical Authorization Flow

```text
Inventory Application

↓

RoleID + SecretID

↓

Vault Authentication

↓

Generate Token

↓

Attach Policies

↓

Application Requests Secret

↓

Vault Checks

• Token

• Attached Policies

• Requested Path

• Requested Capability

↓

ALLOW / DENY
```

---

# What We Learned Today

- Created a real Vault Policy.
- Uploaded the policy into Vault.
- Attached the policy to an AppRole.
- Generated RoleID and SecretID.
- Logged in using AppRole.
- Received a service token.
- Verified token metadata.
- Stored secrets inside Vault.
- Successfully enforced authorization.
- Debugged a real 403 Permission Denied.
- Understood the difference between KV v1 and KV v2 paths.
- Learned that Vault authorizes based on the API path.

---

# Interview Questions

## Q1

What is attached to an AppRole?

**Answer:**

One or more Vault policies.

---

## Q2

How does Vault decide whether a request is allowed?

**Answer:**

Vault checks:

1. The token.
2. The policies attached to the token.
3. The requested path.
4. The requested capability.

If everything matches:

```text
200 OK
```

Otherwise:

```text
403 Permission Denied
```

---

## Q3

Why did the policy fail even though it looked correct?

**Answer:**

The policy was written for KV Version 2 (`mykv1/data/...`), while the secrets engine was KV Version 1 (`mykv1/...`). Since the paths did not match, Vault denied the request.

---

## Q4

How would you troubleshoot a `403 Permission Denied` in Vault?

**Answer:**

1. Verify the application authenticated successfully.
2. Check the token with `vault token lookup`.
3. Confirm the expected policy is attached.
4. Use `vault token capabilities <path>` to verify what Vault allows.
5. Compare the policy path with the actual API path (including KV v1 vs KV v2).
6. Update the policy if the paths don't match.

---

# Key Takeaways

- Authentication answers **who you are**.
- Policies answer **what you can do**.
- Tokens carry policies.
- Vault is path-based.
- Authorization is evaluated against the API path.
- KV v1 and KV v2 use different paths.
- `vault token capabilities` is one of the best debugging commands for authorization issues.
- A `403 Permission Denied` is often a policy or path mismatch—not necessarily an authentication problem.