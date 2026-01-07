# AUDIT LOGS

## What Are Audit Logs in Vault?

Audit logs record every request and response that flows through Vault.

This includes:

1. Authentication attempts

2. Secret reads/writes

3. Policy changes

4. Token usage

5. Permission denials

Once enabled, audit logging cannot be disabled accidentally without explicit action.

## Why Audit Logs Are Important?

Audit logs are critical for security and compliance.

They help with:

1. Security investigations

2. Compliance (SOC2, ISO, HIPAA, PCI-DSS)

3. Detecting unauthorized access

4. Debugging permission issues

Without audit logs → Vault is blind.

## What Is an Audit Device?

An Audit Device defines where and how Vault writes audit logs. Vault supports multiple audit devices at the same time.

| Device Type |        Description        |
| :---------- | :-----------------------: |
| File        |   Writes logs to a file   |
| stdout      |      Logs to console      |
| socket      |  Sends logs over TCP/UDP  |
| syslog      | Sends logs to system logs |
