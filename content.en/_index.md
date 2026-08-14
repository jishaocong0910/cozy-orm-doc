# CozyORM

Lightweight Go ORM with method chaining, minimalist API, and multi-database support.

# Features

- **Method Chaining**: Adopting Go 1.27's revolutionary generic method feature.
- **Minimalist API**: Only two base execution APIs (Query & Mutation), along with common CRUD APIs.
- **Dynamic SQL**: Supports dynamic SQL construction, compatible with all SQL dialects and driver placeholders.
- **Getting Generated Keys**: Provides various strategies for getting generated keys after inserting.
- **Custom Mapping**: Provides enhanced custom type mapping.
- **Transaction Propagation**: Propagates transactions via Context, making database operations reusable.
- **Predefined Entity**: For dynamic mapping when selecting only a few fields.
- **Code Generation**: Provides code generation for supported databases.

## Tested Drivers

- github.com/go-sql-driver/mysql
- github.com/sijms/go-ora/v2
- github.com/godror/godror
- github.com/jackc/pgx/v5
- github.com/lib/pq
- github.com/microsoft/go-mssqldb
- modernc.org/sqlite
- github.com/mattn/go-sqlite3
