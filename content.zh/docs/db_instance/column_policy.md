---
title: 字段策略
weight: 3
---

# 字段策略

字段策略作用于高级执行器，用于配置字段在数据库操作事件中的统一处理逻辑。通过
{{<code>}}orm.NewColumnPolicyConfig(column string){{</code>}}创建字段策略配置，其方法`ForTable`用于指定作用的表，若不指定则为所有表的默认配置，方法`IgnoreTable`可排除指定表。每个表的字段只会使用一个策略配置，匹配到多个时，优先使用指定了 `ForTable` 的配置，依然后多个则取最后添加的那个。

## 事件处理

字段策略可配置插入、更新和软删除事件，对应方法`OnInsert`、`OnUpdate`、`OnDeleteSoftly`，事件提供了如下处理方法。

{{<html>}}
<table>
    <thead>
        <tr>
            <th>事件</th><th>处理方法</th><th>描述</th>
        </tr>
    </thead>
    <tbody> 
        <tr>
            <td rowspan="3">OnInsert/OnUpdate</td><td style="width: 21em;"><code>Value(force bool, batchReuse bool, value func(ctx context.Context) any)</code></td><td>在插入/更新记录时生成字段值。<code>force</code>指定是否强制使用生成的值，若为<code>false</code>则仅在插入/更新的值为<code>nil</code>时使用。<code>batchReuse</code>指定是否在批量插入/更新（高级执行器<code>Insert</code>/<code>UpdateRows</code>）时只生成一次并复用，若为<code>false</code>则每行都生成一次。<code>value</code>为字段值的生成函数。</td>
        </tr>
        <tr>
            <td><code>Expr(force bool, batchReuse bool, expr func(ctx context.Context) string)</code></td><td>与<code>Value</code>方法功能相同，区别是字段值为<code>expr</code>函数返回的SQL表达式。</td>
        </tr>
        <tr>
            <td><code>Never()</code></td><td>不赋值。将忽略插入/更新记录时对字段的赋值。</td>
        </tr>
        <tr>
            <td rowspan="2">OnDeleteSoftly</td><td><code>AssignedPkMode[T](normalValue&nbsp;T)</code></td><td rowspan="2">指定软删除模式，见<a href="#软删除模式">[软删除模式]</a>。</td>
        </tr>
        <tr>
            <td><code>AssignedNullMode[T](normalValue&nbsp;T)</code>
        </tr>
    </tbody>
</table>
{{</html>}}

*Example*

```go
db := orm.Config{
	RawDB:  rawDB,
	DBType: orm.DBType_.MySQL,
	ColumnPolicyConfigs: orm.ColumnPolicyConfigs{
		// 所有表的id字段使用自定义的ID生成器创建，并且禁止更新。
		orm.NewColumnPolicyConfig("id").
			OnInsert().Value(true, false, func(ctx context.Context) any {
			return myIdGenerator.Int64()
		}).OnUpdate().Never(),

		// 插入user表时，若name字段值为nil则随机生成。
		orm.NewColumnPolicyConfig("name").ForTable("user").
			OnInsert().Value(false, false, func(ctx context.Context) any {
			return fmt.Sprintf("user_%09d", rand.Intn(1000000000))
		}),

		// 插入user表时，region字段值从Context中获取。
		orm.NewColumnPolicyConfig("region").ForTable("user").
			OnInsert().Value(true, true, func(ctx context.Context) any {
			if region, ok := ctx.Value("region").(string); ok {
				return region
			}
			return nil
		}),
	},
}.Build()
```

## 内置策略

字段策略配置中以`Use`开头的方法是一些内置的常用策略。

| 方法          | 描述                                                                 |
|---------------|----------------------------------------------------------------------|
| UseCreateTime | 适用于创建时间字段，插入记录时字段值使用`time.Now()`生成，禁止更新。 |
| UseUpdateTime | 适用于更新时间字段，插入和更新记录时字段值使用`time.Now()`生成。     |
| UseRowVersion | 适用于版本号字段，插入记录时值为`1`，更新时递增。                    |

*Example*

```go
db := orm.Config{
	RawDB:  rawDB,
	DBType: orm.DBType_.MySQL,
	ColumnPolicyConfigs: orm.ColumnPolicyConfigs{
		orm.NewColumnPolicyConfig("create_at").UseCreateTime(),
		orm.NewColumnPolicyConfig("update_at").UseUpdateTime(),
		orm.NewColumnPolicyConfig("version").UseRowVersion(),
	},
}.Build()
```

## 软删除模式

软删除模式用于启用高级执行器`DeleteSoftly`。软删除的设计兼容表的唯一约束——使唯一约束在删除记录后失效，实现方案是：表增加删除标记字段，创建唯一索引时与删除标记字段做联合索引。{{<code>}}orm.NewColumnPolicyConfig(column string){{</code>}}的`column`参数为删除标记字段，`OnDeleteSoftly`的事件方法`AssignedPkMode`和`AssignedNullMode`用于指定模式，其中`normalValue`参数指定了正常（未删除）数据的查询条件，两种模式的区别如下。

### AssignedPkMode

该模式删除记录时会将删除标记字段值赋值为主键。要求表必须有且仅有一个主键字段，实体对应字段必须带`pk`[标签](../../entity/tag)，且删除标记字段类型与主键字段类型相同。所有数据库都可使用该模式。

*Example*

```mysql
CREATE TABLE user (
	id BIGINT PRIMARY KEY,
	name VARCHAR(20) NOT NULL,
	deleted BIGINT NOT NULL DEFAULT 0,
	UNIQUE (name, deleted)
);
```

```go
type User struct {
	Id      *int64 `orm:"pk"`
	Name    *string
	Deleted *int64
}

func main() {
	// ...

	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
		ColumnPolicyConfigs: orm.ColumnPolicyConfigs{
			orm.NewColumnPolicyConfig("deleted").OnDeleteSoftly().AssignedPkMode(0),
		},
	}.Build()

	db.FindOne[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL:
	// SELECT id, name, deleted FROM user WHERE id = 1 AND deleted = 0

	db.DeleteSoftly[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL:
	// UPDATE user SET deleted = id WHERE id = 1 AND deleted = 0
}
```

### AssignedNullMode

该模式删除记录时会将删除标记字段值赋值为`null`。要求唯一约束在含有`null`值字段时失效，因此仅适用于部分数据库，如MySQL、PostgreSQL和SQLite。而Oracle和SQL Server的唯一索引非此特性，因此不能使用该模式。可通过下面的SQL测试。

```sql
-- 测试是否可用AssignedNullMode

CREATE TABLE tab (
	col1 VARCHAR(10),
	col2 VARCHAR(10),
	UNIQUE (col1, col2)
);

insert into tab values('a', null);
insert into tab values('a', null); -- 此语句必须成功才能使用AssignedNullMode
```

*Example*

```mysql
CREATE TABLE user (
	id BIGINT PRIMARY KEY,
	name VARCHAR(20) NOT NULL,
	deleted TINYINT(1) DEFAULT 0,
	UNIQUE (name, deleted)
);
```

```go
type User struct {
	Id      *int64
	Name    *string
	Deleted *bool
}

func main() {
	// ...

	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
		ColumnPolicyConfigs: orm.ColumnPolicyConfigs{
			orm.NewColumnPolicyConfig("deleted").OnDeleteSoftly().AssignedNullMode(0),
		},
	}.Build()

	db.FindOne[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL:
	// SELECT id, name, deleted FROM user WHERE id = 1 AND deleted = 0

	db.DeleteSoftly[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", 1)
	}).Do()
	// 执行SQL:
	// UPDATE user SET deleted = NULL WHERE id = 1 AND deleted = 0
}
```