This documentation summarizes the process of replicating a production PostgreSQL database to a staging environment located on a separate server.

---

## 🛠 Database Replication Guide: Prod to Staging

This guide covers the "Piped Snapshot" method, which transfers data directly between servers without creating large intermediate files on disk.

### 1. Prerequisites

Before running the replication command, ensure the following connectivity requirements are met:

* **Network:** The Staging server must be able to reach the Production server on port **5432**.

* **Postgres Configuration (Prod):**
* *Edit postgresql.conf:*  `/etc/postgresql/$(ls /etc/postgresql/)/main/postgresql.conf` and set: `listen_addresses = '[STAGING_IP]'` or  `listen_addresses = '*'`
* *Edit pg_hba.conf:* `/etc/postgresql/$(ls /etc/postgresql/)/main/pg_hba.conf` and add this in the last line `host all all [STAGING_IP]/32 md5` or `host all all 0.0.0.0/32 scram-sha-256`
* *Apply changes:* `sudo systemctl restart postgresql`
* *Firewall allow connection:* `sudo ufw allow from [STAGING_IP] to any port 5432 proto tcp`

* **Database Users:**
* *Source:* A user with `SELECT` privileges on the production DB (e.g., `prod_app_user`).
* *Destination:* A user with `CREATE/OWNER` privileges on the staging DB (e.g., `staging_admin`).

* **Staging Sever**
* *Create pg_user:* 
```sql
CREATE ROLE staging_admin WITH LOGIN PASSWORD 'admin@staging' SUPERUSER;
```
* *Create pg_database:* 
```sql
CREATE DATABASE staging_db OWNER staging_admin;
```

---

### 2. The Replication Command

Run this command from the **Staging Server**. It dumps the production data and pipes it directly into the local staging database.

```bash
PGPASSWORD='[PROD_PASSWORD]' pg_dump -h [PROD_IP] -U [PROD_USER] --no-owner --no-privileges [PROD_DB_NAME] | PGPASSWORD='[STAGING_PASSWORD]' psql -h localhost -U [STAGING_USER] [STAGING_DB_NAME]

```

#### Key Flags Explained:

| Flag | Purpose |
| --- | --- |
| `--no-owner` | Prevents errors if the Production user doesn't exist on Staging. The Staging user will become the new owner. |
| `--no-privileges` (or `-x`) | Skips exporting specific GRANT/REVOKE permissions to avoid "role does not exist" errors. |
| `PGPASSWORD` | Sets the password for the immediate command without a manual prompt. |

---

### 3. Post-Migration Tasks

After the data is transferred, you may need to grant permissions to your staging application user if it differs from the `staging_admin`.

**Run these SQL commands on the Staging DB:**

```sql
-- Grant access to all existing tables
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO [STAGING_APP_USER];

-- Grant access to all sequences (required for ID autoincrement)
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO [STAGING_APP_USER];

```

---

### 4. Troubleshooting Common Errors

#### "Role [name] does not exist"

* **Cause:** The dump is trying to assign ownership or permissions to a user that only exists on Production.
* **Fix:** Ensure `--no-owner` and `--no-privileges` are included in your `pg_dump` command.

#### "Connection timed out" or "No route to host"

* **Cause:** Firewall is blocking port 5432 or `listen_addresses` in `postgresql.conf` is set to `localhost` only on the Prod server.
* **Fix:** Update `postgresql.conf` to `listen_addresses = '*'` and check your cloud provider's Security Groups/Firewall.

---

### 🔒 Security Best Practice

If your production database contains sensitive user data (emails, passwords, phone numbers), it is highly recommended to run an **anonymization script** immediately after the import to replace real data with "dummy" values.

### BASH Script to refresh data

Once the setup is done, you can save this as sync_db.sh on your staging server to refresh the data whenever you want:

```BASH
#!/bin/bash
echo "Cleaning old staging data..."
dropdb -h localhost -U staging_admin staging_db
createdb -h localhost -U staging_admin staging_db

echo "Syncing from Production..."
PGPASSWORD='[PROD_PASSWORD]' pg_dump -h [PROD_IP] -U [PROD_USER] --no-owner --no-privileges [PROD_DB_NAME] | PGPASSWORD='[STAGING_PASSWORD]' psql -h localhost -U [STAGING_USER] [STAGING_DB_NAME]
echo "Sync Complete."
```

### POSTGRES COMMANDS

### Database & User Management (Inside `psql`)

These are the commands you run after typing `sudo -u postgres psql` to manage your environment.

| Command | Description |
| --- | --- |
| `\du` | List all users (roles) and their permissions. |
| `\l` | List all databases on the server. |
| `\dn` | List all schemas (typically just `public` for a fresh install). |
| `\dt` | List all tables in the currently connected database. |
| `\c [dbname]` | Connect to a specific database (e.g., `\c staging_db`). |
| `\conninfo` | Shows details about your current connection (IP, port, user). |
| `SELECT usename FROM pg_shadow;` | List all users with passwords (Only super user can view this). |
| `SELECT usename, usesuper, usecreatedb, valuntil FROM pg_user;` | List users with attributes |
| `SELECT rolname FROM pg_roles WHERE rolname = 'staging_admin';` | Find a user |
---

### Table & Data Inspection (Inside `psql`)

Once you have migrated your data, you will need to verify that everything arrived correctly.

| Command | Description |
| --- | --- |
| `\dt+` | List tables *with* detailed information (size, description). |
| `\d [tablename]` | Show the structure of a specific table (columns, data types, indexes). |
| `\x` | Toggle "expanded display" mode (great for reading wide tables). |
| `SELECT count(*) FROM [tablename];` | Verify data row count between Prod and Staging. |

---


### 💡 Pro-Tips for Navigation

* **Exiting:** If you ever get stuck in a long list or a prompt, type `\q` to exit the `psql` shell and return to the terminal.
* **SQL Queries:** Remember, anything that doesn't start with a `\` is treated as SQL. You can run standard queries like `SELECT * FROM users LIMIT 5;` at any time to verify actual data content.

