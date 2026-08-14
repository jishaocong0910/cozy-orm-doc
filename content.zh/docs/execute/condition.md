---
title: 查询条件
weight: 6
---

# 查询条件

`orm.Cond`提供了各种方法用于拼接条件，支持链式调用。

## 方法概述

| 方法                                         | 条件操作符/表达式/描述                                                              |
|----------------------------------------------|-------------------------------------------------------------------------------------|
| `Eq(column string, arg any)`                 | `=`                                                                                 |
| `Ne(column string, arg any)`                 | `<>`                                                                                |
| `Gt(column string, arg any)`                 | `>`                                                                                 |
| `Lt(column string, arg any)`                 | `<`                                                                                 |
| `Ge(column string, arg any)`                 | `>=`                                                                                |
| `Le(column string, arg any)`                 | `<=`                                                                                |
| `Like(column string, str string)`            | `LIKE '%...%'`                                                                      |
| `LikeLeft(column string, str string)`        | `LIKE '...%'`                                                                       |
| `LikeRight(column string, str string)`       | `LIKE '%...'`                                                                       |
| `LikePattern(column string, pattern string)` | `LIKE '...'`                                                                        |
| `In(column string, args []any)`              | `IN(...)`                                                                           |
| `Between(column string, min, max any)`       | `BETWEEN ... AND ...`                                                               |
| `IsNull(column string)`                      | `IS NULL`                                                                           |
| `IsNotNull(column string)`                   | `IS NOT NULL`                                                                       |
| `Not()`                                      | 下个条件增加`NOT`修饰。                                                             |
| `Or()`                                       | 下个条件使用`OR`连接。                                                              |
| `Sub(func(c *orm.Cond))`                     | 子条件处理函数，使用参数`c`拼接子条件，详见[[子条件]](#子条件)。                    |
| `Expr(func(c *orm.CondExpr))`                | 条件表达式处理函数，使用参数`c`拼接SQL和设置参数，详见[[条件表达式]](#条件表达式)。 |

*Example*

```go
// 查找订单
func FindOrders(ctx context.Context, userId int64, merchantId int64, orderNo *string, status *int8, startAt *time.Time, endAt *time.Time, page int, pageSize int) ([]*Order, error) {
	return db.Find[Order](ctx).Cond(func(c *orm.Cond) {
		c.Eq("user_id", userId).Eq("merchant_id", merchantId)
		if orderNo != nil {
			c.Eq("order_no", orderNo)
		}
		if status != nil {
			c.Eq("status", status)
		}
		if startAt != nil {
			if endAt != nil {
				c.Between("create_at", startAt, endAt)
			} else {
				c.Ge("create_at", startAt)
			}
		} else if endAt != nil {
			c.Le("create_at", endAt)
		}
	}).OrderBy(func(o *orm.OrderBy) {
		o.Asc("id")
	}).Page((page-1)*pageSize, pageSize).Do()
	// orderNo、startAt、endAt参数不为nil时，执行SQL:
	// SELECT id, order_no, user_id, merchant_id, amount, status, create_at
	// FROM order
	// WHERE user_id = $1
	//   AND merchant_id = $2
	//   AND order_no = $3
	//   AND create_at BETWEEN $4 AND $5
	// ORDER BY id ASC
	// LIMIT $6 OFFSET $7
}
```

## 子条件

`Sub`方法用于创建子条件，子条件会自动添加括号对

*Example*

```go
// 获取用户单个订单
func GetOrder(ctx context.Context, userId int64, merchantId int64, orderNo string) ([]*Order, error) {
	return db.Find[Order](ctx).Select("id", "order_no", "user_id", "merchant_id", "amount", "status", "create_at").
		Cond(func(c *orm.Cond) {
			c.Eq("user_id", userId).Eq("merchant_id", merchantId).Sub(func(c *orm.Cond) {
				c.Eq("order_no", orderNo).Or().Eq("trade_no", orderNo)
			})
		}).Do()
	// 执行SQL:
	// SELECT id, order_no, user_id, merchant_id, amount, status, create_at
	// FROM order
	// WHERE user_id = $1
	//   AND merchant_id = $2
	//   AND (order_no = ? OR trade_no = $3)
}
```

## 条件表达式

`Expr`方法用于拼接原生SQL表达式，并且可以设置参数。整个表达式会视为一个子条件，自动添加括号对。

*Example*

```go
// 查找根据日期用户的订单
func FindOrdersByDate(ctx context.Context, userId int64, merchantId int64, date time.Time) ([]*Order, error) {
	return db.Find[Order](ctx).Select("id", "order_no", "user_id", "merchant_id", "amount", "status", "create_at").
		Cond(func(c *orm.Cond) {
			c.Eq("user_id", userId).Eq("merchant_id", merchantId).Expr(func(c *orm.CondExpr) {
				c.Text("create_at = DATE(").Arg(date).Text(")")
			})
		}).OrderBy(func(o *orm.OrderBy) {
		o.Asc("id")
	}).Do()
	// 执行SQL:
	// SELECT id, order_no, user_id, merchant_id, amount, status, create_at
	// FROM order
	// WHERE user_id = $1
    //   AND merchant_id = $2
	//   AND (create_at = DATE($3))
}
```




