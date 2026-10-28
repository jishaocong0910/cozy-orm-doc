---
title: 高级执行器
weight: 4
---

# 高级执行器

高级执行器提供了单表CRUD功能，所有创建执行器的方法都具有一个泛型`E`，泛型必须是实体，且不能是[预定义实体](../../entity/tuple)。

## Find

`Find`执行器通过方法`orm.DB.Find[E]`创建，用于查询多行记录。如果配置了[软删除模式](../../db_instance/column_policy/#软删除模式)，会自动添加正常（未删除）数据的过滤条件。

| 方法                              | 描述                                                         |
|-----------------------------------|--------------------------------------------------------------|
| `Select(...string)`               | 查询的表字段，默认为所有字段。                               |
| `OnDemand(*orm.Demand)`           | 按需指定查询的表字段，详见[[按需字段]](#按需字段)。          |
| `Cond(func(c *orm.Cond))`         | 条件处理函数，详见[[查询条件]](../condition)。               |
| `OrderBy(func(o *orm.OrderBy))`   | 排序处理函数，详见[[排序]](#排序)。                          |
| `Page(offset int, rowCount int) ` | 分页，需配置[分页模式](../../db_instance/config/#分页模式)。 |
| `IncludeDeleted()`                | 包含软删除的数据（启用软删除模式时使用）。                   |
| `LastClause(string)`              | SQL末尾的子句，例如`FOR UPDATE`。                            |
| `Do() ([]*E, error)`              | 执行并返回实体。                                             |

*Example*

```go
type User struct {
	Id      *int64
	Name    *string
}

func main() {
	rawDB, _ := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")

	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL, //自动配置分页模式
	}.Build()

	users := db.Find[User](nil).Select("id", "name").
		Cond(func(c *orm.Cond) {
			c.Gt("id", 10)
		}).
		OrderBy(func(o *orm.OrderBy) {
			o.Asc("status").Desc("create_at")
		}).Page(0, 10).LastClause("FOR UPDATE").Do()
	// 执行SQL:
	// SELECT id, name FROM user WHERE id > 10 ORDER BY status ASC, create_at DESC LIMIT 10 FOR UPDATE
}
```

## FindOne

`FindOne`执行器通过方法`orm.DB.FindOne[E]`创建，它基于`Find`执行器，用于查询单行记录。

| 方法                              | 描述                                                 |
|-----------------------------------|------------------------------------------------------|
| `Select(...string)`               | 同`Find`执行器。                                     |
| `OnDemand(*orm.Demand)`           | 同`Find`执行器。                                     |
| `Cond(func(c *orm.Cond))`         | 同`Find`执行器。                                     |
| `OrderBy(func(o *orm.OrderBy))`   | 同`Find`执行器。                                     |
| `Page(offset int, rowCount int) ` | 同`Find`执行器。                                     |
| `IncludeDeleted()`                | 同`Find`执行器。                                     |
| `LastClause(string)`              | 同`Find`执行器。                                     |
| `Lenient()`                       | 指示查询到多行记录时返回首个记录（默认为返回错误）。 |
| `Do() (*E, error)`                | 执行并返回实体。                                     |

## Insert

`Insert`执行器通过方法`orm.DB.Insert[E]`创建，用于插入记录。支持所有[获取生成Key模式](../../db_instance/config/#获取生成key模式)，其中对于`Oracle`模式限制插入的实体数量不能超过1000个。

| 方法                  | 描述                                                                           |
|-----------------------|--------------------------------------------------------------------------------|
| `Entities(...*E)`     | 插入的实体，只会插入非`nil`字段。若插入多个实体，则以首个实体的非nil字段为准。 |
| `Required(...string)` | 必定会插入的字段，若值为`nil`，则插入`null`。                                  |
| `Do() (int64, error)` | 执行并返回影响行数。                                                           |

*Example*

```go
type User struct {
	Id     *int64 `orm:"auto"`
	Name   *string
	Email  *string
	Status *int8
}

func main() {
	rawDB, _ := sql.Open("postgres", "postgres://postgres:12345678@localhost:5432/postgres?sslmode=disable")

	db := orm.Config{
		RawDB:       rawDB,
		DBType:      orm.DBType_.PostgreSQL, //自动配置获取生成Key模式
	}.Build()

	user := &User{Name: new("Alex")}
	
	db.Insert[User](nil).Entities(user).Required("status").Do()
	// 执行SQL:
	// INSERT INTO user(name, status) VALUES('Alex', NULL) RETURNING id
	
	fmt.Println(*user.Id) //打印生成的ID

	/* 插入多个实体以首个实体的非nil字段为准 */
	
	db.Insert[User](nil).Entities(
		&User{Name: new("John")},
		&User{Name: new("Charlie"), Email: new("charlie@example.com")},
	).Do()
	// 执行SQL:
	// INSERT INTO user(name) VALUES ('John'), ('Charlie')

	db.Insert[User](nil).Entities(
		&User{Name: new("Tom"), Email: new("tom@example.com")},
		&User{Name: new("Jack")},
	).Do()
	// 执行SQL:
	// INSERT INTO user(name, email) VALUES ('Tom', 'tom@example.com'), ('Jack', NULL)
}
```

## Update

`Update`执行器通过方法`orm.DB.Update[E]`创建，用于更新记录。如果配置了[软删除模式](../../db_instance/column_policy/#软删除模式)，会自动添加正常（未删除）数据的过滤条件。

| 方法                                         | 描述                                                                                                         |
|----------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| `Entity(*E)`                                 | 更新的实体，只会更新非`nil`字段。带`pk`[标签](../../entity/tag)的字段不会被更新，且非`nil`时会自动作为条件。 |
| `Required(...string)`                        | 必定会更新的字段，若值为`nil`，则更新为`null`。                                                              |
| `OnDemand(*orm.Demand)`                      | 按需指定更新的表字段，详见[[按需字段]](#按需字段)。                                                          |
| `Set(string, any)`                           | 设置字段的更新值。                                                                                           |
| `SetExpr(string, string)`                    | 设置字段更新为指定的SQL表达式。                                                                              |
| {{<code>}}Cond(func(c *orm.Cond)){{</code>}} | 条件处理函数，详见[[查询条件]](../condition)。                                                               |
| `IncludeDeleted()`                           | 包含软删除的数据（启用软删除模式时使用）。                                                                   |
| `SkipSafety()`                               | 指示跳过防止全表操作的安全检查。                                                                             |
| `Do() (int64, error)`                        | 执行并返回影响行数。                                                                                         |

*Example*

```go
type User struct {
	Id       *int64 `orm:"pk"`
	Name     *string
	Email    *string
	Status   *int8
	UpdateAt *time.Time
}

func main() {
	rawDB, _ := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")

	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
	}.Build()

	user := &User{Id: new(int64(1)), Name: new("Alex")}

	affected, _ := db.Update[User](nil).Entity(user).SetExpr("update_at", "now()").Required("status").
		Cond(func(c *orm.Cond) {
			c.Eq("status", 1)
		}).Do()
	// 执行SQL:
	// UPDATE user SET name = 'Alex', status = NULL, update_at = now() WHERE id = 1 AND status = 1
}
```

> [!TIP]
>
> 参数及[字段策略](../../db_instance/column_policy/#字段策略)都会影响字段更新，详见[[更新字段优先级]](#更新字段优先级)。

## UpdateBatch

`UpdateBatch`执行器通过方法`orm.DB.UpdateBatch[E]`创建，用于批量按行更新记录。它要求实体`E`必须有且仅有一个带`pk`[标签](../../entity/tag)的字段，更新的实体中该字段不能为`nil`。如果配置了[软删除模式](../../db_instance/column_policy/#软删除模式)，会自动添加正常（未删除）数据的过滤条件。

| 方法                                         | 描述                                                                                                                                                            |
|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Entities(... *E)`                           | 更新的实体，只会更新非`nil`字段，若更新多个实体，则以首个实体的非nil字段为准。带`pk`[标签](../../entity/tag)的不会被更新，会自动作为更新条件，且必须不为`nil`。 |
| `Required(...string)`                        | 必定会更新的字段，若值为`nil`，则插入`null`。                                                                                                                   |
| `OnDemand(*orm.Demand)`                      | 按需指定更新的表字段，详见[[按需字段]](#按需字段)。                                                                                                             |
| `Set(string, any)`                           | 设置字段的更新值。                                                                                                                                              |
| `SetExpr(string, string)`                    | 设置字段更新为指定的SQL表达式。                                                                                                                                 |
| {{<code>}}Cond(func(c *orm.Cond)){{</code>}} | 条件处理函数，详见[[查询条件]](../condition)。                                                                                                                  |
| `IncludeDeleted()`                           | 包含软删除的数据（启用软删除模式时使用）。                                                                                                                      |
| `Do() (int64, error)`                        | 执行并返回影响行数。                                                                                                                                            |

*Example*

```go
type User struct {
	Id        *int64 `orm:"pk"`
	Name      *string
	Email     *string
	AvatarUrl *string
	Status    *int8
	UpdateAt  *time.Time
}

func main() {
	rawDB, _ := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")

	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
	}.Build()

	users := []*User{
		{
			Id: new(int64(1)), Name: new("Alex"),
			Email: new("alex@example.com"),
		},
		{
			Id: new(int64(2)), Name: new("John"),
			Email: new("john@example.com"),
		},
		{
			Id: new(int64(3)), Name: new("Charlie"),
			AvatarUrl: new("https://example.com/avatar.jpg"), //不会被更新
		},
	}

    affected, _ := db.UpdateBatch[User](nil).Entities(users...).SetExpr("update_at", "now()").Required("status").
		Cond(func(c *orm.Cond) {
			c.Eq("status", 1)
		}).Do()
	// 执行SQL:
	// UPDATE
	//    user SET
	//    name = CASE id
	//        WHEN 1 THEN 'Alex'
	//        WHEN 2 THEN 'John'
	//        WHEN 3 THEN 'Charlie'
	//    END,
	//    email = CASE id
	//        WHEN 1 THEN 'alex@example.com'
	//        WHEN 2 THEN 'john@example.com'
	//        WHEN 3 THEN NULL
	//    END,
	//    status = NULL,
	//    update_at = now()
	// WHERE id IN(1, 2, 3)
	//   AND status = 1
}
```

> [!TIP]
>
> 参数及[字段策略](../../db_instance/column_policy/#字段策略)都会影响字段更新，详见[[更新字段优先级]](#更新字段优先级)。

## Delete

`Delete`执行器通过方法`orm.DB.Delete[E]`创建，用于物理删除记录。

| 方法                      | 描述                                           |
|---------------------------|------------------------------------------------|
| `Cond(func(c *orm.Cond))` | 条件处理函数，详见[[查询条件]](../condition)。 |
| `SkipSafety()`            | 指示跳过防止全表操作的安全检查。               |
| `Do() (int64, error)`     | 执行并返回影响行数。                           |

*Example*

```go
affected, _ := db.Delete[User](nil).Cond(func(c *orm.Cond) {
	c.Eq("id", 1)
}).Do()
// 执行SQL:
// DELETE FROM user WHERE id = 1
```

## DeleteSoftly

`DeleteSoftly`执行器通过方法`orm.DB.DeleteSoftly[E]`创建，用于软删除记录。必须配置[软删除模式](../../db_instance/column_policy/#软删除模式)才可使用。


| 方法                      | 描述                                           |
|---------------------------|------------------------------------------------|
| `Cond(func(c *orm.Cond))` | 条件处理函数，详见[[查询条件]](../condition)。 |
| `SkipSafety()`            | 指示跳过防止全表操作的安全检查。               |
| `Do() (int64, error)`     | 执行并返回影响行数。                           |

*Example*

```go
type User struct {
	Id      *int64 `orm:"pk"`
	Name    *string
	Deleted *int64
}

func main() {
	rawDB, _ := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")

	db := orm.Config{
		RawDB:       rawDB,
		DBType:      orm.DBType_.MySQL,
		ColumnPolicyConfigs: orm.ColumnPolicyConfigs{
			orm.NewColumnPolicyConfig("deleted").OnDeleteSoftly().AssignedPkMode(0),
		},
	}.Build()

	db.DeleteSoftly[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL: 
	// UPDATE user SET deleted = id WHERE id = 1 AND deleted = 0
}
```

## 按需字段

高级执行器的查询/更新操作，可通过任意结构体来确定需要查询/更新的字段。这个结构体对于查询操作来说，是一个最终会被映射的目标，对于更新操作，则是更新数据的来源。

执行器的`OnDemand`参数用于设置字段需求，字段需求通过`orm.DemandFor[D]()`创建，其中`D`必须是结构体。`D`与`E`名称相同的字段将被作为需求字段。

*查询按需字段示例*

假设有个Http接口用于查询简单的用户信息，可根据接口的响应体来确定查询字段。

```go
// user表实体
type User struct {
	_         struct{}   `orm:"table=user"`
	Id        *int64     `orm:"column=id;pk"`
	Name      *string    `orm:"column=name"`
    AvatarUrl *string    `orm:"column=avatar_url"`
	Email     *string    `orm:"column=email"`
	Status    *int8      `orm:"column=status"`
	CreateAt  *time.Time `orm:"column=create_at"`
	UpdateAt  *time.Time `orm:"column=update_at"`
}

// 接口响应体
type UserBasicResp struct {
	Id        *int64  `json:"id"`
    Name      *string `json:"name"`
    AvatarUrl *string `json:"avatarUrl"`
}

func main() {
	http.HandleFunc("GET /user/basic", func(writer http.ResponseWriter, request *http.Request) {
        id := request.URL.Query().Get("id")
		
		// ...

		// 按需指定查询字段，若不指定则会查询所有字段。
		user, _ := db.FindOne[User](nil).
			//Select("id", "name", "avatar_url"). //硬编码方式
			OnDemand(orm.DemandFor[UserBasicResp]()).
			Cond(func(c *orm.Cond) {
				c.Eq("id", id)
			}).Do()
		// 执行SQL:
		// SELECT id, name, avatar_url FROM user WHERE id = ?

        resp := UserBasicResp{
			Id:        user.Id,
			Name:      user.Name,
			AvatarUrl: user.AvatarUrl,
		}

		writer.Header().Set("Content-Type", "application/json")
		json.NewEncoder(writer).Encode(resp)
	})
	
	http.ListenAndServe(":8080", nil)
}
```

*更新按需字段示例*

假设有个Http接口用于更新用户信息，可根据接口请求体来确定更新的字段。

```go
// user表实体
type User struct {
	_         struct{}   `orm:"table=user"`
	Id        *int64     `orm:"column=id;pk"`
	Name      *string    `orm:"column=name"`
	AvatarUrl *string    `orm:"column=avatar_url"`
	Email     *string    `orm:"column=email"`
	Status    *int8      `orm:"column=status"`
	CreateAt  *time.Time `orm:"column=create_at"`
	UpdateAt  *time.Time `orm:"column=update_at"`
}

// 接口请求体
type UserUpdateReq struct {
	Id        *int64  `json:"id"`
	Name      *string `json:"name"`
	AvatarUrl *string `json:"avatarUrl"`
	Email     *string `json:"email"`
}

func main() {
	http.HandleFunc("POST /user/update", func(writer http.ResponseWriter, request *http.Request) {
		var req UserUpdateReq
		json.NewDecoder(request.Body).Decode(&req)

		// ...

		// 按需指定更新字段，若不指定则只会更新非nil字段，无法设置null值。
		db.Update[User](nil).Must().
			Entity(&User{
				Id:        req.Id,
				Name:      req.Name,
				AvatarUrl: req.AvatarUrl,
				Email:     req.Email,
			}).
			//Required("name", "avatar_url", "email"). //硬编码方式
			OnDemand(orm.DemandFor[UserUpdateReq]()).
			Do()
		// req的不为nil的字段为Name和AvatarUrl时，执行SQL:
		// UPDATE user SET name = ?, avatar_url = ?, email = NULL WHERE id = ?
	})
	http.ListenAndServe(":8080", nil)
}
```

## 更新字段优先级

以下字段一定出现在`Update`和`UpdateBatch`执行器的更新字段列表中：

* `OnDemand`匹配的字段。
* 无`OnDemand`参数时，更新的实体中非nil字段。
* `Set`、`SetExpr`指定的字段，`Required`指定的字段，[字段策略](../../db_instance/column_policy/#字段策略)命中字段。

字段的赋值优先级按以下顺序：

1. 强制的字段策略生成的值。
2. `Set`、`SetExpr`指定的值。
3. 非`nil`的实体字段值。
4. 非强制的字段策略生成的值。