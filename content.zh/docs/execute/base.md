---
title: 基础执行器
weight: 3
---

# 基础执行器

基础执行器是CozyORM的核心功能，它通过自定义SQL执行，并且兼容数据库获取生成Key的机制。

## Query

`Query`执行器使用方法`orm.DB.Query[E]`创建，其中泛型`E`必须是结构体，用于执行SELECT语句，底层使用方法`sql.Stmt.QueryContext`执行。

| 方法                                                   | 描述                                                             |
|--------------------------------------------------------|------------------------------------------------------------------|
| {{<code>}}BuildSql(func(b *orm.SQLBuilder)){{</code>}} | 构建SQL处理函数，详见[[SQL构建器]](../sql_builder)               |
| `MapTo(...*E)`                                         | 将查询结果映射到指定的实体，指定后`Do`方法将不会再返回查询结果。 |
| `Do() ([]*E, error)`                                   | 执行SQL并返回实体。                                              |

*Example*

```go
users, _ := db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
	b.Text("SELECT id, name, email FROM user WHERE id = 1")
}).Do()
```

## Mutation

`Mutation`执行器使用方法`orm.DB.Mutation`创建，用于执行INSERT、UPDATE和DELETE语句，底层使用方法`sql.Stmt.ExecContext`执行。

| 方法                                                   | 描述                                                                                                                                                                                                                                        |
|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| {{<code>}}BuildSql(func(b *orm.SQLBuilder)){{</code>}} | 构建SQL处理函数，详见[[SQL构建器]](../sql_builder)                                                                                                                                                                                          |
| `MapTo[E]( ...*E)`                                     | 将`sql.Result.LastInsertId`方法返回的ID值映射到指定实体。需配置[获取生成Key模式](../../db_instance/config/#获取生成key模式)为`FirstInsertId`或`LastInsertId`，泛型`E`必须是一个有且仅有一个带`auto`[标签](../../entity/tag)字段的实体类型。 |
| `Do() (int64, error)`                                  | 执行SQL并返回影响行数。                                                                                                                                                                                                                     |

*Example*

```go   
affected, _ := db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
	b.Text("UPDATE user SET name = 'Alice' WHERE id = 1")
}).Do()
```

## 获取生成Key

对于支持`sql.Result.LastInsertId`方法的数据库，例如MySQL、SQLite，可使用`Mutation`执行器来获取生成的Key。

*Mutation执行器示例*

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

	db.Mutation(nil).MapTo(users...).BuildSql(func(b *orm.SQLBuilder) {
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

部分数据库不支持`sql.Result.LastInsertId`方法，而是通过查询结果集返回生成Key，例如PostgreSQL、SQL Server。`Query`执行器兼容这种机制。通过`MapTo`方法可将生成的Key映射到实体中。

*PostgreSQL获取生成Key（Go原生示例）*

```go
type User struct {
	Id    *int64
	Name  *string
	Email *string
}

func main() {
	rawDB, _ := sql.Open("postgres", "postgres://postgres:12345678@localhost:5432/postgres?sslmode=disable")

	users := []*User{
		{Name: new("Alex"), Email: new("alex@example.com")},
		{Name: new("John"), Email: new("john@example.com")},
		{Name: new("Charlie"), Email: new("charlie@example.com")},
	}

	rows, err := rawDB.Query("INSERT INTO user(name, email) VALUES($1, $2), ($3, $4), ($5, $6) RETURNING id",
		users[0].Name, users[0].Email, users[1].Name, users[1].Email, users[2].Name, users[2].Email)
	if err != nil {
		panic(err)
	}

	var ids []int64
	for i := 0; rows.Next(); i++ {
		var id int64
		rows.Scan(&id)
		ids = append(ids, id)
	}

	fmt.Println(ids[0])
	fmt.Println(ids[1])
	fmt.Println(ids[2])
}
```

*PostgreSQL获取生成Key（Query执行器示例）*

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

	db.Query[User](nil).MapTo(users...).BuildSql(func(b *orm.SQLBuilder) {
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

> [!WARNING]
>
> 一些处理插入冲突的数据库方言可能导致获取的生成Key不准确，例如：
> * MySQL：`ON DUPLICATE KEY UPDATE ...`
> * SQLite、PostgreSQL：`ON CONFLICT ... DO NOTHING`。
