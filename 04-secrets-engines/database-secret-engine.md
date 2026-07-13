Database Secrets Engine
The Database secrets engine dynamically generates database credentials on-demand.
Instead of storing static usernames/passwords, Vault can:
· Create a new database user automatically when requested
· Set a time-to-live (TTL) for that user
· Revoke the credentials automatically when they expire
Use case:
· Your application connects to a database without hard-coding credentials
· Reduces risk of leaks
· Ensures every app or micro-service gets unique, temporary credentials
How it works

1. Vault connects to your database using a root/admin account.
2. You configure a role in Vault that defines:
   o Which SQL commands to run to create users
   o Default TTL for the credentials
3. When an app requests credentials from Vault:
   o Vault creates a new user in the database
   o Returns username + password to the app
   o Credentials automatically expire after TTL
   When you enable the database secrets engine in Vault, it always has these logical paths:
   · database/config/ → Where Vault keeps connection info for databases (host, plugin, admin username/password, etc.)
   · database/roles/ → Where Vault keeps “blueprints” for generating dynamic database credentials

Enable database secrets engine: $vault secrets enable database
Database config file for mysql: $vault write database/config/my-mysql-db plugin_name=mysql-database-plugin connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" username="root" password="1937"
