---
title: 字段类型
weight: 2
---

# 字段类型

实体字段不支持嵌套结构体和非导出的字段，这些字段会被忽略。使用`nil`映射数据库的`null`值，规定字段种类必须为 **Pointer**、**Slice**和**Map**。

## Go内置类型

支持以下Go内置类型，以及它们的类型定义和别名。

| 种类    | 内置类型                                                                                                                                                                                 |
|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Pointer | `int`、`int8`、`int16`、`int32`、`int64`、`uint`、`uint8`、`uint16`、`uint32`、`uint64`、`float32`、`float64`、`bool`、`string`、`[N]byte`（数组）、`time.Time`、`uuid.UUID`(Go1.27新增) |
| Slice   | `byte`                                                                                                                                                                                   |
| Map     | （只支持自定义类型）                                                                                                                                                                     |

*Example*

```go
type MyInt64 int64

type MyString = string

type MyStringPtr *string

type User struct {
    Id    *MyInt64
    Name  *MyString
	Email MyStringPtr
}
```

## 自定义类型

### Valuer&Scanner

CozyORM支持Go内置的`driver.Valuer`和`sql.Scanner`接口，实现这些接口的类型将被CozyORM支持，因此支持一些第三方类型，例如
`github.com/shopspring/decimal.Decimal`。

*Example*

```go
import (
    "github.com/shopspring/decimal"
)

type Account struct {
    Id      *int64
    UserId  *int64
    Balance *decimal.Decimal
}
```

### Converter

CozyORM拥有独立的自定义类型机制：实现`orm.Converter[T]`接口。泛型`T`是一个中间类型，用于在自定义类型值和数据库字段值之间进行转换。`T`必须是支持的Go内置类型，为了方便，允许指针写成值类型。

| 方法         | 描述                                                                                                                                                                                                                     |
|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ToArg() T`  | 自定义类型的值作为SQL参数值时，会调用此方法进行转换。若值为`nil`则不会被调用。                                                                                                                                           |
| `ToField(T)` | 查询时先将数据库字段值转换为间接类型，再调用此方法转换为自定义类型。若数据库字段值为`null`则不会被调用。**该方法的接受者必须是指针**，被调用时，接受者指向其底层类型的零值，转换过程是将间接类型值转化为接受者指向的值。 |

> [!WARNING]
>
> 如果`orm.Converter.ToField(T)`方法的接受者底层类型是`map`，由于它是零值，不能使用`map[key]=value`设置值，必须先用`make`初始化。

*Example*

```go
import (
  "encoding/json"
  "strings"
)

// 实体
type Product struct {
	Id          *int64
	Name        *string
	Description *Description
    Tags        Tags
	SkuMap      SkuMap
}

// 自定义类型
type Description struct {
	Title  string   `json:"title"`
	Body   string   `json:"body"`
}

// 此方法接受者可以是值或指针
func (d *Description) ToArg() string {
    // 只有值不为nil才会调用此方法，因此d一定不为nil
	b, _ := json.Marshal(d)
	return string(b)
}

// 此方法接受者必须是指针
func (d *Description) ToField(src string) {
	json.Unmarshal([]byte(src), d)
}

// 自定义类型
type Tags []string

func (p Tags) ToArg() string {
    return strings.Join(p, ",")
}

func (p *Tags) ToField(src string) {
    // 数据库字段值不为null时该方法才会被调用，因此src一定不为nil
    *p = strings.Split(src, ",")
}

// 自定义类型
type SkuMap map[string]string

func (s SkuMap) ToArg() *string {
	if b, err := json.Marshal(s); err == nil {
		return new(string(b))
	}
	return nil
}

func (s *SkuMap) ToField(src *string) {
	json.Unmarshal([]byte(*src), s)
}
```