---
title: 标签
weight: 3
---

# 标签

标签格式：`orm:"<tag>=<value>"`，多个标签时用`;`分隔。

| 标签   | 描述                                                                                                                                                                               |
|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| table  | 映射的表名，只能用于第一个字段，且为匿名的空结构体，优先级高于DB实例配置的`TabNameMapper`参数。                                                                                    | 
| column | 映射的表字段名，优先级高于DB实例配置的`ColNameMapper`参数。                                                                                                                        |
| pk     | 主键                                                                                                                                                                               |
| auto   | 自动生成字段。[获取生成Key模式](../../db_instance/config/#获取生成key模式)将返回带有该标签的字段。标签值为自动递增步长，默认为`1`，仅对`FirstInsertId`和`LastInsertId`模式有效。 |
| ignore | 忽略该字段。                                                                                                                                                                       |

*Example*

```go
type User struct {
	_        struct{}   `orm:"table=user"`
	Id       *int64     `orm:"column=id;pk;auto"`
	Name     *int64     `orm:"column=name"`
	CreateAt *time.Time `orm:"column=create_at"`
	Extra    any        `orm:"ignore"`
}
```
