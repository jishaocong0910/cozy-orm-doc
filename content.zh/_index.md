# CozyORM

轻量级Go语言ORM框架，链式调用模式，极少的API，灵活兼容各种数据库。

## 特色

- **链式调用**：依托Go 1.27泛型方法的革命性特性，实现流畅的链式调用。
- **轻量API**：仅查询、变更两个基础执行API，同时封装了CRUD等通用API。
- **SQL构建**：支持动态SQL构建，兼容各种SQL方言与参数占位符。
- **获取生成Key**：兼容各主流数据库插入记录后获取自动生成Key的不同机制。
- **自定义映射**：提供更友好的自定义类型映射功能。
- **事务传播**：基于Context实现事务传播，使数据库操作代码得到复用。
- **预定义实体**：用于查询少量字段场景下进行动态映射。
- **代码生成**：提供多种常用数据库的实体代码生成器。

## 已测试驱动

- github.com/go-sql-driver/mysql
- github.com/sijms/go-ora/v2
- github.com/godror/godror
- github.com/jackc/pgx/v5
- github.com/lib/pq
- github.com/microsoft/go-mssqldb
- modernc.org/sqlite
- github.com/mattn/go-sqlite3