---
title: 获取生成 Key
weight: 5
---

# 获取生成 Key

根据所使用的数据库，在 DB 实例配置中设置 `GetGeneratedKeyMode` 参数，并为实体中需要回填的字段添加 `auto` [标签](../../entity/tag)。配置完成后，使用`Insert` 执行器插入记录时，会将自动生成的 Key 回填到实体对应的字段中。

## 模式

`orm.GetGeneratedKeyMode_` 提供了以下选项。

| 选项            | 描述                                                                                                                                                         | 
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FirstInsertId   | 将 `sql.Result.LastInsertId()` 的返回值作为第一条插入记录的 ID，可适配 MySQL。                                                                               |
| LastInsertId    | 将 `sql.Result.LastInsertId()` 的返回值作为最后一条插入记录的 ID，可适配 SQLite。                                                                            |
| InsertReturning | 通过 `INSERT ... RETURNING ...` 语法返回生成的 Key，可适配 PostgreSQL、SQLite。                                                                              |
| SQLServer       | 通过 SQL Server 的 `INSERT ... OUTPUT ...` 语法返回生成的 Key，专门适配 SQL Server。                                                                         |
| Oracle          | 通过 Oracle 的 `INSERT ... RETURNING ... INTO ...` 语法返回生成的 Key，专门适配 Oracle，只支持被 {{<code>}}GENERATED ... AS IDENTITY{{</code>}} 修饰的字段。 |

## 限制

`FirstInsertId` 和 `LastInsertId` 模式会先检查所有待插入实体中标记为 `auto` 的字段，只要任意一个字段的值不为 `nil`，就不会执行回填操作。使用 `ON DUPLICATE KEY UPDATE ...`、`ON CONFLICT ... DO NOTHING` 或 `INSERT IGNORE ...` 等冲突处理语法时，回填的 Key 可能不准确。

`InsertReturning` 和 `SQLServer` 模式在多行插入时，由于数据库不保证返回行的顺序与插入行顺序一致，因此回填的 Key 可能与实体不匹配。具体行为可查阅 PostgreSQL、SQL Server 和 SQLite 的相关文档。此外，`InsertReturning` 模式也会受到冲突处理语法的影响，回填的 Key 可能不准确。

`Oracle` 模式单次插入最多支持 1000 行。

建议使用自定义 ID 生成器，在插入前生成 ID。

## 使用基础执行器

基础执行器`Query`和`Mutation`也支持在插入记录时获取生成Key并回填到实体中。

`Mutation` 支持和 `FirstInsertId` 和 `LastInsertId` 模式，使用 `Mutation` 执行器插入实体时，通过指定`MapIdentity`参数即可将生成的 Key 回填到实体中。

*Mutation 执行器获取生成Key示例*

```go
type User struct {
	Id    *int64 `orm:"auto"`
	Name  *string
	Email *string
}

func main() {
	rawDB, _ := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")

	db := orm.Config{
		RawDB:               rawDB,
		GetGeneratedKeyMode: orm.GetGeneratedKeyMode_.FirstInsertId,
	}.Build()

	users := []*User{
		{Name: new("Alex"), Email: new("alex@example.com")},
		{Name: new("John"), Email: new("john@example.com")},
		{Name: new("Charlie"), Email: new("charlie@example.com")},
	}

	db.Mutation(nil).MapIdentity(users...).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("INSERT INTO user(name, email) VALUES(?, ?), (?, ?), (?, ?)")
		for _, user := range users {
			b.Args(user.Name, user.Email)
		}
	}).Do()

	fmt.Println(users[0].Id)
	fmt.Println(users[1].Id)
	fmt.Println(users[2].Id)
}
```

PostgreSQL、SQL Server 这类不支持 `sql.Result.LastInsertId` 方法的数据库，它们通过查询结果集返回生成 Key。`Query` 执行器兼容这种机制，通过指定 `Map` 参数可将查询结果集按顺序映射到实体中。指定 `Map` 参数后 `Do` 方法将不会返回实体。

*Query 执行器获取生成Key示例*

```go
type User struct {
	Id    *int64
	Name  *string
	Email *string
}

func main() {
	rawDB, _ := sql.Open("postgres", "postgres://postgres:12345678@localhost:5432/postgres?sslmode=disable")

	db := orm.Config{
		RawDB: rawDB,
	}.Build()

	users := []*User{
		{Name: new("Alex"), Email: new("alex@example.com")},
		{Name: new("John"), Email: new("john@example.com")},
		{Name: new("Charlie"), Email: new("charlie@example.com")},
	}

	db.Query[User](nil).Map(users...).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("INSERT INTO user(name, email) VALUES($1, $2), ($3, $4), ($5, $6) RETURNING id")
		for _, user := range users {
			b.Args(user.Name, user.Email)
		}
	}).Do()

	fmt.Println(users[0].Id)
	fmt.Println(users[1].Id)
	fmt.Println(users[2].Id)
}
```