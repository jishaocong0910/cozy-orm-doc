---
title: 获取生成Key
weight: 5
---

# 获取生成Key

获取生成Key即获取插入记录中自动递增或序列生成的字段值，不同数据库方式各异。根据数据库特性选择适配的模式，并在实体中对自动生成的字段添加`auto`[标签](../../entity/tag)，即可在插入记录后将字段值设置到实体中。

`orm.GetGeneratedKeyMode_`有以下选项，其中基础执行器`Mutation`只支持`FirstInsertId`和`LastInsertId`，高级执行器`Insert`支持所有。

| 选项            | 描述                                                                                                                                                 |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| FirstInsertId   | 将`sql.Result.LastInsertId()`的返回值作为第一个插入记录的ID，可适配MySQL。                                                                           |
| LastInsertId    | 将`sql.Result.LastInsertId()`的返回值作为最后一个插入记录的ID，可适配SQLite。                                                                        |
| InsertReturning | 通过语法`INSERT ... RETURNING ...`返回生成的Key，可适配PostgreSQL、SQLite。                                                                          |
| SQLServer       | 通过SQL Server方言`INSERT ... OUTPUT ...`返回生成的Key，专门适配SQL Server。                                                                         |
| Oracle          | 通过Oracle的方言`INSERT ... RETURNING ... INTO ...`返回生成的Key，专门适配Oracle，只支持被{{<code>}}GENERATED ... AS IDENTITY{{</code>}}修饰的字段。 |
