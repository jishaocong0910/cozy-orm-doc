---
title: 快速开始
weight: 1
---

# 快速开始

## 安装

```shell
go get github.com/jishaocong0910/cozy-orm
```

## 基础执行器

```go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
	orm "github.com/jishaocong0910/cozy-orm"
)

/*
-- MySQL init script

CREATE TABLE `user` (
	`id` bigint PRIMARY KEY AUTO_INCREMENT,
	`name` varchar(20),
	`email` varchar(20),
	`phone` varchar(20),
	`avatar_url` varchar(1000),
	`address` varchar(255),
	`status` tinyint,
	`level` tinyint,
	`create_at` datetime(3) DEFAULT CURRENT_TIMESTAMP(3)
);
*/

type User struct {
	Id        *int64 `orm:"pk;auto"`
	Name      *string
	Email     *string
	Phone     *string
	AvatarUrl *string
	Address   *string
	Status    *int8
	Level     *int8
	CreateAt  *time.Time
}

func main() {
	// open a DB
	rawDB, err := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}

	// create an orm.DB instance
	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
	}.Build()

	// insert
	u := &User{
		Name:      new("Alice"),
		Email:     new("example@email.com"),
		Phone:     new("123456789"),
		AvatarUrl: new("https://example.com/avatar.jpg"),
		Address:   new("anytown"),
		Status:    new(int8(1)),
		Level:     new(int8(0)),
		CreateAt:  new(time.Now()),
	}
	affected, err := db.Mutation(nil).MapTo[User](u).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("INSERT INTO user(name, email, phone, avatar_url, address, status, level, create_at) VALUES(?, ?, ?, ?, ?, ?, ?, ?)")
		b.Args(u.Name, u.Email, u.Phone, u.AvatarUrl, u.Address, u.Status, u.Level, u.CreateAt)
	}).Do()
	if err != nil {
		panic(err)
	}
	fmt.Printf("affected: %d, id: %d\n", affected, *u.Id) // auto increment key

	// query
	u2, err := db.Query[User](nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("SELECT * FROM user WHERE id = ?").Args(u.Id)
	}).Do()
	if err != nil {
		panic(err)
	}
	j, _ := json.Marshal(u2)
	fmt.Println(string(j))

	// update
	affected, err = db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("UPDATE user SET status = ?, level = ? WHERE id = ?").Args(2, 1, u.Id)
	}).Do()
	if err != nil {
		panic(err)
	}
	fmt.Println("affected:", affected)

	// delete
	affected, err = db.Mutation(nil).BuildSql(func(b *orm.SQLBuilder) {
		b.Text("DELETE FROM user WHERE id = ?").Args(u.Id)
	}).Do()
	if err != nil {
		panic(err)
	}
	fmt.Println("affected:", affected)
}
```

## 高级执行器

```go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
	orm "github.com/jishaocong0910/cozy-orm"
)

/*
-- MySQL init script

CREATE TABLE `user` (
	`id` bigint PRIMARY KEY AUTO_INCREMENT,
	`name` varchar(20),
	`email` varchar(20),
	`phone` varchar(20),
	`avatar_url` varchar(1000),
	`address` varchar(255),
	`status` tinyint,
	`level` tinyint,
	`create_at` datetime(3) DEFAULT CURRENT_TIMESTAMP(3)
);
*/

type User struct {
	Id        *int64 `orm:"pk;auto"`
	Name      *string
	Email     *string
	Phone     *string
	AvatarUrl *string
	Address   *string
	Status    *int8
	Level     *int8
	CreateAt  *time.Time
}

func main() {
	// open a DB
	rawDB, err := sql.Open("mysql", "root:12345678@tcp(127.0.0.1:3306)/test?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}

	// create an orm.DB instance
	db := orm.Config{
		RawDB:  rawDB,
		DBType: orm.DBType_.MySQL,
	}.Build()

	// insert
	u := &User{
		Name:      new("Alice"),
		Email:     new("example@email.com"),
		Phone:     new("123456789"),
		AvatarUrl: new("https://example.com/avatar.jpg"),
		Address:   new("anytown"),
		Status:    new(int8(1)),
		Level:     new(int8(0)),
		CreateAt:  new(time.Now()),
	}
	affected, err := db.Insert[User](nil).Entities(u).Do()
	if err != nil {
		panic(err)
	}
	fmt.Printf("affected: %d, id: %d\n", affected, *u.Id) // auto increment key

	// query
	u2, err := db.FindOne[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", u.Id)
	}).Do()
	if err != nil {
		panic(err)
	}
	j, _ := json.Marshal(u2)
	fmt.Println(string(j))

	// update
	u3 := &User{
		Id:     u.Id,
		Status: new(int8(2)),
		Level:  new(int8(1)),
	}
	affected, err = db.Update[User](nil).Entity(u3).Do()
	if err != nil {
		panic(err)
	}
	fmt.Println("affected:", affected)

	// delete
	affected, err = db.Delete[User](nil).Cond(func(c *orm.Cond) {
		c.Eq("id", u.Id)
	}).Do()
	if err != nil {
		panic(err)
	}
	fmt.Println("affected:", affected)
}
```