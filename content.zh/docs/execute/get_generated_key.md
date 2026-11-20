---
title: 获取生成Key
weight: 5
---

# 获取生成Key

根据所使用的数据库，在 DB 实例配置中设置`GetGeneratedKeyMode`参数，并为实体中需要回填的字段添加`auto`[标签](../../entity/tag)。配置完成后，使用`Mutation`或`Insert`执行器插入记录时，会将自动生成的 Key 回填到实体对应的字段中。

## 模式

`orm.GetGeneratedKeyMode_`提供了以下选项。`Mutation`只支持`FirstInsertId`和`LastInsertId`模式，`Insert`支持所有模式。

| 选项            | 描述                                                                                                                                                 | 
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| FirstInsertId   | 将`sql.Result.LastInsertId()`的返回值作为第一条插入记录的ID，可适配MySQL。                                                                           |
| LastInsertId    | 将`sql.Result.LastInsertId()`的返回值作为最后一条插入记录的ID，可适配SQLite。                                                                        |
| InsertReturning | 通过`INSERT ... RETURNING ...`语法返回生成的Key，可适配PostgreSQL、SQLite。                                                                          |
| SQLServer       | 通过SQL Server的`INSERT ... OUTPUT ...`语法返回生成的Key，专门适配SQL Server。                                                                       |
| Oracle          | 通过Oracle的`INSERT ... RETURNING ... INTO ...`语法返回生成的Key，专门适配Oracle，只支持被{{<code>}}GENERATED ... AS IDENTITY{{</code>}}修饰的字段。 |

## 限制

`FirstInsertId`和`LastInsertId`模式会先检查所有待插入实体中标记为`auto`的字段，只要任意一个字段的值不为`nil`，就不会执行回填操作。使用`ON DUPLICATE KEY UPDATE ...`、`ON CONFLICT ... DO NOTHING`或`INSERT IGNORE ...`等冲突处理语法时，回填的Key可能不准确。

`InsertReturning`和`SQLServer`模式在多行插入时，由于数据库不保证返回行的顺序与插入行顺序一致，因此回填的 Key 可能与实体不匹配。具体行为可查阅PostgreSQL、SQL Server和SQLite的相关文档。此外，`InsertReturning`模式也会受到冲突处理语法的影响，回填的Key可能不准确。

`Oracle`模式单次插入最多支持1000行。

建议使用自定义ID生成器，在插入前生成ID。