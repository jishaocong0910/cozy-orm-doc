---
title: 事务传播
weight: 2
---

# 事务传播

执行器、事务实例和事务钩子，都会检查`ctx`参数中的值，若存在所属DB实例创建的事务数据库连接（`*sql.Tx`），则表明`ctx`中存在事务，然后加入该事务。若`ctx`中不存在事务，则执行器和事务钩子以非事务执行，事务实例则会创建新的事务数据库连接，然后添加到`ctx`中。

以下示例中，调用`createOrder`时，两个函数会在同一个事务中执行。

```go
// 创建订单
func CreateOrder(ctx context.Context, order *Order, payment *Payment) error {
	return db.Tx(ctx).Do(func(ctx context.Context) error {
		db.Insert[Order](ctx).Must().Entities(order).Do()
		db.Insert[Payment](ctx).Must().Entities(payment).Do()
		err = deductInventory(ctx, order)
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

		db.Update[Inventory](ctx).Must().Set("stock", remainingStock).Cond(func(c *orm.Cond) {
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