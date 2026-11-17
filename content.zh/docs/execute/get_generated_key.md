---
title: 获取生成Key
weight: 5
---

# 获取生成Key

插入记录后，可获取数据库自动生成的key（例如自增列或序列生成的值），并将其回填到实体对应的字段中。

## 模式

不同数据库获取生成Key的方式不同，因此需要根据所用数据库设置DB实例配置的`GetGeneratedKeyMode`参数。同时，在实体中为需要回填的字段添加`auto`[标签](../../entity/tag)。

`orm.GetGeneratedKeyMode_`提供了以下选项：

| 选项            | 描述                                                                                                                                                 | 
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| FirstInsertId   | 将`sql.Result.LastInsertId()`的返回值作为第一个插入记录的ID，可适配MySQL。                                                                           |
| LastInsertId    | 将`sql.Result.LastInsertId()`的返回值作为最后一个插入记录的ID，可适配SQLite。                                                                        |
| InsertReturning | 通过语法`INSERT ... RETURNING ...`返回生成的Key，可适配PostgreSQL、SQLite。                                                                          |
| SQLServer       | 通过SQL Server方言`INSERT ... OUTPUT ...`返回生成的Key，专门适配SQL Server。                                                                         |
| Oracle          | 通过Oracle的方言`INSERT ... RETURNING ... INTO ...`返回生成的Key，专门适配Oracle，只支持被{{<code>}}GENERATED ... AS IDENTITY{{</code>}}修饰的字段。 |

## 限制

`FirstInsertId`和`LastInsertId`模式会先检查所有实体的生成Key字段。只要其中任意一个字段不为`nil`，就不会对任何实体回填生成Key。使用`ON DUPLICATE KEY UPDATE ...`、`ON CONFLICT ... DO NOTHING`或`INSERT IGNORE ...`等冲突处理语法时，回填的Key可能不准确。

`InsertReturning`和`SQLServer`模式在多行插入时，因为数据库不保证返回行的顺序与输入行一致，不保证回填的Key与实体正确对应。此外，`InsertReturning`模式也会受到冲突处理语法的影响，导致回填的Key可能不准确。

`Oracle`模式单次插入最多支持1000行。

为避免上述限制，建议使用自定义ID生成器，在插入前生成ID。

## 基础执行器