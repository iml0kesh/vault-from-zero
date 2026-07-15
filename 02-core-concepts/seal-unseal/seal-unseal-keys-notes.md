# Vault Seal, Unseal, Master Key, Unseal Keys, Root Token & Shamir's Secret Sharing


how Vault is able to decrypt those secrets, why Vault becomes sealed after a restart**, and why Unseal Keys exist.

---

# Where are Secrets Stored?

Secrets are permanently stored in the configured **Storage Backend**.

Examples:

- Integrated Storage (Raft)
- Consul

```text
Application
      │
      ▼
Vault
      │
Encrypt Secret
      │
      ▼
Storage Backend
(Encrypted Data)
```

Vault never stores plaintext secrets in the storage backend.

---

# What if an attacker steals the Storage Backend?

Suppose an attacker steals:

```text
raft.db
```

or

```text
Consul Database
```

Can they read the secrets? **No.**

Reason: The storage backend only contains **encrypted data**.

Example:

```text
Storage Backend

Encrypted Secret
Encrypted Secret
Encrypted Secret
```

Without the encryption key, these values are useless.

---

# How does Vault decrypt secrets?

Whenever an application requests a secret:

```text
Application

↓

Vault

↓

Decrypt Secret

↓

Return Secret
```

Vault must use a cryptographic key to decrypt the data.

This key is called the **Master Key**.

---

# Where is the Master Key stored?

The Master Key is **NOT** stored inside the Storage Backend.

Otherwise an attacker would steal:

```text
Encrypted Secrets

+

Master Key
```

which would completely defeat encryption.

Instead:

The Master Key exists only in **Vault's memory (RAM)** while Vault is running.

```text
Vault Process

Master Key (RAM)

↓

Decrypt

↓

Storage Backend
```

---

# What happens when Vault crashes?

Suppose Vault crashes.

```text
Vault Process

↓

Stops
```

RAM is cleared.

Therefore:

```text
Master Key

↓

Lost
```

The encrypted secrets still exist inside the Storage Backend.

But Vault can no longer decrypt them.

---

# Why does Vault become Sealed?

After Vault starts again:

```text
Storage Backend

↓

Encrypted Secrets
```

But...

```text
Master Key

Missing
```

Without the Master Key:

Vault cannot decrypt anything.

Therefore Vault starts in the **Sealed** state.

---

# What does Sealed mean?

A Sealed Vault means:

- Storage Backend is available.
- Encrypted data is available.
- Master Key is NOT available.
- Vault cannot decrypt secrets.

Applications cannot read secrets while Vault is sealed.

---

# Manual Unseal

To recover the Master Key after a restart:

Vault asks administrators to provide **Unseal Keys**.

Example:

```text
Unseal Key 1

Unseal Key 2

Unseal Key 3

Unseal Key 4

Unseal Key 5
```

Suppose the threshold is:

```text
5 Shares

Threshold = 3
```

Vault waits until any three valid Unseal Keys are entered.

Example:

```text
Key 1

+

Key 3

+

Key 5
```

Once the threshold is reached:

Vault reconstructs the Master Key.

The Master Key is loaded into RAM.

Vault becomes Unsealed.

---

# Why not simply give the administrator the Master Key?

Imagine only one administrator has the Master Key.

Problems:

- The key may be stolen.
- The key may be copied.
- The administrator may lose it.
- The administrator may leave the company.
- One person has complete control.

This creates a huge security risk.

---

# Why Unseal Keys?

Instead of giving the actual Master Key:

Vault splits it into multiple **shares**.

These shares are called **Unseal Keys**.

Example:

```text
Master Key

↓

Split

↓

Share 1

Share 2

Share 3

Share 4

Share 5
```

No single administrator possesses the complete Master Key.

---

# Shamir's Secret Sharing

Vault does **NOT** physically cut the Master Key into pieces.

Instead, Vault uses a cryptographic algorithm called:

**Shamir's Secret Sharing**

It creates multiple mathematical shares.

Example:

```text
Master Key

↓

Shamir's Secret Sharing

↓

Share 1

Share 2

Share 3

Share 4

Share 5
```

These shares are mathematically related to the Master Key.

---

# Important Property of Shamir's Secret Sharing

Suppose:

```text
Shares = 5

Threshold = 3
```

Any three shares can reconstruct the Master Key.

Example:

```text
1 + 2 + 3

✓
```

```text
1 + 4 + 5

✓
```

```text
2 + 3 + 5

✓
```

---

# What if an attacker steals only one share?

Nothing useful.

One share does **NOT** reveal:

- the Master Key
- part of the Master Key
- any useful pattern

Even two shares reveal nothing if the threshold is three.

This is the biggest strength of Shamir's Secret Sharing.

---

# What happens after Vault reconstructs the Master Key?

After the threshold is reached:

```text
Master Key

↓

Loaded into RAM

↓

Vault Unsealed
```

The individual Unseal Keys are no longer needed by the running Vault process.

Vault discards them from memory.

Administrators continue to store their own copies safely for future restarts.

---

# Boot Process

```text
Vault Starts

↓

Reads Storage Backend

↓

Encrypted Data Found

↓

Master Key Missing

↓

Vault Sealed

↓

Admins Enter Unseal Keys

↓

Threshold Reached

↓

Master Key Reconstructed

↓

Loaded into RAM

↓

Vault Unsealed

↓

Applications Can Read Secrets
```

---

# Master Key vs Unseal Keys vs Root Token

These are three completely different things.

| Component | Purpose |
|-----------|---------|
| Master Key | Encrypts and decrypts Vault data |
| Unseal Keys | Reconstruct the Master Key after startup |
| Root Token | Provides full administrative access to Vault |

They solve different problems.

---

# Master Key

Purpose:

- Encrypt secrets.
- Decrypt secrets.

The Master Key is never used for authentication.

---

# Unseal Keys

Purpose:

- Reconstruct the Master Key.

Unseal Keys do NOT:

- read secrets
- log into Vault
- grant permissions

Their only job is helping Vault recover the Master Key.

---

# Root Token

The Root Token is simply the most privileged authentication token.

It is used for:

- Administration
- Managing policies
- Enabling auth methods
- Enabling secrets engines
- Reading and writing secrets (because it has full privileges)

It does **NOT**:

- Encrypt data
- Decrypt data
- Unseal Vault

---

# What if the Root Token is revoked?

Vault can still:

- Encrypt secrets.
- Decrypt secrets.
- Serve applications using other valid tokens.

Because encryption is handled by the **Master Key**, not by the Root Token.

---

# Can a valid Root Token read secrets while Vault is sealed?

No.

Even if authentication succeeds:

Vault is still sealed.

Without the Master Key in RAM:

Vault cannot decrypt the encrypted data.

Therefore no secrets can be returned.

---

# Complete Request Flow

Whenever a client requests a secret:

```text
Client

↓

Is Vault Unsealed?

↓

No

↓

Stop
```

If Vault is Unsealed:

```text
Vault

↓

Is Token Valid?

↓

Does Policy Allow?

↓

Decrypt Secret

↓

Return Secret
```

For a request to succeed:

- Vault must be Unsealed.
- Token must be valid.
- Policy must allow the operation.

---

# Key Takeaways

- Secrets are stored encrypted inside the Storage Backend.
- The Master Key is stored only in RAM while Vault is running.
- After a restart, RAM is cleared and the Master Key is lost.
- Vault starts in the Sealed state after every restart (manual unseal).
- Unseal Keys are cryptographic shares, not pieces of the Master Key.
- Vault uses Shamir's Secret Sharing to create Unseal Keys.
- Below the threshold, the shares reveal no useful information.
- After reconstruction, the Master Key lives in RAM.
- The Root Token is only for administration and authentication.
- The Root Token is not involved in encryption or decryption.
- A valid Root Token cannot read secrets while Vault is sealed.

---

# Interview Questions

## Q1. Where are Vault secrets stored?

**Answer:**

Secrets are stored in the configured Storage Backend (Raft, Consul, etc.) in an encrypted form.

---

## Q2. Where is the Master Key stored?

**Answer:**

The Master Key exists only in Vault's memory (RAM) while Vault is unsealed.

---

## Q3. Why does Vault become sealed after a restart?

**Answer:**

RAM is cleared during a restart, so the Master Key is lost. Without the Master Key, Vault cannot decrypt the encrypted data in the storage backend.

---

## Q4. What is the purpose of Unseal Keys?

**Answer:**

Unseal Keys are cryptographic shares generated using Shamir's Secret Sharing. They are used to reconstruct the Master Key after Vault starts.

---

## Q5. Does one Unseal Key reveal part of the Master Key?

**Answer:**

No. A single Unseal Key reveals no useful information about the Master Key.

---

## Q6. What is the difference between the Master Key and the Root Token?

**Answer:**

The Master Key encrypts and decrypts Vault data. The Root Token provides administrative access to Vault but is not involved in encryption.

---

## Q7. Can a valid Root Token read secrets while Vault is sealed?

**Answer:**

No. Even with a valid Root Token, Vault cannot decrypt secrets while it is sealed because the Master Key is not available in memory.