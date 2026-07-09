# Enable UserPass Authentication

```bash
vault auth enable userpass
```

Verify:

```bash
vault auth list
```

Output:

```text
Path         Type
----         ----
userpass/    userpass
token/       token
```

---

# Create a Policy

Example policy:

```hcl
path "secret/data/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

Save as:

```text
dev-policy.hcl
```

Upload policy:

```bash
vault policy write dev-policy dev-policy.hcl
```

Verify:

```bash
vault policy list
```

---

# Create a User

Syntax

```bash
vault write auth/userpass/users/<username> \
password="<password>" \
policies="<policy>"
```

Example

```bash
vault write auth/userpass/users/lokesh \
password="Password@123" \
policies="dev-policy"
```

---

# Read User Details

```bash
vault read auth/userpass/users/lokesh
```

Example Output

```text
policies = ["dev-policy"]
token_ttl = 768h
token_max_ttl = 768h
```

---

# List Users

```bash
vault list auth/userpass/users
```

Example Output

```text
Keys
----
lokesh
john
admin
```

---

# Login using CLI

```bash
vault login -method=userpass \
username=lokesh \
password=Password@123
```

Example Output

```text
Success! You are now authenticated.

token
token_accessor
token_duration
token_policies
identity_policies
```

---

# Login using API

```bash
curl \
--request POST \
--data '{"password":"Password@123"}' \
http://127.0.0.1:8200/v1/auth/userpass/login/lokesh
```

---

# Login using Vault UI

1. Open Vault UI.
2. Select **UserPass**.
3. Enter username.
4. Enter password.
5. Click **Sign In**.
6. Vault generates a client token after successful authentication.

---

# Change Password

```bash
vault write auth/userpass/users/lokesh \
password="NewPassword@123"
```

---

# Delete User

```bash
vault delete auth/userpass/users/lokesh
```

---

# Disable UserPass Authentication

```bash
vault auth disable userpass
```

---