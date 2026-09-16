# Backup script for Zabbix configuration data (MySQL/PostgreSQL)

[![CI](https://github.com/npotorino/zabbix-backup/actions/workflows/main.yml/badge.svg)](https://github.com/npotorino/zabbix-backup/actions/workflows/main.yml)

`zabbix-dump` is a single Bash script that backs up a [Zabbix](http://www.zabbix.com/) MySQL or
PostgreSQL database, covering every Zabbix version from 1.3.1 up to 7.4.

Instead of dumping the whole database (which for a busy Zabbix instance can mean gigabytes of
history/trends data), it dumps a **configuration-only** backup: small "configuration" tables
(hosts, items, triggers, actions, users, ...) are backed up with their data, while large
operational/runtime tables are backed up schema-only, keeping the backup small and fast.

## How it works

Every table Zabbix has ever shipped is listed at the end of the script, together with the Zabbix
version range it existed in and a `SCHEMAONLY` flag for tables that should be excluded from the
data dump:

```
service_alarms             1.3.1    - 7.4.14
service_problem            6.0.0    - 7.4.14    SCHEMAONLY
service_problem_tag        6.0.0    - 7.4.14    SCHEMAONLY
```

At runtime the script reads the current table list from the database itself, so it only acts on
tables that actually exist in your Zabbix version — a `SCHEMAONLY` table that doesn't exist yet
(e.g. on an older Zabbix) is simply ignored.

Tables flagged `SCHEMAONLY` (history, history_uint, trends, trends_uint, events, problem,
service_problem, auditlog, ...) get their **schema backed up but no rows**. Everything else is
backed up in full, including data.

### Handling unknown tables (new Zabbix versions)

If your Zabbix database contains a table the script doesn't know about yet (for example right
after a Zabbix upgrade that added new tables before this script's table list was updated), it
will by default **stop with an error** rather than silently guess whether the table should be
backed up. You can change this behavior:

- `-f` — treat unknown tables as configuration data and back them up in full (safe default if
  you're unsure, at the cost of a possibly larger backup).
- `-i` — ignore unknown tables entirely (exclude them from the backup).

Either way, please [open an issue](https://github.com/npotorino/zabbix-backup/issues) so the
table list can be updated for everyone.

## Requirements

- `bash`
- For MySQL: the `mysql` and `mysqldump` client binaries
- For PostgreSQL: the `psql` and `pg_dump` client binaries
- `dig` (optional, used for reverse DNS lookup of the database host; skip with `-n`)

## Download

Download the latest (stable) release here:

https://github.com/npotorino/zabbix-backup/releases/latest

or clone the repository directly:

```bash
git clone https://github.com/npotorino/zabbix-backup
cd zabbix-backup
```

## More information

Please see the [Project Wiki](https://github.com/npotorino/zabbix-backup/wiki).

## Usage

```
zabbix-dump [options]
```

| Option | Description | Default |
|---|---|---|
| `-t DATABASE_TYPE` | Database type: `mysql` or `psql` | `mysql` |
| `-H DBHOST` | Hostname/IP of the database server | `127.0.0.1` |
| `-P DBPORT` | DBMS port | `3306` (mysql) / `5432` (psql) |
| `-s DBSOCKET` | Path to a DBMS socket file, as an alternative to `-H`/`-P` | — |
| `-S SCHEMA` | Database schema (PostgreSQL only) | `public` |
| `-d DATABASE` | Name of the Zabbix database | `zabbix` |
| `-u DBUSER` | DBMS user | `zabbix` |
| `-p DBPASSWORD` | DBMS user password (use `-` to be prompted) | no password |
| `-o DIR` | Directory to save the dump to (`-` streams the dump to stdout) | `$PWD` |
| `-z ZABBIX_CONFIG` | Read DB host/credentials from a Zabbix server config file | `/etc/zabbix/zabbix_server.conf` |
| `-Z` | Do not try to read the Zabbix server configuration | — |
| `-c MYSQL_CONFIG` | MySQL only: read DB host/credentials from a MySQL config file | — |
| `-r NUM` | Rotate backups, keeping up to `NUM` generations (matched by filename) | keep all |
| `-x` | Compress using XZ instead of GZip (smaller, slower) | — |
| `-0` | Do not compress the dump | — |
| `-n` | Skip reverse DNS lookup of the database host | — |
| `-N` | Add column names to `INSERT INTO .. VALUES ..`, quoting as needed | — |
| `-C` | Use PostgreSQL's custom dump format (required for `pg_restore`) | plain SQL |
| `-T` | Keep triggers' PROBLEM state as-is instead of resetting it (see restore notes below) | reset |
| `-B` | Keep TimescaleDB's `ts_insert_blocker` triggers in the dump (PostgreSQL only, see restore notes below) | stripped |
| `-f` | Force backup of unknown tables (full data, forward compatibility) | — |
| `-i` | Ignore unknown tables (exclude them from the backup) | — |
| `-q` | Quiet mode: no output except errors (for cron/batch use) | — |
| `-h`, `--help` | Show help | — |
| `--version` | Show the script version | — |

Run `./zabbix-dump --help` for the full built-in help text.

## Examples

### Backup

Backup a local MySQL Zabbix database, reading connection details from
`/etc/zabbix/zabbix_server.conf`:

```bash
./zabbix-dump
```

Backup Zabbix with PostgreSQL and TimescaleDB:

```bash
./zabbix-dump -t psql -H localhost -P 5432 -o /var/backup
```

Backup without reading `zabbix_server.conf`, asking for the password interactively, keeping the
last 7 generations:

```bash
./zabbix-dump -Z -d zabbixdb -u zabbix -p - -o /var/backup -r 7
```

### Restore

> **Note on foreign keys:** the tables the script dumps schema-only (`history*`, `trends*`,
> `events`, `problem`, `service_problem`, ...) are intentionally left empty. The dump is built so
> that no table with data references one of those emptied tables, so a restore stays
> referentially consistent. The safest way to load it back is against an already-created
> (empty) target schema using `--disable-triggers --data-only`, as shown below — this avoids
> re-validating foreign keys that a plain, single-pass SQL restore may check as part of applying
> post-data constraints.

> **Note on trigger state:** `triggers.value` (0 = OK, 1 = PROBLEM) is configuration data, so it
> *is* restored — but since `problem`/`events` data isn't, a trigger that was in the PROBLEM state
> at backup time comes back with no matching row in `problem` to justify it, and `zabbix-server`
> treats that value as "no change" on startup, so it may never re-fire a notification for a
> trigger that's genuinely still down. To avoid this, `zabbix-dump` **automatically appends**
> ```sql
> UPDATE triggers SET value = 0, lastchange = 0, error = '' WHERE value = 1;
> ```
> to the dump, so it takes effect as part of a normal restore (`mysql < dump.sql` /
> `psql < dump.sql`) with no extra step required. Two caveats:
> - This only works when the dump is restored as plain SQL. A PostgreSQL custom-format dump
>   (`-C`) is a binary archive restored with `pg_restore`, which can't have SQL appended to it —
>   in that case the statement is written to a `<dumpfile>.post-restore.sql` companion file
>   instead (or printed to stderr when dumping to stdout with `-o -`); run it manually against
>   the restored database.
> - The reset assumes you're restoring into a freshly emptied database. If instead you're
>   restoring the configuration into a database that keeps its own live monitoring data (e.g.
>   syncing prod config onto a test server without touching the test server's history), pass
>   `-T` to skip the automatic reset, and run this more selective version yourself so you don't
>   clear triggers that still have a genuinely active problem there:
>   ```sql
>   UPDATE triggers SET value = 0, lastchange = 0, error = ''
>    WHERE value = 1
>      AND triggerid NOT IN (SELECT objectid FROM problem WHERE r_eventid IS NULL);
>   ```

#### MySQL

```bash
# systemctl stop zabbix-server
gunzip zabbix_cfg_localhost_20200730-1810_db-mysql-5.0.1.sql.gz
mysql -u zabbix -p zabbix < zabbix_cfg_localhost_20200730-1810_db-mysql-5.0.1.sql
# systemctl start zabbix-server
```

(If the database doesn't exist yet, create it first — see the
[Zabbix installation docs](https://www.zabbix.com/documentation/current/en/manual/installation/install)
for the recommended character set/collation for your Zabbix version.)

#### PostgreSQL

```bash
# systemctl stop zabbix-server.service
gunzip /var/backup/zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql.gz
sudo -u postgres psql zabbix < /var/backup/zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql
# systemctl start zabbix-server.service
```

(If the database doesn't exist yet, create it first: `sudo -u postgres createdb -O zabbix zabbix`.)

#### PostgreSQL + TimescaleDB

`zabbix-dump` writes a **configuration-only** backup, so history/trends stay schema-only and the
dump carries no usable TimescaleDB state. The hypertables have to be rebuilt from the
`timescaledb.sql` script shipped with your Zabbix version, *after* the configuration has been
restored.

Restoring the dump as-is and starting the server fails with:

```
[Z3005] query failed: ... ERROR: table "history" is not a hypertable
[select set_integer_now_func('history', 'zbx_ts_unix_now', true)]
```

because the `ts_insert_blocker` triggers contained in the dump get applied to tables that aren't
(yet) hypertables (see [#11](https://github.com/npotorino/zabbix-backup/issues/11)). A plain-format
PostgreSQL dump strips those triggers **automatically by default** (pass `-B` at backup time to
keep them instead — see Usage above); then just let Zabbix rebuild the hypertables on the (now
empty) history tables:

```bash
systemctl stop zabbix-server

# 1. empty database (assumes the 'zabbix' role exists, otherwise:
#    sudo -u postgres createuser --pwprompt zabbix)
sudo -u postgres dropdb zabbix
sudo -u postgres createdb -O zabbix zabbix

# 2. restore the configuration
zcat zabbix_cfg_<host>_<date>_db-psql-<version>.sql.gz | sudo -u postgres psql zabbix

# 3. install the extension and rebuild the hypertables with Zabbix's own script
#    (history/trends are empty, so the conversion is clean)
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
sudo -u postgres psql zabbix < /usr/share/zabbix-sql-scripts/postgresql/timescaledb.sql

systemctl start zabbix-server
```

Use the `timescaledb.sql` from the **same** Zabbix version as the backup; its path depends on
your packages (e.g. `/usr/share/zabbix-sql-scripts/postgresql/`).

**Notes**

- Restoring a dump made with `-B`, or one from before this fix, still hits the error above —
  strip the triggers manually before restoring in that case:
  ```bash
  zcat zabbix_cfg_<host>_<date>_db-psql-<version>.sql.gz \
    | grep -v 'CREATE TRIGGER ts_insert_blocker' \
    | sudo -u postgres psql zabbix
  ```
- If the restore also fails on `_timescaledb_catalog` objects, create the backup with `-S public`
  so the internal TimescaleDB schemas are never dumped.
- `pg_dump` 15.15 / 16.x / 17.x+ (the Aug 2025 security fix) writes `\restrict` / `\unrestrict`
  meta-commands into the dump. An older restoring `psql` reports `invalid command \restrict`;
  either upgrade the client or drop those lines, e.g. add `sed -E '/^\\(un)?restrict /d'` to the
  pipe in step 2.
- A dump made with a current `zabbix-dump` no longer needs any extra handling for the
  `c_service_problem_1` foreign key here — `service_problem`/`service_problem_tag` are dumped
  schema-only (see the note on foreign keys above). Only dumps made before that fix need it.

#### PostgreSQL, custom dump format (`-C`) with `pg_restore`

A different approach using the original Zabbix schema and `pg_restore` with the custom dump
format (`-C`). Zabbix version: 5.0.

```bash
# create the backup using the custom format
./zabbix-dump -t psql -C -H localhost -P 5432 -o /var/backup

# ...later, restore it:
# systemctl stop zabbix-server.service
su - postgres
dropdb zabbix
# we assume the zabbix user already exists, if it doesn't: createuser --pwprompt zabbix
createdb -O zabbix zabbix
cat /usr/share/zabbix-postgresql/schema.sql | psql -h 127.0.0.1 -U zabbix -d zabbix
gunzip /var/backup/zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql.gz
pg_restore --disable-triggers --data-only -d zabbix /var/backup/zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql
# systemctl start zabbix-server.service
```

If the underlying database uses TimescaleDB, install the extension and rebuild the hypertables as
in the "PostgreSQL + TimescaleDB" steps above — the same `ts_insert_blocker` pitfall applies here
too, but since a custom-format archive is binary, the triggers can't be stripped from the dump
file directly. Instead, `zabbix-dump -C` writes a companion `<dumpfile>.toc` file next to it
(unless `-B` was passed) — restore with that TOC instead of a plain `pg_restore` invocation to
skip them:

```bash
pg_restore --use-list=zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql.toc \
  --disable-triggers --data-only -d zabbix zabbix_cfg_localhost_20200730-1810_db-psql-5.0.1.sql
```

## Development

- The full table list is generated with the included [`get-table-list.pl`](get-table-list.pl)
  helper, which walks tagged Zabbix releases to find every table's version range. New releases
  still need a human pass to flag any new/changed table as `SCHEMAONLY` where appropriate.
- See [`TESTING.md`](TESTING.md) for instructions to spin up disposable MySQL/PostgreSQL Zabbix
  containers for manual backup/restore testing.
- Pull requests are checked by [ShellCheck](https://www.shellcheck.net/) via GitHub Actions
  (see [`.github/workflows/main.yml`](.github/workflows/main.yml)).

## License

Released under the [MIT License](LICENSE.txt).

## Version history

**0.9.14 (2026-09-16)**
- ENH: Support for Zabbix 7.4
- FIX: exclude `service_problem`/`service_problem_tag` data (orphaned FK to schema-only `problem`/`events`)
- FIX: pin dbversion query to a single row, fixing corrupted dump filenames (#16)
- FIX: only add DBPORT for mysql when no socket is in use (#30)
- ENH: reset stale trigger PROBLEM state in the dump by default, with `-T` to opt out (#34)
- ENH: strip TimescaleDB's `ts_insert_blocker` triggers from the dump by default, with `-B` to opt out (#11)

**0.9.13 (2025-03-17)**
- ENH: Support for Zabbix 7.2

**0.9.12 (2024-09-02)**
- ENH: Support for Zabbix 7.0

**0.9.11 (2023-05-02)**
- ENH: Support for Zabbix 6.4
- FIX: force decimal interpretation in db versioning

**0.9.10 (2022-09-30)**
- FIX: use the default schema for PostgreSQL, when command line parameter is not set

**0.9.9 (2022-08-31)**
- ENH: zabbix dump add custom format (pg), stdout dump (@gullevek)
- ENH: minimal automated syntax check through Github Actions
- FIX: table schema for PostgreSQL (@jokay)
- FIX: retrieving DBPassword when it contains a "=" (@mathieumd)

**0.9.8 (2022-07-27)**
- ENH: Support for Zabbix 6.2

**0.9.7 (2022-05-10)**
- ENH: Support for Zabbix 6.0

**0.9.6 (2021-12-13)**
- FIX: fix item_rtdata retention (@kernbug)

**0.9.5 (2021-07-23)**
- ENH: Support for Zabbix 5.2 and 5.4
- ENH: Add column names to INSERT INTO .. VALUES and quote names as needed (@diffway)

**0.9.4 (2021-01-21)**
- ENH: Support for Zabbix 5.0
- FIX: Be more specific on detecting mysql socket

**0.9.3 (2020-01-17)**

- ENH: Check for unknown tables
- ENH: Speed up MySQL backup by not calling mysqldump for every single table anymore
- ENH: New option -S to specify PostgreSQL schema
- FIX: Stabilize and enhance PostgreSQL dump
- FIX: Skip IP reverse lookup for localhost, fix multiline dig answers

**0.9.2 (2020-01-16)**

- ENH: Support for Zabbix 4.4
- ENH: Fix (non-critical) shellcheck issues (Mario Trangoni)
- CHG: Fix and enhance helper script get-table-list.pl
- FIX: Escape special characters while reading password (ironbishop)
- FIX: Re-enable accidentally disabled cleanup of postgresql password file
- FIX: Insert hostname into backup file also if database resides on localhost

**0.9.1 (2019-03-21)**

- FIX: Correctly process hostname option -H (Tiago Cruz)

**0.9.0 (2019-03-14)**

- NEW: Support for PostgreSQL databases (Sergey Galkin)
- NEW: Option -P to specify database server port (Sergey Galkin)
- NEW: Support for socket connections to MySQL and PostgreSQL server (Greg Cockburn)
- NEW: Database connection parameters are read from Zabbix servern config by default (Greg Cockburn)
- ENH: Support for Zabbix 4.0 (Wesley Schaft)
- ENH: Options -h and --help to show help (hostname is now specified using -H)
- ENH: Options -Z to skip reading the Zabbix server config file
- CHG: Rename script to "zabbix-dump"
- FIX: Support for whitespaces in database parameters (Greg Cockburn)
- FIX: Support for backslashes in manually entered password (Greg Cockburn)

**0.8.2 (2016-09-08)**

- NEW: Option -x to use XZ instead of GZ for compression (Jonathan Wright)
- NEW: Option -0 for "no compression"
- FIX: Evil space was masking end of here-document (fixed in #8 by @msjmeyer)
- FIX: Prevent "Warning: Using a password on the command line interface can be insecure."

**0.8.1 (2016-07-11)**

- ENH: Added Zabbix 3.0.x tables to list (added & tested by Ruslan Ohitin)

**0.8.0 (2016-01-22)**

- FIX: Only invoke `dig` if available
- ENH: Option -c to use a MySQL config ("options") file (suggested by Daniel Schneller)
- ENH: Option -r to rotate backup files (Daniel Schneller)
- ENH: Add database version to filename if available
- ENH: Add quiet mode. IP reverse lookup optional (Daniel Schneller)
- ENH: Bash related fixes (Misu Moldovan)
- CHG: Default output directory is now $PWD instead of script dir

**0.7.1 (2015-01-27)**

- NEW: Parsing of commandline arguments implemented
- ENH: Try reverse lookup of IPs and include hostname/IP in filename
- REV: Stop if database password is wrong

**0.7.0 (2014-10-02)**

- ENH: Complete overhaul to make script work with lots of Zabbix versions

**0.6.0 (2014-09-15)**

- REV: Updated the table list for use with zabbix v2.2.3

**0.5.0 (2013-05-13)**

- NEW: Added table list comparison between database and script

**0.4.0 (2012-03-02)**

- REV: Incorporated mysqldump options (suggested by Jonathan Bayer)

**0.3.0 (2012-02-06)**

- ENH: Backup of Zabbix 1.9.x / 2.0.0, removed unnecessary use of
  variables (DATEBIN etc) for commands that use to be in $PATH

**0.2.0 (2011-11-05)**
