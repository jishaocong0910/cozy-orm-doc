---
title: 基础执行器
weight: 3
---

# 基础执行器

基础执行器是CozyORM的核心功能，它通过自定义SQL执行，并且兼容数据库获取生成Key的机制。

## 执行器

### Query

`Query`执行器使用方法`orm.DB.Query[E]`创建，其中泛型`E`必须是结构体，用于执行SELECT语句，底层使用方法`sql.Stmt.QueryContext`执行。

| 方法                                                   | 描述                                                             |
|--------------------------------------------------------|------------------------------------------------------------------|
| {{<code>}}BuildSql(func(b *orm.SQLBuilder)){{</code>}} | 构建SQL处理函数，详见[SQL构建器](#sql构建器)                     |
| `MapTo(...*E)`                                         | 将查询结果映射到指定的实体，指定后`Do`方法将不会再返回查询结果。 |
| `Do() ([]*E, error)`                                   | 执行SQL并返回实体。                                              |

*Example*

```go
users, _ := db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
	b.Text("SELECT id, name, email FROM user WHERE id = 1")
}).Do()
```

### Mutation

`Mutation`执行器使用方法`orm.DB.Mutation`创建，用于执行INSERT、UPDATE和DELETE语句，底层使用方法`sql.Stmt.ExecContext`执行。

| 方法                                                   | 描述                                                                                                                                                                                                                                        |
|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| {{<code>}}BuildSql(func(b *orm.SQLBuilder)){{</code>}} | 构建SQL处理函数，详见[SQL构建器](#sql构建器)                                                                                                                                                                                                |
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

## SQL构建器

`orm.SQLBuilder`用于动态构建SQL语句和设置参数，提供了以下方法并支持链式调用。

### Text

拼接文本。

*Example*

```go
type User struct {
	Id     *int64
	Name   *string
	Email  *string
	Status *int8
}

func main() {
	// ...

	user := &User{
		Id:    new(int64(1)),
		Name:  new("Alex"),
		Email: new("alex@example.com"),
	}

	db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
		i := 1
		sep := " "
		b.Text("Update user SET")
		if user.Name != nil {
			b.Text(sep).Text("name = $"+strconv.Itoa(i)).Args(user.Name)
			i++
			sep = ", "
		}
		if user.Email != nil {
			b.Text(sep).Text("email = $"+strconv.Itoa(i)).Args(user.Email)
			i++
			sep = ", "
		}
		if user.Status != nil {
			b.Text(sep).Text("status = $"+strconv.Itoa(i)).Args(user.Status)
			i++
			sep = ", "
		}
		b.Text(" WHERE id = $"+strconv.Itoa(i)).Args(user.Id)
	}).Do()
	// 执行SQL：
	// Update user SET name = $1, email = $2 WHERE id = $3
	//
	// 参数：
	// $1="Alex" $2="alex@example.com" $3=1
}
```

### Quote

拼接引用标识符，限定符由DB实例配置的`IdentifierDelimiter`参数所指定。

*Example*

```go
type User struct {
	Id     *int64
	Name   *string
	Email  *string
	Status *int8
}

func main() {
	// ...
	
	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.PostgreSQL,
	}.Build()

	user := &User{
		Id:    new(int64(1)),
		Name:  new("Alex"),
		Email: new("alex@example.com"),
	}

	db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
		i := 1
		sep := " "
		b.Text("Update ").Quote("user").Text(" SET")
		if user.Name != nil {
			b.Text(sep).Quote("name").Text(" = $" + strconv.Itoa(i)).Args(user.Name)
			i++
			sep = ", "
		}
		if user.Email != nil {
			b.Text(sep).Quote("email").Text(" = $" + strconv.Itoa(i)).Args(user.Email)
			i++
			sep = ", "
		}
		if user.Status != nil {
			b.Text(sep).Quote("status").Text(" = $" + strconv.Itoa(i)).Args(user.Status)
			i++
			sep = ", "
		}
		b.Text(" WHERE ").Quote("id").Text(" = $" + strconv.Itoa(i)).Args(user.Id)
	}).Do()
	// 执行SQL：
	// Update "user" SET "name" = $1, "email" = $2 WHERE "id" = $3
	//
	// 参数：
	// $1="Alex" $2="alex@example.com" $3=1
}
```

### Arg

拼接参数占位符并设置参数，参数占位符由DB实例配置的`ParamPrefix`参数所指定。

*Example*

```go
type User struct {
	Id     *int64
	Name   *string
	Email  *string
	Status *int8
}

func main() {
	// ...
	
	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.PostgreSQL, // 自动配置参数占位符
	}.Build()

	user := &User{
		Id:    new(int64(1)),
		Name:  new("Alex"),
		Email: new("alex@example.com"),
	}

	db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
		sep := " "
		b.Text("Update ").Quote("user").Text(" SET")
		if user.Name != nil {
			b.Text(sep).Quote("name").Text(" = ").Arg(user.Name)
			sep = ", "
		}
		if user.Email != nil {
			b.Text(sep).Quote("email").Text(" = ").Arg(user.Email)
			sep = ", "
		}
		if user.Status != nil {
			b.Text(sep).Quote("status").Text(" = ").Arg(user.Status)
			sep = ", "
		}
		b.Text(" WHERE ").Quote("id").Text(" = ").Arg(user.Id)
	}).Do()
	// 执行SQL：
	// Update "user" SET "name" = $1, "email" = $2 WHERE "id" = $3
	//
	// 参数：
	// $1="Alex" $2="alex@example.com" $3=1
}
```

### Args

添加参数。

*Example*

```go
db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
	b.Text("SELECT * FROM user WHERE id = $1 AND status = $2")
	b.Args(1, 2)
}).Do()
```

### ForEach

用于遍历切片拼接SQL语句，添加开始、分隔和结束符号。

方法参数：`(sep separate, items []T, handler func(i int, item T))`。其中`handler`的参数`i`为当前元素的索引，`t`为当前元素。`sep`通过SQL构建器的以下方法指定。

| 函数                                       | 描述                                                   |
|--------------------------------------------|--------------------------------------------------------|
| `Sep(separator string)`                    | 拼接分隔符。                                           |
| `SepWrap(open, separator close string)`    | 拼接开始、分隔和结束符。                               |
| `SepWrapOpt(open, separator close string)` | 拼接开始、分隔和结束符，切片为空时不拼接开始和结束符。 |

*Example*

```go
func main() {
	// ...

	columns := []string{"id", "name", "email"}
	ids := []int64{101, 102, 103}
	var status []int8

	db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("SELECT ")
		b.ForEach(b.Sep(", "), columns, func(_ int, item string) {
			b.Text(item)
		})
		b.Text(" FROM user WHERE id IN")
		b.ForEach(b.SepWrap("(", ", ", ")"), ids, func(_ int, item int64) {
			b.Arg(item)
		})
		b.ForEach(b.SepWrapOpt("AND status IN(", ", ", " )"), status, func(_ int, item int8) {
			b.Arg(item)
		})
	}).Do()
	// 执行SQL：
	// SELECT id, name, email FROM user WHERE id IN($1, $2, $3)
	//
	// 参数：
	// $1=101 $2=102 $3=103
}
```

### Accept

接受自定义的SQL拼接。

方法参数：`(w SQLWriter)`。`SQLWriter`是一个接口，要求实现`WriteSQL(*SQLBuilder)`方法，用于自定义拼接SQL语句。

*Example*

```go
type ColumnWriter struct {
	Columns []string
}

func (c ColumnWriter) WriteSQL(b *orm.SQLBuilder) {
	b.ForEach(b.Sep(", "), c.Columns, func(_ int, item string) {
		b.Text(item)
	})
}

func main() {
	// ...

	columns := ColumnWriter{Columns: []string{"id", "name", "email"}}

	db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("SELECT ").Accept(columns).Text(" FROM user WHERE id = 1")
	}).Do()
	// 执行SQL：
	// SELECT id, name, email FROM user WHERE id = 1
}
```

### Cancel

取消SQL的执行，不会返回错误。调用该方法后应立即`return`。

*Example*

```go
func main() {
	// ...

	var ids []int64

	users, err := db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
		if len(ids) == 0 {
			b.Cancel()
			return
		}
		b.Text("SELECT id, name, email FROM user WHERE id IN")
		b.ForEach(b.SepWrap("(", ",", ")"), ids, func(_ int, item int64) {
			b.Arg(item)
		})
	}).Do()
	// users=[]*User{}, err=nil
}
```

### Error

取消SQL的执行，并返回错误。调用该方法后应立即`return`。

*Example*

```go
func main() {
	// ...

	var ids []int64

	users, err := db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
		if len(ids) == 0 {
			b.Error(errors.New("ids is empty"))
			return
		}
		b.Text("SELECT id, name, email FROM user WHERE id IN")
		b.ForEach(b.SepWrap("(", ",", ")"), ids, func(_ int, item int64) {
			b.Text(item)
		})
	}).Do()
	// users=[]*User{}, err="ids is empty"
}
```
