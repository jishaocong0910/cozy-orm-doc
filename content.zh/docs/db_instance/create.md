---
title: 创建实例
weight: 1
---

# 创建实例

`orm.DB`是一个数据库操作实例，称为DB实例，用于连接数据库并提供数据库操作方法。DB实例通过`orm.Config`配置后调用`Build`方法创建。

*Example*

```go
// open a DB
rawDB, err := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")
if err != nil {
	panic(err)
}

// create a orm.DB instance
db := orm.Config{
	RawDB:  rawDB,
	DBType: orm.DBType_.MySQL,
}.Build()
```