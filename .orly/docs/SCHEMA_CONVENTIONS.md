# Schema Conventions

## Repository policy

Read the repository's schema sources, migration registration and deployment
policy before changing its database. Preserve shipped migrations and live data;
destructive changes need the owner's approval for the specific change.
Never infer a disposable database from a version threshold in another product.

Use the repository's naming, identity and timestamp conventions consistently.
Qualify table names, grant only the operations each runtime role needs, and test
with the real schema through shared fixture helpers.
Keep migration registration and position assertions synchronized with each edit.
