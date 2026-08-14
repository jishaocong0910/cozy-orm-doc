---
title: 简介
weight: 1
---

## 简介

实体（Entity）是CozyORM中用于表示查询结果或持久化数据的结构体。实体的字段对应数据库中的列，使用`nil`映射数据库的`null`值。

*Example*

```go
type User struct {
    Id    *int64
    Name  *string
    Email *string
}
```