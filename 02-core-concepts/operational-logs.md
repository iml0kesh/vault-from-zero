## What are Operational Logs?

Operational logs record how an application or system is running internally. They describe system behavior, health, and failures, not user intent.

## What operational logs contain?

| Question               | Example                    |
| :--------------------- | :------------------------- |
| is the service running | Vault started Successfully |
| is it healthy?         | High latency detected      |
| Did something break    | Database connection failed |
| why did it fail?       | Timeout, config error      |
| When did it fail       | Timestamp                  |

## Explanation:

Operational logs help us understand how a system behaves internally. They include startup logs, performance metrics, errors, and dependency failures. These logs are used by DevOps and SRE teams for monitoring, troubleshooting, and alerting. Tools like Splunk and New Relic centralize these logs, create dashboards, and trigger alerts which integrate with ServiceNow for incident management.
