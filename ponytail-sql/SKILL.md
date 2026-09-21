---
name: ponytail-sql
description: Personal SQL rules - keeping queries (and only queries) in the repository layer and out of controllers, services and stores, how a query is written (named params, tenant scoping, bounded lists, bulk writes, upserts), and defining unique keys together with their table in schema files. Load before writing or reviewing SQL, a repository function, a schema file, or a controller/route handler that touches the database.
---

# ponytail-sql

SQL-only slice of the [ponytail](../ponytail/SKILL.md) working-style rules, for when you want
the SQL conventions without the rest. project-specific CLAUDE.md rules still win where they're
more specific.

## data access: sql lives in repositories, never in controllers

for any codebase with a controller/route layer and a repository/data-access layer, queries
belong only in the repository layer - raw SQL strings and ORM query builders alike.

- a controller/route handler parses input, calls a repository function, and shapes the
  reply - it never issues a query (raw SQL or `db.select()/getDb()`-style builder calls)
  directly against the database
- if a route needs a lookup a repository doesn't yet expose, add or extend a repository
  function for it rather than reaching for the db client inline - check for an existing
  repository function first, since the lookup may already exist
- this applies even for "just a quick lookup" (e.g. resolving an id before a 404 check) -
  small one-off queries are exactly the ones that end up duplicated across route files
- the layer is queries only, in both directions: no service, store or tool holds a db client or
  a query string (a bulk insert and a table of query templates count), and a repository function
  runs one query and returns rows; deduplicating, merging, defaulting, policy and building the
  rows to write live in the service/store that calls it

## sql patterns: how a query is written

applies to any dialect. examples come from endpoint-server's repositories (postgres, `:name`
placeholders); clickhouse says `{name:Type}` instead.

- named parameters only: values are never concatenated or template-interpolated into the SQL
  text. a `${...}` inside a query is only a fragment assembled from fixed strings (an optional
  filter, a `VALUES` row list, a shared predicate constant) whose values went into the params
  object beside it
- scope by tenant on every tenant-owned table, joined tables included
  (`ON d.id = t.device_id AND d.sys_group_id = :sysGroupId`), and take the tenant id as an
  explicit repository argument. a cross-tenant query (global table, fleet-wide analytics) is the
  deliberate exception; make that obvious from the function name or a comment
- name the columns you select instead of `SELECT *`; the exception is copying a row forward
  unchanged (re-inserting into a replacing table), where `*` stops a column added later from
  being reset to its default
- bound every list query with `LIMIT`, and end its `ORDER BY` in a unique column so ties don't
  make pages or cursors skip or repeat rows; a lookup expecting one row says `LIMIT 1`
- push filters into the query: build optional conditions as a list of fixed strings
  (`AND ip.ecosystem = :ecosystem`) and add their values to params, rather than fetching
  broadly and filtering in the caller
- write in bulk, one statement per batch and never one per row (a row per round trip puts
  dozens of serial queries on the request path). keep a batch under the driver's bind-parameter
  limit (postgres: 65535, so chunk a multi-row `VALUES`), or pass parallel arrays and
  `unnest(:a::text[], :b::int[])` so the parameter count stays flat
- upserts name their conflict target (`ON CONFLICT (cols) DO UPDATE SET x = EXCLUDED.x`, or
  `DO NOTHING`). postgres rejects a statement that hits the same row twice, so the caller
  dedupes the batch first; say so on the function
- cast aggregates in the query (`count(*)::int`); the driver hands bigint back as a string
- `RETURNING id` when the caller needs the new id or needs to know a row matched

## schema files: define unique keys together with the table

in a schema file (e.g. `schema.sql`), a table's unique keys go right after that table's
`CREATE TABLE`, before the next table. never gather them in a separate block at the end
of the file.

- order per table: `CREATE TABLE`, then that table's `CREATE UNIQUE INDEX` (postgres) or
  `ALTER TABLE ... ADD UNIQUE KEY` (mysql), then the next table
- adding a table: write its unique keys with it, in the same place
- adding a unique key to an existing table: put it after that table, even if the file
  already collects keys at the bottom
- why: reading one table's definition should show which columns are unique, without
  searching the rest of the file
