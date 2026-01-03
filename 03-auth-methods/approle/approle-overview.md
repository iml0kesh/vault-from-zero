# APPROLE

Approle Auth method is used by machines, apps, CI/CD - NOT Humans.

## Steps involved in approle process:

1. Enable approle auth method
2. Create a policy and a role
3. Get RoleID and SecretID of the role
4. Deliver RoleID and SecretID to the application (Like Jenkins)
5. Authenticate to vault using RoleID & SecretID.
