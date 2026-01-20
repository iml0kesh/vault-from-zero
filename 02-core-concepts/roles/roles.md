ROLES
A Role is a template that tells vault what kind of token to create when someone authenticates. They define behavior.
Auth method + Role = Token (With policies & rules)
Ex: Human Login (OIDC / LDAP)
User: Ravi
Team: dev-team
Auth method: OIDC (Google / Azure AD)

Step 0: role set-up
Role created by admin: auth/oidc/role/dev-role
Role rules:
	IF:
		group = “dev-team”
	THEN:
		policies = [“dev-policy”]
token_ttl = 8h 

Step 1: User tries to login
Ravi runs vault login –method=oidc
Now vault redirects you to OIDC login page to log into your account after successfully logging into your account the OIDC will return a JSON file containing the user details like email, groups, user_id etc.
Step 2: vault receives the JSON file
{
  "email": "ravi@company.com",
  "groups": ["dev-team", "frontend"],
  "user_id": "aad-98324"
}

Step 3: Vault matches ROLE
Vault iterates over the configured roles for that auth method and evaluates each role’s constraints against the received identity data.
Step 4: Vault issues token
