---
title: SQL构建器
weight: 5
---

# SQL构建器

`orm.SQLBuilder`用于动态构建SQL语句和设置参数，提供了以下方法并支持链式调用。

## Text

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

## Quoted

拼接引用标识符，限定符为DB实例配置的`IdentifierDelimiter`参数所指定。

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
		b.Text("Update ").Quoted("user").Text(" SET")
		if user.Name != nil {
			b.Text(sep).Quoted("name").Text(" = $" + strconv.Itoa(i)).Args(user.Name)
			i++
			sep = ", "
		}
		if user.Email != nil {
			b.Text(sep).Quoted("email").Text(" = $" + strconv.Itoa(i)).Args(user.Email)
			i++
			sep = ", "
		}
		if user.Status != nil {
			b.Text(sep).Quoted("status").Text(" = $" + strconv.Itoa(i)).Args(user.Status)
			i++
			sep = ", "
		}
		b.Text(" WHERE ").Quoted("id").Text(" = $" + strconv.Itoa(i)).Args(user.Id)
	}).Do()
	// 执行SQL：
	// Update "user" SET "name" = $1, "email" = $2 WHERE "id" = $3
	//
	// 参数：
	// $1="Alex" $2="alex@example.com" $3=1
}
```

## Arg

拼接参数占位符并设置参数，参数占位符为DB实例配置的`ParamPrefix`参数所指定。

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
		b.Text("Update ").Quoted("user").Text(" SET")
		if user.Name != nil {
			b.Text(sep).Quoted("name").Text(" = ").Arg(user.Name)
			sep = ", "
		}
		if user.Email != nil {
			b.Text(sep).Quoted("email").Text(" = ").Arg(user.Email)
			sep = ", "
		}
		if user.Status != nil {
			b.Text(sep).Quoted("status").Text(" = ").Arg(user.Status)
			sep = ", "
		}
		b.Text(" WHERE ").Quoted("id").Text(" = ").Arg(user.Id)
	}).Do()
	// 执行SQL：
	// Update "user" SET "name" = $1, "email" = $2 WHERE "id" = $3
	//
	// 参数：
	// $1="Alex" $2="alex@example.com" $3=1
}
```

## Args

添加参数。

*Example*

```go
db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
	b.Text("SELECT * FROM user WHERE id = $1 AND status = $2")
	b.Args(1, 2)
}).Do()
```

## ForEach

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

## Accept

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

## Cancel

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

## Error

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