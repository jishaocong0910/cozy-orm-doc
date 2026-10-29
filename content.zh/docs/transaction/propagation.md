---
title: 事务传播
weight: 2
---

# 事务传播

执行器、事务实例和事务钩子会检查`ctx`参数中是否存在所属DB实例创建的事务连接（`*sql.Tx`）。若存在，即表明`ctx`中已有事务，执行器、事务钩子、事务实例都会加入它。若不存在，执行器和事务钩子以非事务执行，事务实例则会创建新的事务连接并设置到`ctx`中。

以下示例中，调用`createOrder`时，两个函数会在同一个事务中执行。

```go
// 创建订单
func CreateOrder(ctx context.Context, order *Order, payment *Payment) error {
	return db.Tx(ctx).Do(func(ctx context.Context) error {
		db.Insert[Order](ctx).Must().Entities(order).Do()
		db.Insert[Payment](ctx).Must().Entities(payment).Do()
		err := DeductInventory(ctx, order)
		if err != nil {
			return err
		}
		return nil
	})
}

// 扣减库存
func DeductInventory(ctx context.Context, order *Order) error {
	return db.Tx(ctx).Do(func(ctx context.Context) error {
		inventory, _ := db.FindOne[Inventory](ctx).Must().Cond(func(c *orm.Cond) {
			c.Eq("sku_id", order.SkuId)
		}).LastClause("FOR UPDATE").Do()

		remainingStock := *inventory.Stock - *order.Quantity
		if remainingStock < 0 {
			return errors.New("not enough stock")
		}

		db.Update[Inventory](ctx).Must().Assignment(func(a *orm.Assignment) {
			a.SetValue("stock", remainingStock)
		}).Cond(func(c *orm.Cond) {
			c.Eq("id", inventory.Id)
		}).Do()

		db.Insert[InventoryLog](ctx).Must().Entities(&InventoryLog{
			InventoryId:    inventory.Id,
			OrderId: order.Id,
			Stock:          order.Quantity,
			StockBefore:    inventory.Stock,
			StockAfter:     new(remainingStock),
			Operation:      new("deduct"),
		}).Do()
		return nil
	})
}
```