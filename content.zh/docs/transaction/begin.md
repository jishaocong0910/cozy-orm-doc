---
title: 开始事务
weight: 1
---

# 开始事务

`orm.Tx`是一个事务实例，用于管理事务的生命周期。通过`orm.DB.Tx(ctx context.Context)`创建事务实例，`ctx`参数为`nil`时将使用`context.Background()`作为默认值。`orm.Tx`具有以下方法并支持链式调用。

| 方法                                                               | 描述                                                                                                                                                                                        |
|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Must()`                                                           | 默认会将错误返回，此参数会直接panic。                                                                                                                                                       |
| `TxOptions(*sql.TxOptions)`                                        | 设置事务选项。                                                                                                                                                                              |
| {{<code>}}Do(func(ctx&nbsp;context.Context)&nbsp;error){{</code>}} | 业务处理函数，用于执行事务中的数据库操作，若发生`panic`，事务实例会将其恢复并返回error。将业务处理函数的`ctx`参数传递给执行器，即可使执行器在事务中执行，详见[[事务传播]](../propagation)。 |

*Example*

```go
// 创建订单
func CreateOrder(normalCtx context.Context, order *Order, payment *Payment) error {
	return db.Tx(normalCtx).Do(func(ctx context.Context) error {
		// 业务处理函数的ctx参数必须传递给执行器，才能使其在事务中执行。
		_, err := db.Insert[Order](ctx).Entities(order).Do()
		if err != nil {
			return err
		}

		// 这里执行器设置了Must参数，产生error会被panic，orm.Tx实例会recover并返回error
		db.Insert[Payment](ctx).Must().Entities(payment).Do()

		/* 以下代码，是导致执行器不在事务中执行的错误示范 */

		// 没有传入ctx参数
		// _, err := db.Insert[Order](nil).Entities(order).Do()
		// if err != nil {
		//     return err
		// }

		// 传错ctx参数
		// _, err = db.Insert[TradeOrder](normalCtx).Entities(tradeOrder).Do()
		// if err != nil {
		//     return err
		// }
		return nil
	})
}
```