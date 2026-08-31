# 20 — Database Migrations

A **database migration** is a versioned, controlled change to a database schema or data that can be applied consistently across environments.

## 1. Why migrations exist

Changing a Java entity does not automatically mean every database environment has the required schema.

Typical environments include:

```text
Developer DB
Test DB
Staging DB
Production DB
```

Manual schema changes can leave environments at different schema versions.

Migrations make schema evolution explicit and repeatable:

```text
schema change
    ↓
versioned migration
    ↓
stored in source control
    ↓
reviewed
    ↓
applied during deployment
```

## 2. Versioned migration files

A common Flyway structure is:

```text
src/main/resources/db/migration/

V1__create_users.sql
V2__add_phone_number.sql
V3__create_orders.sql
```

Example:

```sql
ALTER TABLE users
ADD COLUMN phone_number VARCHAR(20);
```

The database maintains migration history so the tool knows which migrations have already been applied.

Conceptually:

```text
V1 ✅
V2 ✅
V3 ❌
```

Running the application/deployment causes the pending migration to be applied:

```text
V1 ✅
V2 ✅
V3 ✅
```

## 3. Flyway

**Flyway** is a common database-migration tool used with Spring Boot.

Conceptually:

```text
Spring Boot
     ↓
Flyway
     ↓
PostgreSQL
```

Flyway uses versioned migration files and maintains a schema-history table recording applied migrations.

## 4. Why not rely on Hibernate `ddl-auto=update` in production?

Hibernate can automatically attempt schema changes based on entity definitions:

```properties
spring.jpa.hibernate.ddl-auto=update
```

This can be convenient during development, but production schema evolution is generally better handled by explicit migrations.

With explicit migrations:

```text
Developer
   ↓
writes migration
   ↓
Git / code review
   ↓
deployment
   ↓
migration runs
   ↓
production schema changes
```

The schema history is explicit, reviewable, and reproducible.

## 5. Backward-compatible migrations

The difficult part of production migrations is that application deployment and database migration may overlap.

During a rolling deployment, different application versions can temporarily run at the same time:

```text
Server A → V2
Server B → V1
Server C → V1
```

The database change must therefore often be compatible with both old and new application versions during the transition.

## 6. Expand → Migrate → Contract

A common safe pattern is:

```text
EXPAND
   ↓
MIGRATE
   ↓
CONTRACT
```

Example: rename `full_name` to `name`.

### Step 1 — Expand

Add the new column without immediately removing the old one:

```sql
ALTER TABLE users
ADD COLUMN name VARCHAR(255);
```

Now both exist:

```text
full_name
name
```

This gives old and new application versions a transition period.

### Step 2 — Migrate

Backfill existing data:

```sql
UPDATE users
SET name = full_name
WHERE name IS NULL;
```

Application code can then be changed to understand/use the new column. During a transition, an application may temporarily write both columns when necessary.

### Step 3 — Contract

After old application versions no longer depend on `full_name`, remove it:

```sql
ALTER TABLE users
DROP COLUMN full_name;
```

The key idea is that destructive changes happen only after consumers have stopped depending on the old schema.

## 7. Safely adding a required column

Adding a nullable column is often easier than immediately adding a required one.

A safer sequence can be:

```text
1. Add nullable column
2. Deploy code that handles NULL
3. Backfill existing rows
4. Deploy code that always supplies a value
5. Add NOT NULL constraint
```

This avoids forcing old application versions to immediately satisfy a new constraint they don't know about.

## 8. Safely removing a column

Removing a column can break an older application version that still queries or writes it.

Safer sequence:

```text
application stops using old_column
        ↓
old application versions are gone
        ↓
drop old_column
```

Again, this is an application/database deployment-order problem.

## 9. Rollbacks are not always simple

A schema change can sometimes be reversed, but data changes may not be recoverable simply by reversing the DDL.

For example:

```sql
ALTER TABLE users DROP COLUMN old_phone;
```

Adding the column back does not restore the data that was deleted.

Therefore production systems often favor:

- forward-compatible migrations
- backups/recovery mechanisms for serious data-loss scenarios
- carefully planned destructive changes

Do not assume every migration has a safe automatic rollback.

## 10. Migration files should be treated as immutable

Once a versioned migration has been applied, do not casually edit it.

For example:

```text
V2 → already applied
```

If a new change is required, create:

```text
V3 → new change
```

Think of migration history as an append-only sequence.

## 11. Hibernate vs migration tool

These have different responsibilities:

```text
Hibernate/JPA
→ maps Java entities to database data

Flyway
→ manages explicit database schema evolution
```

A useful Spring Boot architecture is:

```text
Spring Boot
   ├── Hibernate/JPA → entity/data mapping
   │
   └── Flyway        → schema migrations
              ↓
          PostgreSQL
```

## Interview Quick Recall

> Migration = a versioned, controlled database schema/data change.

> Store migrations in source control and apply them consistently across environments.

> Flyway is a common migration tool with Spring Boot.

> Explicit migrations are generally preferred over relying on Hibernate `ddl-auto=update` for production schema evolution.

> Rolling deployments mean old and new application versions may temporarily coexist, so schema changes often need backward compatibility.

> Expand → Migrate → Contract is a common safe pattern for schema changes such as renames/removals.

> Add nullable → backfill → deploy code using it → enforce NOT NULL is a common safe pattern for introducing required data.

> Destructive schema changes should happen only after old application versions no longer depend on the old structure.

> Migration rollback is not necessarily data rollback; restoring deleted data may require backups/recovery.

> Applied migration files should be treated as immutable; add a new migration for a new change.
