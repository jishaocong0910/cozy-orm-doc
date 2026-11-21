---
title: 公共参数
weight: 2
---

# 公共参数

所有执行器都具有一个参数 `ctx context.Context`，它会透传到[日志记录器](../../log/logger)、[字段策略](../../db_instance/column_policy)，用于[事务传播](../../transaction/propagation)，若为 `nil` 时将使用 `context.Background()` 作为默认值。`ctx` 是执行器关联事务、传递上下文的关键，因此设计为在创建执行器时传入。

除了 `ctx` 之外，其他参数都是通过链式调用方法进行设置，所有执行器都具有的以下方法。

| 方法                     | 描述                                                                                                                                                                                               |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Must()`                 | 指示发生错误时 panic，并将原错误作为 panic 值（默认为返回错误）。                                                                                                                                  |
| `Description(string)`    | 用于描述 SQL 业务，打印在 SQL 日志里。                                                                                                                                                             |
| `SqlLogLevel(orm.LogLevel)` | 指定 SQL 日志的级别。通过[枚举](../../db_instance/config/#枚举) `orm.LogLevel_` 选择，优先级高于 DB 实例配置的 `SqlLogLevel` 参数。指定为 `orm.LogLevel_.UNDEFINED` 可以忽略打印本次执行的 SQL 日志。 |

*Example*

```go
db.FindOne[User](ctx).Description("describe the SQL").Cond(func(c *orm.Cond) {
	c.Eq("id", 1)
}).Do()
```