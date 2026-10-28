---
title: 日志记录器
weight: 1
---

# 日志记录器

实现`orm.Logger`并配置`orm.Config`的`Logger`和`SqlLogLevel`参数即可打印日志。

*Example*

```go
func main() {
	// ...

	db = orm.Config{
		RawDB:       rawDB,
		DBType:      orm.DBType_.MySQL,
		Logger:      myLogger{},
		SqlLogLevel: orm.LogLevel_.Info,
	}.Build()

	ctx := context.WithValue(context.Background(), "operator", "admin1013")

	db.Mutation(ctx).Must().Description("update product").BuildSql(func(b *orm.SQLBuilder) {
		b.Text("Update product SET status = ").Arg("draft").Text(" WHERE id = ").Arg(1)
	}).Do()
	// 打印日志:
	// level: info; operator: admin1013; desc: update product; SQL: Update product SET status = ? WHERE id = ?; args: draft(string) 1(int); affected: 1; cost: 20ms
}

type myLogger struct {
}

func (l myLogger) Debug(ctx context.Context, msg string) {
	opr, _ := ctx.Value("operator").(string)
	fmt.Printf("level: debug; operator: %s; %s\n", opr, msg)
}

func (l myLogger) Info(ctx context.Context, msg string) {
	opr, _ := ctx.Value("operator").(string)
	fmt.Printf("level: info; operator: %s; %s\n", opr, msg)
}

func (l myLogger) Warn(ctx context.Context, msg string) {
	opr, _ := ctx.Value("operator").(string)
	fmt.Printf("level: warn; operator: %s; %s\n", opr, msg)
}

func (l myLogger) Error(ctx context.Context, msg string) {
	opr, _ := ctx.Value("operator").(string)
	fmt.Printf("level: error; operator: %s; %s\n", opr, msg)
}

```
