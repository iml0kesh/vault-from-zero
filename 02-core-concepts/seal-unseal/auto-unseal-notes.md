# Vault Auto Unseal & Recovery Keys

Until now, we learned how Vault works with **Manual Unseal** using Shamir's Secret Sharing.

In this chapter, we learned how Vault works in production using **Auto Unseal**.

---

# Why Auto Unseal?

Imagine Vault is running in production.

At 3:00 AM, the server unexpectedly restarts.

With Manual Unseal:

```text
Vault Starts

↓

Sealed

↓

Admin 1 enters Unseal Key

↓

Admin 2 enters Unseal Key

↓

Admin 3 enters Unseal Key

↓

Vault Unsealed
```

This requires administrators to wake up and manually unseal Vault.

For production environments, this is not practical.

---

# Auto Unseal

With Auto Unseal:

```text
Vault Starts

↓

AWS KMS / Azure Key Vault / GCP KMS / HSM

↓

Vault Automatically Unsealed

↓

Applications Continue Working
```

No administrator intervention is required.

---

# Does AWS KMS store the Master Key?

**No.**

This is one of the biggest misconceptions.

AWS KMS **does not store Vault's Master Key**.

Instead:

- Vault stores an **encrypted copy of the Master Key**.
- AWS KMS stores a **Seal Key**.
- AWS KMS uses the Seal Key to decrypt the encrypted Master Key during startup.

---

# Seal Key

The Seal Key is managed by the external KMS service.

Example:

```text
AWS KMS

↓

Seal Key
```

Vault never downloads the Seal Key itself.

Instead, Vault asks AWS KMS to perform cryptographic operations.

---

# Auto Unseal Boot Flow

```text
Vault Starts

↓

Encrypted Master Key Found

↓

Vault Sends Encrypted Master Key to AWS KMS

↓

AWS KMS Verifies IAM Permissions

↓

AWS KMS Decrypts the Master Key

↓

Plaintext Master Key Returned to Vault

↓

Master Key Loaded into RAM

↓

Vault Unsealed
```

---

# What if someone steals the Storage Backend?

Suppose an attacker steals:

```text
Raft Database
```

The attacker gets:

- Encrypted Secrets
- Encrypted Master Key

Can they read the secrets?

**No.**

Because they still need AWS KMS to decrypt the encrypted Master Key.

Without permission to use the KMS key, they cannot obtain the plaintext Master Key.

---

# Manual Unseal vs Auto Unseal

## Manual Unseal

```text
Master Key

↓

Shamir's Secret Sharing

↓

Unseal Keys

↓

Admins Reconstruct Master Key
```

---

## Auto Unseal

```text
Master Key

↓

Encrypted

↓

AWS KMS (Seal Key)

↓

Vault Automatically Recovers Master Key
```

---

# Recovery Keys

Recovery Keys exist **only when using Auto Unseal**.

Since AWS KMS already handles unsealing, Vault no longer generates Unseal Keys.

Instead, Vault generates **Recovery Keys**.

---

# When are Recovery Keys generated?

During initialization.

## Manual Unseal

```bash
vault operator init
```

Output:

```text
5 Unseal Keys

1 Initial Root Token
```

---

## Auto Unseal

```bash
vault operator init
```

Output:

```text
5 Recovery Keys

1 Initial Root Token
```

Vault generates either **Unseal Keys** or **Recovery Keys**, never both.

---

# Purpose of Recovery Keys

Recovery Keys **do not** unseal Vault.

Instead, they are used for recovery operations, such as:

- Generating a new Root Token
- Other sensitive recovery operations

---

# Root Token Recovery

Suppose:

- Vault is running.
- AWS KMS successfully unsealed Vault.
- Every Root Token is accidentally revoked.

Administrators can recover by running:

```bash
vault operator generate-root
```

Vault asks for the Recovery Keys.

Example:

```text
Recovery Key 1

↓

Recovery Key 3

↓

Recovery Key 5
```

Once the threshold is reached:

Vault generates a **new Root Token**.

---

# Why can't Recovery Keys unseal Vault?

Because unsealing is already handled by AWS KMS.

Recovery Keys are only used to prove that enough trusted administrators approve sensitive recovery operations.

---

# Manual Unseal Root Recovery

Manual Unseal does not have Recovery Keys.

Instead, Vault reuses the **Unseal Keys**.

Example:

```bash
vault operator generate-root
```

Vault asks for the **Unseal Keys**.

After the threshold is reached:

A new Root Token is generated.

---

# Unseal Keys vs Recovery Keys

| Feature | Unseal Keys | Recovery Keys |
|----------|-------------|---------------|
| Available in Manual Unseal | ✅ Yes | ❌ No |
| Available in Auto Unseal | ❌ No | ✅ Yes |
| Unseal Vault | ✅ Yes | ❌ No |
| Generate New Root Token | ✅ Yes (Manual Unseal) | ✅ Yes (Auto Unseal) |
| Recover Master Key | ✅ Yes | ❌ No |

---

# Complete Comparison

## Manual Unseal

```text
Vault Starts

↓

Admins Enter Unseal Keys

↓

Master Key Reconstructed

↓

Vault Unsealed

↓

Unseal Keys can also generate a new Root Token
```

---

## Auto Unseal

```text
Vault Starts

↓

AWS KMS Decrypts Master Key

↓

Vault Unsealed

↓

Recovery Keys generate a new Root Token
```

---

# Key Takeaways

- Auto Unseal removes the need for administrators to manually unseal Vault.
- AWS KMS does not store Vault's Master Key.
- AWS KMS stores and manages the Seal Key.
- Vault stores an encrypted copy of the Master Key.
- During startup, AWS KMS decrypts the encrypted Master Key.
- Recovery Keys exist only with Auto Unseal.
- Recovery Keys cannot unseal Vault.
- Recovery Keys are used for recovery operations such as generating a new Root Token.
- In Manual Unseal, Unseal Keys perform both unsealing and Root Token recovery.

---

# Interview Questions

## Q1. Why is Auto Unseal used?

**Answer:**

Auto Unseal allows Vault to recover automatically after a restart without requiring administrators to manually enter Unseal Keys.

---

## Q2. Does AWS KMS store Vault's Master Key?

**Answer:**

No. AWS KMS stores and manages the Seal Key. Vault stores an encrypted copy of the Master Key.

---

## Q3. What is the purpose of the Seal Key?

**Answer:**

The Seal Key is used by AWS KMS to decrypt Vault's encrypted Master Key during startup.

---

## Q4. What are Recovery Keys?

**Answer:**

Recovery Keys are generated only in Auto Unseal mode. They are used for recovery operations such as generating a new Root Token and cannot unseal Vault.

---

## Q5. Can Recovery Keys unseal Vault?

**Answer:**

No. Vault is unsealed by AWS KMS (or another external seal provider). Recovery Keys cannot reconstruct the Master Key.

---

## Q6. What is used to generate a new Root Token?

**Answer:**

- Manual Unseal: Unseal Keys.
- Auto Unseal: Recovery Keys.

---

## Q7. What happens if AWS KMS is unavailable?

**Answer:**

Vault cannot automatically recover the Master Key, so it remains sealed. Recovery Keys cannot replace AWS KMS for unsealing.