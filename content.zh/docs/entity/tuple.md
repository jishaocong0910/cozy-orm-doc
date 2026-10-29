---
title: 预定义实体
weight: 4
---

# 预定义实体

预定义实体是用于动态映射查询结果的类型，方便查询少量字段或非表字段（例如聚合函数）等业务，只能用于基础执行器`Query`。

* `orm.Tuple[T]`
* `orm.Tuple2[T1, T2]`
* `orm.Tuple3[T1, T2, T3]`
* `orm.Tuple4[T1, T2, T3, T4]`
* `orm.Tuple5[T1, T2, T3, T4, T5]`
* `orm.Tuple6[T1, T2, T3, T4, T5, T6]`

泛型的数量代表可映射字段的最大数量，且须满足[字段类型](../field_type)要求，为了方便，允许指针写成值类型。

*Example*

```go
func main() {
	// ...

	emails, _ := db.Query[orm.Tuple2[int64, *string](nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("SELECT id, email FROM user WHERE id IN(1, 2, 3)")
	}).Do()

	for _, t := range emails {
		fmt.Println(t.Field1, t.Field2)
	}

	orders, _ := db.Query[orm.Tuple3[int64, int64, int64](nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("SELECT user_id, COUNT(*), SUM(order_amount) FROM order " +
			"WHERE status = 'paid' AND user_id IN(1, 2, 3) GROUP BY user_id")
	}).Do()

	for _, t := range orders {
		fmt.Println(t.Field1, t.Field2, t.Field3)
	}
}
```