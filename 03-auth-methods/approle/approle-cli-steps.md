## Enable Approle auth

```bash
vault auth method approle
```

let's create demo K/V secret for our role to access. \
Enable kv secrets engine

```bash
vault secrets enable --version=2 --path=kv2 kv
```

Demo Data

```bash
vault kv put kv2/mysql/webapp db_name="users" username="admin" password="passw0rd"
```

### create a policy

Logged as admin, When approle auth method is enabled, It gets mounted at the `/auth/approle` path. In this example, let's create a role for the app `(jenkins)`.

create a `.hcl` file using vs code or preferred text editor. \
`jenkins.hcl`

```bash
# Read secret data
path "kv2/data/mysql/webapp" {
  capabilities = ["read"]
}

# Allow metadata access (required for kv get)
path "kv2/metadata/mysql/*" {
  capabilities = ["read", "list"]
}

# Required for Vault CLI
path "sys/internal/ui/mounts/*" {
  capabilities = ["read"]
}
```

after creating the `jenkins.hcl` file let's load the policy file into the vault

```bash
vault policy write jenkins <path_of_jenkins.hcl>
```

now let's create a `jenkins` role with `jenkins` policy attached

```bash
vault write auth/approle/role/jenkins token_policies="jenkins"
```

now Let's verify weatheer jenkins roles is created and policy attached or not:

```bash
vault read auth/approle/jenkins
```

Now we need to get the RoleID and SecretID

To Retrieve RoleID

```bash
vault read auth/approle/role/jenkins/role-id
```

To Generate SecretID

```bash
vault write -force auth/approle/role/jenkins/secret-id
```

Now logout as root user and login as the client (Jenkins). These role-id and secret-id are shared with Jenkins to login automatically.

```bash
vault write auth/approle/login role_id="<role_id>" secret_id="<secret_id>"
```

After logging in the token will be generated with the attached policies and time limitation's to access the vault secrets.

in our case let's check for the read access of the webapp

```bash
vault kv get secret/mysql/webapp
```
