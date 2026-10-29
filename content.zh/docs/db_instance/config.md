---
title: 配置
weight: 2
---

# 配置

## 配置参数

`orm.Config`的字段为DB实例的配置参数。

| 参数                | 类型                      | 描述                                                                                                                                                                                                                                                                 |
|---------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RawDB               | `*sql.DB`                 | 数据库连接池，必填。                                                                                                                                                                                                                                                 |
| Logger              | `orm.Logger`              | 日志记录器，用于打印SQL等日志，详见[日志记录器](../../log/logger)。                                                                                                                                                                                                  |
| SqlLogLevel         | `orm.LogLevel`            | SQL日志级别。通过枚举`orm.LogLevel_`选择。                                                                                                                                                                                                                           |
| TabNameMapper       | `*orm.NameMapper`         | 默认的**实体名->表名**映射规则，默认为转小写下划线，见[名称映射](#名称映射)。                                                                                                                                                                                        |
| ColNameMapper       | `*orm.NameMapper`         | 默认的**实体字段名->表字段名**映射规则，默认为转小写下划线，见[名称映射](#名称映射)。                                                                                                                                                                                |
| DBType              | `orm.DBType`              | 数据库类型，指定后将自动适配`ParamPrefix`、&#8203;`GetGeneratedKeyMode`、&#8203;`PageMode`和`IdentifierDelimiter`参数。通过枚举`orm.DBType_`选择，当前支持MySQL、PostgreSQL、Oracle、SQL Server、SQLite。对于Oracle，会同时设置`TabNameMapper`默认值为转大写下划线。 |
| ParamPrefix         | `string`                  | 参数占位符前缀。若为空字符串则参数占位符号为`?`，否则为前缀拼接从`1`开始的递增数字。例如，若前缀为`:`，则参数占位符为`:1`、`:2`、`:3`...，若前缀为`$`，则为`$1`、`$2`、`$3` ...。可通过指定`DBType`参数快速配置。                                                    |
| GetGeneratedKeyMode | `orm.GetGeneratedKeyMode` | 获取生成Key模式。通过枚举`orm.GetGeneratedKeyMode_`选择，见[获取生成Key模式](#获取生成key模式)。可通过指定`DBType`参数快速配置。                                                                                                                                     |
| PageMode            | `orm.PageMode`            | 分页模式。通过枚举`orm.PageMode_`选择（见[分页模式](#分页模式)）。可通过指定`DBType`参数快速配置。                                                                                                                                                                   |
| IdentifierDelimiter | `orm.IdentifierDelimiter` | 标识符界定符。通过枚举`orm.IdentifierDelimiter_`选择，见[标识符界定符](#标识符界定符)。可通过指定`DBType`参数快速配置。                                                                                                                                              |
| ColumnPolicyConfigs | `orm.ColumnPolicyConfigs` | 字段策略配置，详见[字段策略](../column_policy)。                                                                                                                                                                                                                     |

## 枚举

CozyORM中，以下划线结尾的全局变量是枚举类型，通过`<enum>.<option>`选择枚举选项，例如`orm.DBType_.MySQL`。

> [!TIP]
>
> 关于枚举的设计，可查看项目：[https://github.com/jishaocong0910/enum](https://github.com/jishaocong0910/enum)

## 获取生成Key模式

获取生成Key，即获取插入记录中自动递增或序列生成的字段值，不同数据库方式各异。根据数据库特性选择适配的模式，并在实体中对自动生成的字段添加`auto`[标签](../../entity/tag)，即可在插入记录后将字段值设置到实体中。

`orm.GetGeneratedKeyMode_`有以下选项，其中基础执行器`Mutation`只支持`FirstInsertId`和`LastInsertId`，高级执行器`Insert`支持所有。

| 选项            | 描述                                                                                                                                                 |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| FirstInsertId   | 将`sql.Result.LastInsertId()`的返回值作为第一个插入记录的ID，可适配MySQL。                                                                           |
| LastInsertId    | 将`sql.Result.LastInsertId()`的返回值作为最后一个插入记录的ID，可适配SQLite。                                                                        |
| InsertReturning | 通过语法`INSERT ... RETURNING ...`返回生成的Key，可适配PostgreSQL、SQLite。                                                                          |
| SQLServer       | 通过SQL Server方言`INSERT ... OUTPUT ...`返回生成的Key，专门适配SQL Server。                                                                         |
| Oracle          | 通过Oracle的方言`INSERT ... RETURNING ... INTO ...`返回生成的Key，专门适配Oracle，只支持被{{<code>}}GENERATED ... AS IDENTITY{{</code>}}修饰的字段。 |

## 分页模式

`orm.PageMode_`有以下选项，作用于高级执行器`Find`。

| 选项        | 描述                                                     |
|-------------|----------------------------------------------------------|
| LimitOffset | 通过语法`LIMIT ... OFFSET ...`分页。                     |
| OffsetFetch | 通过语法`OFFSET ROWS ... FETCH NEXT ... ROWS ONLY`分页。 |

## 标识符界定符

`orm.IdentifierDelimiter_`有以下选项。

| 选项        | 描述            |
|-------------|-----------------|
| Backtick    | 反引号`` ` ` `` |
| DoubleQuote | 双引号`" "`     |
| Bracket     | 方括号`[ ]`     |

## 名称映射

通过`orm.NewNameMapper()`创建名称映射器，然后链式调用其方法指定映射规则，可指定多个规则，将按调用顺序处理。

| 方法              | 描述         |
|-------------------|--------------|
| LowerCamelCase    | 转小驼峰     |
| LowerSnakeCase    | 转小写下划线 |
| LowerFirstLiteral | 首字母小写   |
| UpperCamelCase    | 转大驼峰     |
| UpperSnakeCase    | 转大写下划线 |
| UpperFirstLiteral | 首字母大写   |
| AddPrefix         | 添加前缀     |
| AddSuffix         | 添加后缀     |
| SubPrefix         | 删除前缀     |
| SubSuffix         | 删除后缀     |

*Example*

```go
type UserInfo struct {
	Id        *int64
	UserId    *int64
	AvatarUrl *string
}

func main() {
	// ...

	db := orm.Config{
		RawDB:         rawDB,
		DBType:        orm.DBType_.MySQL,
		TabNameMapper: orm.NewNameMapper().LowerSnakeCase().AddPrefix("tb_"),
	}.Build()

	db.FindOne[UserInfo](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL:
	// SELECT id, user_id, avatar_url FROM tb_user_info WHERE id = 1
}
```