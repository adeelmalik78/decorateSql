# Liquibase `decorate-sql` demo

This repo demonstrates the `decorate-sql` command, new in **Liquibase Secure 6.0.2**.

`decorate-sql` turns raw SQL scripts into Liquibase changelogs **without connecting to a database**. It keeps your SQL as written and wraps each statement (or group of statements) in a changeset with an ID and author. You can then review the result, run policy checks, and deploy it with `liquibase update`.

Use it when you already have plain SQL scripts, such as Entity Framework migration scripts or hand-written DDL, and want Liquibase to track and deploy them.

📖 Documentation: [decorate-sql reference (6.0.2)](https://docs.liquibase.com/secure/reference-guide-6-0-2/decorate-sql)

## Requirements

- Liquibase Secure 6.0.2 or later, with a license key (`--license-key`, `liquibase.licenseKey`, or `LIQUIBASE_LICENSE_KEY`)
- A supported dialect: **Microsoft SQL Server (`mssql`)** or **Oracle (`oracle`)**
- Optional: `sqlcmd` (SQL Server) or `sqlplus` (Oracle) if you want changesets to run through a native executor. This repo sets the `sqlcmd` path in [liquibase.sqlcmd.conf](liquibase.sqlcmd.conf).

## Repo layout

```
.
├── sql/
│   ├── raw/                     # Raw SQL scripts (input)
│   │   ├── example1.sql         #   EF Core-style migration script with GO batches and transactions
│   │   └── example2.sql         #   Plain DDL and DML
│   └── decorated/               # Generated Liquibase formatted SQL changelogs (output)
│       ├── example1.mssql.sql
│       └── example2.mssql.sql
├── changelog.xml                # Root changelog that includes the decorated files
├── decorate-sql_commands.txt    # The commands used in this demo
├── liquibase.properties         # Changelog and connection settings used by update
├── liquibase.sqlcmd.conf        # Path to the sqlcmd executable
└── liquibase.flowfile.yaml      # Flow file that runs status, update, and history
```

## Example 1: decorate a single SQL file

```bash
liquibase decorate-sql \
    --sql-file=sql/raw/example1.sql \
    --changelog-file=sql/decorated/example1.mssql.sql \
    --db-type=mssql \
    --executor=sqlcmd \
    --author=amalik \
    --overwrite-output-file=true
```

The output format is set by the extension of `--changelog-file` (`.sql`, `.xml`, `.yaml`, `.yml`, or `.json`). Here it is a formatted SQL changelog.

**Before** (`sql/raw/example1.sql`):

```sql
IF OBJECT_ID(N'[__EFMigrationsHistory]') IS NULL
BEGIN
    CREATE TABLE [__EFMigrationsHistory] (...);
END;
GO

BEGIN TRANSACTION;
CREATE TABLE [Categories] (...);

CREATE TABLE [Customers] (...);
```

**After** (`sql/decorated/example1.mssql.sql`):

```sql
-- liquibase formatted sql

-- changeset amalik:1 splitStatements:false runWith:sqlcmd
IF OBJECT_ID(N'[__EFMigrationsHistory]') IS NULL
BEGIN
    CREATE TABLE [__EFMigrationsHistory] (...);
END;

-- changeset amalik:2 splitStatements:false runWith:sqlcmd
CREATE TABLE [Categories] (...);

-- changeset amalik:3 splitStatements:false runWith:sqlcmd
CREATE TABLE [Customers] (...);
```

Things to notice:

- Each DDL statement gets its own changeset, numbered in file order.
- `GO` lines are removed, because each batch is already its own changeset.
- Transaction control outside blocks, such as `BEGIN TRANSACTION`, is removed. Liquibase manages transactions per changeset.
- `--executor=sqlcmd` adds `runWith:sqlcmd` to every changeset. Leave it out to run through JDBC.
- `--author` is stamped on every changeset. The default is `raw-sql`.

`sql/raw/example2.sql` shows another case: consecutive `INSERT` statements are grouped into a single changeset (`amalik:4` in [sql/decorated/example2.mssql.sql](sql/decorated/example2.mssql.sql)).

## Example 2: decorate a folder of SQL scripts

```bash
liquibase decorate-sql \
    --sql-folder=sql/raw \
    --changelog-file=sql/decorated.xml \
    --db-type=mssql \
    --executor=sqlcmd \
    --author=amalik \
    --overwrite-output-file=true
```

In folder (batch) mode:

- Every `.sql` file in the folder is decorated in file-name order, and each gets its own changelog.
- `--changelog-file` becomes the **root changelog** that includes them, so it must be `.xml`, `.yaml`, `.yml`, or `.json`, not `.sql`.
- Add `--recursive=true` to include subfolders.
- `--sql-folder` takes a comma-separated list of folders.

## Deploy the decorated changelog

[changelog.xml](changelog.xml) includes the decorated files:

```xml
<include file="sql/decorated/example1.mssql.sql" relativeToChangelogFile="true"/>
<include file="sql/decorated/example2.mssql.sql" relativeToChangelogFile="true"/>
```

Set your connection details in `liquibase.properties`, then deploy:

```bash
liquibase update
```

Or run the flow file, which runs `status`, `update`, and `history`:

```bash
liquibase flow
```

`update` records each changeset in `DATABASECHANGELOG`, so later runs only deploy changesets that have not run yet.

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `--sql-file` | Yes, unless `--sql-folder` is set | | Raw SQL file to decorate. Takes a comma-separated list. |
| `--sql-folder` | Yes, unless `--sql-file` is set | | Comma-separated folders of `.sql` files (batch mode). |
| `--recursive` | No | `false` | Also decorate `.sql` files in subfolders. |
| `--changelog-file` | Yes | | Output changelog. The extension sets the format. |
| `--db-type` | Yes, unless `--url` is set | | `mssql` or `oracle`. |
| `--url` | No | | JDBC URL, used only to infer the dialect. Liquibase does not connect. |
| `--executor` | No | JDBC | `sqlcmd` or `sqlplus`. Sets `runWith` on each changeset. |
| `--author` | No | `raw-sql` | Author for every changeset. |
| `--overwrite-output-file` | No | `false` | Overwrite an existing output file. |
| `--rollback-file` | No | | Rollback SQL to pair with a single `--sql-file`. |
| `--allow-in-block-txn-control` | No | `false` | Allow `COMMIT` or bare `ROLLBACK` inside Oracle PL/SQL blocks. |
| `--allow-txn-control-loss` | No | `false` | Remove savepoints and `SET TRANSACTION` statements instead of failing. |
| `--allow-dialect-mismatch` | No | `false` | Warn instead of failing when the SQL contains syntax from the other dialect. |

Liquibase also accepts camelCase names, such as `--dbType` and `--overwriteOutputFile`. See the [documentation](https://docs.liquibase.com/secure/reference-guide-6-0-2/decorate-sql) for every option and for setting them in `liquibase.properties` or environment variables.

## Tips

- **Add new changes in a new file.** Changeset IDs are numbered by position, so inserting statements in the middle of an already-deployed file and decorating it again shifts the IDs and causes checksum errors.
- **Keep the decorated changelog at a stable path.** The changeset identity includes the file path.
- **Review before you deploy.** Because `decorate-sql` needs no database connection, you can run it in CI and check the output with `liquibase checks run` before `update`.
