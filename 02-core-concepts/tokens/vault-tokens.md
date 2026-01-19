# Vault Tokens
A Vault token is like an ID card + permission slip used to talk to vault.	If you don’t have a valid token, vault will say permission is denied.

**_Vault token = who are you + what you can do + how long you can do it_**

Tokens are core methods for authentication.
What a token contains: policies (permissions), TTL (expiry time), renewable, parent (who created it), Type (Root / Service / Batch / Periodic)

## Token Types:
### Root Token: 
created during vault operator init, has full access, can do anything.

### Service Tokens (hvs.):
used by applications, scripts, humans. Have policies, have TTL, Renewable. Default token type. 
Ex: hvs.CAESIBT9p6ReaWneZKtH5JC0DD7qaL1 KmiAKfmlE9MvlU826GiAKHGh2cy5DeDJZd m4yc0ZJMWJBZmZWVEg2VzM2RXEQLQ

### Batch Tokens (hvb.):
One-time use, not stored in vault, very fast, no renewal, used in high scale jobs
Ex: hvb.CAESIBT9p6ReaWneZKtH5JC0DD7qaL1 KmiAKfmlE9MvlU826GiAKHGh2cy5DeDJZd m4yc0ZJMWJBZmZWVEg2VzM2RXEQLQ 

### Recovery Tokens (hvr.): 
generated when the auto-unseal is initiated.
