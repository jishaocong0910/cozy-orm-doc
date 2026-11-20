---
title: 事务钩子
weight: 3
---

# 事务钩子

事务钩子是在事务实例生命周期的特定节点调用自定义函数的功能。通过 `orm.DB.TxHook(ctx context.Context) (*orm.txHook, bool)` 注册事务钩子，当 `ctx` 存在事务才会注册成功，返回 `*orm.txHook` 和 true，否则返回 `nil` 和 false。

## 钩子节点

`orm.txHook` 是一个事务钩子实例，提供了以下方法用于绑定对应节点的自定义函数。

### BeforeCommit

在事务提交前执行。

方法参数：`(handler func(ctx context.Context) error)`。`handler` 函数的返回值 `error` 会被事务实例返回，`handler` 的 `ctx` 参数为事务钩子实例的 `ctx`，即可以在相同事务下执行。

*Example*

```go
// 导入文件的数据
func ImportFile(ctx context.Context, fileUrl string) error {
	// 若ctx存在事务，则在事务提交之前插入导入文件任务记录（后续再处理）。
	if h, ok := db.TxHook(ctx); ok {
		// hookCtx等于ctx，因此会加入已存在的事务。
		h.BeforeCommit(func(hookCtx context.Context) error {
			contentType, _ := ctx.Value("Content-Type").(string)
			db.Insert[ImportFileTask](hookCtx).Entities(&ImportFileTask{
				FileUrl:     fileUrl,
				ContentType: contentType,
			}).Do()
			return nil
		})
	} else {
		// 若ctx不存在事务，则直接导入数据。
		doImportFile(ctx, fileUrl)
	}
	return nil
}
```

### AfterComplete

在事务完成后执行。

方法参数：`(handler func(ctx context.Context, commit bool), copyCtxKeys ...any)`。`handler` 的 `ctx` 参数基于 `context.Background()` 创建，`commit` 参数在事务提交成功时为 true、回滚时为 false。若 `handler` 发生 panic，会被恢复并打印错误日志，原事务依然是成功的，不会返回错误。`copyCtxKeys` 用于指定需要从事务钩子的 `ctx` 复制到 `handler` 的 `ctx` 中的 key。

```go
// 导入文件的数据
func ImportFile(ctx context.Context, fileUrl string) error {
	// 若ctx存在事务，则在事务提交成功后，导入文件。
	if h, ok := db.TxHook(ctx); ok {
		// hookCtx不等于ctx，不会加入原的事务。
		h.AfterComplete(func(hookCtx context.Context, commit bool) {
			if commit {
 				doImportFile(ctx, fileUrl)
			}
		}, "Content-Type") // 复制ctx中指定key
	} else {
		// 若ctx不存在事务，则直接导入文件。
		doImportFile(ctx, fileUrl)
	}
	return nil
}
```

## 限制

事务实例执行钩子前会关闭钩子的注册，因此在钩子中注册钩子将会失败。

*Example*

```go
func main() {
	db.Tx(nil).Do(func(ctx context.Context) error {
		registerHook1(ctx)
		return nil
	})
}

func registerHook1(ctx context.Context) {
	h, ok := db.TxHook(ctx)
	fmt.Println(ok) // true

	// 执行BeforeCommit时已关闭注册钩子，因此registerHook2、registerHook3都注册失败
	h.BeforeCommit(func(ctx context.Context) error {
		registerHook2(ctx)
		registerHook3(ctx)
		return nil
	})
}

func registerHook2(ctx context.Context) {
	_, ok := db.TxHook(ctx)
	fmt.Println(ok) // false
}

func registerHook3(ctx context.Context) {
	_, ok := db.TxHook(ctx)
	fmt.Println(ok) // false
}
```