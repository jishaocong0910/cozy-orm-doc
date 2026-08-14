---
title: 事务钩子
weight: 3
---

# 事务钩子

事务钩子是在事务实例生命周期的特定节点调用自定义函数的功能。通过`orm.DB.TxHook(ctx context.Context) (*orm.txHook, bool)`创建事务钩子，当`ctx`存在事务才会创建成功，返回`*orm.txHook`和true，否则返回返回`nil`和false。`*orm.txHook`是一个事务钩子实例，提供了以下方法用于绑定自定义函数。

## BeforeCommitSync

在事务提交前同步执行，遵循事务传播机制。

方法参数：`(handler func(ctx context.Context) error)`。`handler`函数的返回值`error`会被事务实例返回，`ctx`参数等于事务钩子实例的`ctx`参数。

*Example*

```go
// 保存操作日志
func SaveOperationLog(ctx context.Context, log *OperationLog) error {
	// 若ctx存在事务，则在事务提交之前插入操作日志，hookCtx等于ctx，因此会加入已存在的事务。
	if h, ok := db.TxHook(ctx); ok {
		h.BeforeCommitSync(func(hookCtx context.Context) error {
			_, err := db.Insert[OperationLog](hookCtx).Must().Entities(log).Do()
			return err
		})
	} else {
		// 若ctx不存在事务，则直接保存操作日志。
		_, err := db.Insert[OperationLog](ctx).Entities(log).Do()
		if err != nil {
			return err
		}
	}
	return nil
}
```

## AfterCompleteAsync

在事务完成后异步执行。

方法参数：`(handler func(ctx context.Context, commit bool), copyCtxKeys ...any)`。`handler`函数的`ctx`参数以`context.Background()`为基础创建，`commit`参数在事务提交成功时为true，回滚时为false。若`handler`发生`panic`，会被恢复并打印错误日志。`copyCtxKeys`参数用于指定需要从事务钩子实例的`ctx`复制到`handler`的`ctx`的值对应的key。

```go
// 导入文件的数据
func ImportFile(ctx context.Context, fileUrl string) error {
	// 若ctx存在事务，则在事务提交成功后，导入Excel数据。
	if h, ok := db.TxHook(ctx); ok {
		h.AfterCompleteAsync(func(hookCtx context.Context, commit bool) {
			if commit {
				doImportExcelFile(ctx, fileUrl)
			}
		}, "Content-Type") // 复制ctx中指定key的value值
	} else {
		// 若ctx不存在事务，则直接导入数据。
		doImportExcelFile(ctx, fileUrl)
	}
	return nil
}
```