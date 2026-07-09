# Interview Questions

### What is UserPass Authentication?

UserPass is an authentication method that allows users to log in to Vault using a username and password. After successful authentication, Vault issues a client token associated with one or more policies.

---

### Does Vault store the user's password?

Yes. Vault securely stores the password (as a password hash, not plain text) as part of the UserPass authentication backend.

---

### What does UserPass authentication return?

A client token that contains the policies and TTL assigned to the authenticated user.

---

### Is UserPass recommended for production?

Generally **no** for large enterprises. Organizations usually use LDAP, OIDC, or SAML for centralized identity management.

---

### Can one user have multiple policies?

Yes.

Example:

```bash
vault write auth/userpass/users/lokesh \
password="Password@123" \
policies="dev-policy,admin-policy"
```

---

### Where are UserPass users stored?

Inside the UserPass authentication backend in Vault's storage (integrated storage or another configured storage backend). Passwords are stored securely as hashes.

---

### What happens after login?

```text
Username
    │
Password
    │
    ▼
Vault verifies credentials
    │
    ▼
Client Token Created
    │
    ▼
Policies Attached
    │
    ▼
Access Granted
```

---