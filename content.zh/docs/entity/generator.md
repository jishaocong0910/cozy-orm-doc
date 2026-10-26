---
title: 代码生成器
weight: 5
---

# 代码生成器

CozyORM提供了实体代码生成器。

## 安装

```shell
go get github.com/jishaocong0910/cozy-orm-gen
```

## 快速开始

```go
package main

import (
	"database/sql"

	orm "github.com/jishaocong0910/cozy-orm"
	gen "github.com/jishaocong0910/cozy-orm-gen"
	
	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	db, err := sql.Open("pgx", "postgres://postgres:12345678@localhost:5432/postgres?sslmode=disable")
	if err != nil {
		panic(err)
	}

	gen.Config{
		DB:       db,
		DBType:   orm.DBType_.PostgreSQL,
		FilePath: "repo/entity",
		Tables:   []string{"user", "order"},
	}.Gen()
}
```

## 配置参数

`gen.Config`字段为生成器的配置参数。

| 配置参数         | 类型                                     | 描述                                                                                                                             |
|------------------|------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| DB               | `*sql.DB`                                | 数据库连接池，必填。                                                                                                             |
| DBType           | `orm.DBType`                             | 数据库类型。通过[枚举](../../db_instance/config/#枚举)`orm.DBType_`选择，当前支持MySQL、PostgreSQL、Oracle、SQL Server、SQLite。 |
| FilePath         | `string`                                 | 输出文件路径。它是`os.Getwd()`的相对路径，为空时使用默认值`entity`。                                                             |
| CoverFile        | `bool`                                   | 是否覆盖已有文件。                                                                                                               |
| OmitNameTag      | `bool`                                   | 不添加`table`和`column`[标签](../../entity/tag)。                                                                                |
| Tables           | `[]string`                               | 需要生成的表。                                                                                                                   |
| IgnoreColumns    | {{<code>}}map[string][]string{{</code>}} | 指定忽略的表字段，这些字段不会出现在实体中，key为表名，value为字段名。                                                           |
| EntityNamePrefix | `string`                                 | 实体名称前缀。                                                                                                                   |
| EntityNameSubfix | `string`                                 | 实体名称后缀。                                                                                                                   |
| TypeMappings     | `gen.TypeMappings`                       | 自定义类型映射规则。                                                                                                             |

## 自定义类型

默认情况下，生成的实体字段是Go内置类型。在Go内置类型不满足业务的场景下，可以在生成实体后手动修改，也可以配置生成器的`TypeMappings`参数，将指定的数据库类型映射到自定义或第三方类型。

通过`gen.NewTypeMapping[T](columnType string)`创建类型映射规则，生成器将按照顺序匹配。泛型`T`为实体字段类型，必须满足[[字段类型]](../field_type)要求，为了方便，允许指针写成值类型。`columnType`参数为数据库字段类型，格式为`<data_type>[(<parameter_list>)]`，参数间分隔符为`,`或空格，通配符`%`表示单个任意值，`_`表示非0。

*Example*

```go
package main

import (
	"database/sql"

	"github.com/jackc/pgx/v5/pgtype"
	orm "github.com/jishaocong0910/cozy-orm"
	gen "github.com/jishaocong0910/cozy-orm-gen"
	"github.com/shopspring/decimal"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	db, err := sql.Open("pgx", "postgres://postgres:12345678@localhost:5432/postgres?sslmode=disable")
	if err != nil {
		panic(err)
	}

	gen.Config{
		DB:     db,
		DBType: orm.DBType_.MySQL,
		Tables: []string{"user", "order"},
		// 按照顺序匹配
		TypeMappings: gen.TypeMappings{
			// 整数的numeric映射为*int64
			gen.NewTypeMapping[int64]("numeric(%,0)"),
			// 非0小数的numeric映射为github.com/shopspring/decimal库的*decimal.Decimal类型
			gen.NewTypeMapping[decimal.Decimal]("numeric(%, _)"),
			// time类型映射为pgx驱动的*pgtype.Time类型
			gen.NewTypeMapping[pgtype.Time]("time"),
		},
	}.Gen()
}
```

> [!TIP]
>
> 具有精度和小数的数字类型，例如`NUMERIC`、`DECIMAL`和`NUMBER`，小数为0时可省略，精度和小数都为任意时可省略参数，例如：
>
> - `DECIMAL(10)`相当于`DECIMAL(10,0)`
> - `DECIMAL`相当于`DECIMAL(%,%)`