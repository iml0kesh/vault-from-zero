# UserPass Authentication Method

## Overview

The UserPass authentication method allows users to authenticate to HashiCorp Vault using a **username** and **password**.

After successful authentication, Vault generates a **client token** based on the policies attached to the user.

This method is commonly used for:
- Development environments
- Testing
- Small internal teams
- Human users

It is **not recommended for production** environments where enterprise identity providers (LDAP, OIDC, SAML) are available.

---

# How UserPass Authentication Works

```text
        User
          │
Username + Password
          │
          ▼
   UserPass Auth Method
          │
Credentials Verified
          │
          ▼
      Vault Token
          │
Attached Policies
          │
          ▼
Access Secrets
```

---

# Authentication Flow

1. User enters username and password.
2. Vault validates the credentials.
3. If authentication succeeds:
   - Vault generates a client token.
   - Policies attached to the user are applied.
4. The client token is used for all future Vault API requests.

---


# Token Generation

UserPass **does not directly grant access to secrets.**

Instead:

```text
Username + Password
        │
        ▼
Vault Authentication
        │
        ▼
Client Token
        │
        ▼
Policies
        │
        ▼
Secrets
```

The token determines what the user can access.

---

# Advantages

- Very easy to configure.
- Good for learning Vault.
- Ideal for development.
- Built into Vault.
- No external identity provider required.

---

# Limitations

- Passwords are managed inside Vault.
- Manual user administration.
- Not suitable for large organizations.
- No enterprise Single Sign-On (SSO).
- Difficult to manage many users.

---


# Common Commands

Enable

```bash
vault auth enable userpass
```

Create User

```bash
vault write auth/userpass/users/lokesh password="Password@123" policies="dev-policy"
```

Login

```bash
vault login -method=userpass username=lokesh password=Password@123
```

Read User

```bash
vault read auth/userpass/users/lokesh
```

List Users

```bash
vault list auth/userpass/users
```

Delete User

```bash
vault delete auth/userpass/users/lokesh
```

Disable

```bash
vault auth disable userpass
```

---



# Key Points

- UserPass is an authentication method.
- Uses username and password.
- Returns a Vault client token.
- Policies determine authorization.
- Suitable for development and testing.
- Not ideal for enterprise production environments.