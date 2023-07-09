---
title: 在 Go 中使用 sqlx 替代 database/sql 操作数据库
date: 2023-06-15 22:39:37
tags:
- Go
- SQL
categories:
- Go
---

[sqlx](github.com/jmoiron/sqlx) 是 Go 语言中一个流行的第三方包，它提供了对 Go 标准库 `database/sql` 的扩展，旨在简化和改进 Go 语言中使用 SQL 的体验，并提供了更加强大的数据库交互功能。`sqlx` 保留了 `database/sql` 接口不变，是 `database/sql` 的超集，这使得将现有项目中使用的 `database/sql` 替换为 `sqlx` 变得相当轻松。

<!-- more -->

本文重点讲解 `sqlx` 在 `database/sql` 基础上扩展的功能，对于 `database/sql` 已经支持的功能则不会详细讲解。如果你对 `database/sql` 不熟悉，可以查看我的另一篇文章[《在 Go 中如何使用 database/sql 来操作数据库》](https://jianghushinian.cn/2023/06/05/how-to-use-database-sql-to-operate-database-in-go/)。

### 安装

`sqlx` 安装方式同 Go 语言中其他第三方包一样：

```bash
$ go get github.com/jmoiron/sqlx
```

### sqlx 类型设计

`sqlx` 的设计与 `database/sql` 差别不大，编码风格较为统一，参考 `database/sql` 标准库，`sqlx` 提供了如下几种与之对应的数据类型：

- `sqlx.DB`：类似于 `sql.DB`，表示数据库对象，可以用来操作数据库。
- `sqlx.Tx`：类似于 `sql.Tx`，事务对象。
- `sqlx.Stmt`：类似于 `sql.Stmt`，预处理 SQL 语句。
- `sqlx.NamedStmt`：对 `sqlx.Stmt` 的封装，支持具名参数。
- `sqlx.Rows`：类似于 `sql.Rows`，`sqlx.Queryx` 的返回结果。
- `sqlx.Row`：类似于 `sql.Row`，`sqlx.QueryRowx` 的返回结果。

以上类型与 `database/sql` 提供的对应类型在功能上区别不大，但 `sqlx` 为这些类型提供了更友好的方法。

### 准备

为了演示 `sqlx` 用法，我准备了如下 MySQL 数据库表：

```sql
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) DEFAULT NULL COMMENT '用户名',
  `email` varchar(255) NOT NULL DEFAULT '' COMMENT '邮箱',
  `age` tinyint(4) NOT NULL DEFAULT '0' COMMENT '年龄',
  `birthday` datetime DEFAULT NULL COMMENT '生日',
  `salary` varchar(128) DEFAULT NULL COMMENT '薪水',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `u_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

你可以使用 MySQL 命令行或图形化工具创建这张表。

### 连接数据库

使用 `sqlx` 连接数据库：

```go
package main

import (
	"database/sql"
	"log"

	_ "github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

func main() {
	var (
		db  *sqlx.DB
		err error
		dsn = "user:password@tcp(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=true&loc=Local"
	)

	// 1. 使用 sqlx.Open 连接数据库
	db, err = sqlx.Open("mysql", dsn)
	if err != nil {
		log.Fatal(err)
	}

	// 2. 使用 sqlx.Open 变体方法 sqlx.MustOpen 连接数据库，如果出现错误直接 panic
	db = sqlx.MustOpen("mysql", dsn)

	// 3. 如果已经有了 *sql.DB 对象，则可以使用 sqlx.NewDb 连接数据库，得到 *sqlx.DB 对象
	sqlDB, err := sql.Open("mysql", dsn)
	if err != nil {
		log.Fatal(err)
	}
	db = sqlx.NewDb(sqlDB, "mysql")

	// 4. 使用 sqlx.Connect 连接数据库，等价于 sqlx.Open + db.Ping
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		log.Fatal(err)
	}

	// 5. 使用 sqlx.Connect 变体方法 sqlx.MustConnect 连接数据库，如果出现错误直接 panic
	db = sqlx.MustConnect("mysql", dsn)
}
```

在 `sqlx` 中我们可以通过以上 5 种方式连接数据库。

`sqlx.Open` 对标 `sql.Open` 方法，返回 `*sqlx.DB` 类型。

`sqlx.MustOpen` 与 `sqlx.Open` 一样会返回 `*sqlx.DB` 实例，但如果遇到错误则会 `panic`。

`sqlx.NewDb` 支持从一个 `database/sql` 包的 `*sql.DB` 对象创建一个新的 `*sqlx.DB` 类型，并且需要指定驱动名称。

使用前 3 种方式连接数据库并不会立即与数据库建立连接，连接将会在合适的时候延迟建立。为了确保能够正常连接数据库，往往需要调用 `db.Ping()` 方法进行验证：

```go
ctx := context.Background()
if err := db.PingContext(ctx); err != nil {
	log.Fatal(err)
}
```

`sqlx` 提供的 `sqlx.Connect` 方法就是用来简化这一操作的，它等价于 `sqlx.Open` + `db.Ping` 两个方法，其定义如下：

```go
func Connect(driverName, dataSourceName string) (*DB, error) {
	db, err := Open(driverName, dataSourceName)
	if err != nil {
		return nil, err
	}
	err = db.Ping()
	if err != nil {
		db.Close()
		return nil, err
	}
	return db, nil
}
```

`sqlx.MustConnect` 方法在 `sqlx.Connect` 方法的基础上，提供了遇到错误立即 `panic` 的功能。看到 `sqlx.MustConnect` 方法的定义你就明白了：

```go
func MustConnect(driverName, dataSourceName string) *DB {
	db, err := Connect(driverName, dataSourceName)
	if err != nil {
		panic(err)
	}
	return db
}
```

以后当你遇见 `MustXxx` 类似方法名时就应该想到，其功能往往等价于 `Xxx` 方法，不过在其内部实现中，遇到 `error` 不再返回，而是直接进行 `panic`，这也是 Go 语言很多库中的惯用方法。

### 声明模型

我们定义一个 `User` 结构体来映射数据库中的 `user` 表：

```go
type User struct {
	ID       int
	Name     sql.NullString `json:"username"`
	Email    string
	Age      int
	Birthday time.Time
	Salary   Salary
	CreatedAt time.Time `db:"created_at"`
	UpdatedAt time.Time `db:"updated_at"`
}

type Salary struct {
	Month int `json:"month"`
	Year  int `json:"year"`
}

// Scan implements sql.Scanner, use custom types in *sql.Rows.Scan
func (s *Salary) Scan(src any) error {
	if src == nil {
		return nil
	}

	var buf []byte
	switch v := src.(type) {
	case []byte:
		buf = v
	case string:
		buf = []byte(v)
	default:
		return fmt.Errorf("invalid type: %T", src)
	}

	err := json.Unmarshal(buf, s)
	return err
}

// Value implements driver.Valuer, use custom types in Query/QueryRow/Exec
func (s Salary) Value() (driver.Value, error) {
	v, err := json.Marshal(s)
	return string(v), err
}
```

`User` 结构体在这里可以被称为「模型」。

### 执行 SQL 命令

`database/sql` 包提供了 `*sql.DB.Exec` 方法来执行一条 SQL 命令，`sqlx` 对其进行了扩展，提供了 `*sqlx.DB.MustExec` 方法来执行一条 SQL 命令：

```go
func MustCreateUser(db *sqlx.DB) (int64, error) {
	birthday := time.Date(2000, 1, 1, 0, 0, 0, 0, time.Local)
	user := User{
		Name:     sql.NullString{String: "jianghushinian", Valid: true},
		Email:    "jianghushinian007@outlook.com",
		Age:      10,
		Birthday: birthday,
		Salary: Salary{
			Month: 100000,
			Year:  10000000,
		},
	}

	res := db.MustExec(
		`INSERT INTO user(name, email, age, birthday, salary) VALUES(?, ?, ?, ?, ?)`,
		user.Name, user.Email, user.Age, user.Birthday, user.Salary,
	)
	return res.LastInsertId()
}
```

这里使用 `*sqlx.DB.MustExec` 方法插入了一条 `user` 记录。

`*sqlx.DB.MustExec` 方法定义如下：

```go
func (db *DB) MustExec(query string, args ...interface{}) sql.Result {
	return MustExec(db, query, args...)
}

func MustExec(e Execer, query string, args ...interface{}) sql.Result {
	res, err := e.Exec(query, args...)
	if err != nil {
		panic(err)
	}
	return res
}
```

与前文介绍的 `sqlx.MustOpen` 方法一样，`*sqlx.DB.MustExec` 方法也会在遇到错误时直接 `panic`，其内部调用的是 `*sqlx.DB.Exec` 方法。

### 执行 SQL 查询

`database/sql` 包提供了 `*sql.DB.Query` 和 `*sql.DB.QueryRow` 两个查询方法，其签名如下：

```go
func (db *DB) Query(query string, args ...any) (*Rows, error)
func (db *DB) QueryRow(query string, args ...any) *Row
```

`sqlx` 在这两个方法的基础上，扩展出如下两个方法：

```go
func (db *DB) Queryx(query string, args ...interface{}) (*Rows, error)
func (db *DB) QueryRowx(query string, args ...interface{}) *Row
```

这两个方法返回的类型正是前文 [sqlx 类型设计](#sqlx-类型设计) 中提到的 `sqlx.Rows`、`sqlx.Row` 类型。

下面来讲解下这两个方法如何使用。

#### Queryx

使用 `*sqlx.DB.Queryx` 方法查询记录如下：

```go
func QueryxUsers(db *sqlx.DB) ([]User, error) {
	var us []User
	rows, err := db.Queryx("SELECT * FROM user")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	for rows.Next() {
		var u User
		// sqlx 提供了便捷方法可以将查询结果直接扫描到结构体
		err = rows.StructScan(&u)
		if err != nil {
			return nil, err
		}
		us = append(us, u)
	}
	return us, nil
}
```

`*sqlx.DB.Queryx` 方法签名虽然与 `*sql.DB.Query` 方法基本相同，但它返回类型 `*sqlx.Rows` 得到了扩展，其提供的 `StructScan` 方法能够方便的将查询结果直接扫描到 `User` 结构体，这极大的增加了便携性，我们再也不用像使用 `*sql.Rows` 提供的 `Scan` 方法那样挨个写出 `User` 的属性了。

#### QueryRowx

使用 `*sqlx.DB.QueryRowx` 方法查询记录如下：

```go
func QueryRowxUser(db *sqlx.DB, id int) (User, error) {
	var u User
	err := db.QueryRowx("SELECT * FROM user WHERE id = ?", id).StructScan(&u)
	return u, err
}
```

`*sqlx.Row` 同样提供了 `StructScan` 方法将查询结果扫描到结构体。

另外，这里使用了链式调用的方式，在调用 `db.QueryRowx()` 之后直接调用了 `.StructScan(&u)`，接收的 `err` 是 `StructScan` 的返回结果。这是因为 `db.QueryRowx()` 的返回结果 `*sqlx.Row` 中记录了错误信息 `err`，如果查询阶段遇到错误会被记录到 `*sqlx.Row.err` 中。在调用 `StructScan` 方法阶段，其内部首先判断 `r.err != nil`，如果存在 `err` 直接返回错误，没有错误则将查询结果扫描到 `dest` 参数接收到的结构体指针，代码实现如下：

```go
type Row struct {
	err    error
	unsafe bool
	rows   *sql.Rows
	Mapper *reflectx.Mapper
}

func (r *Row) StructScan(dest interface{}) error {
	return r.scanAny(dest, true)
}

func (r *Row) scanAny(dest interface{}, structOnly bool) error {
	if r.err != nil {
		return r.err
	}
	...
}
```

`sqlx` 不仅扩展了 `*sql.DB.Query` 和 `*sql.DB.QueryRow` 两个查询方法，它还新增了两个查询方法：

```go
func (db *DB) Get(dest interface{}, query string, args ...interface{}) error
func (db *DB) Select(dest interface{}, query string, args ...interface{}) error 
```

`*sqlx.DB.Get` 方法包装了 `*sqlx.DB.QueryRowx` 方法，用以简化查询单条记录。

`*sqlx.DB.Select` 方法包装了 `*sqlx.DB.Queryx` 方法，用以简化查询多条记录。

接下来讲解这两个方法如何使用。

#### Get

使用 `*sqlx.DB.Get` 方法查询记录如下：

```go
func GetUser(db *sqlx.DB, id int) (User, error) {
	var u User
	// 查询记录扫描数据到 struct
	err := db.Get(&u, "SELECT * FROM user WHERE id = ?", id)
	return u, err
}
```

可以发现 `*sqlx.DB.Get` 方法用起来非常简单，我们不再需要调用 `StructScan` 方法将查询结果扫描到结构体中，只需要将结构体指针当作 `Get` 方法的第一个参数传递进去即可。

其代码实现如下：

```go
func (db *DB) Get(dest interface{}, query string, args ...interface{}) error {
	return Get(db, dest, query, args...)
}

func Get(q Queryer, dest interface{}, query string, args ...interface{}) error {
	r := q.QueryRowx(query, args...)
	return r.scanAny(dest, false)
}
```

根据源码可以看出，`*sqlx.DB.Get` 内部调用了 `*sqlx.DB.QueryRowx` 方法。

#### Select

使用 `*sqlx.DB.Select` 方法查询记录如下：

```go
func SelectUsers(db *sqlx.DB) ([]User, error) {
	var us []User
	// 查询记录扫描数据到 slice
	err := db.Select(&us, "SELECT * FROM user")
	return us, err
}
```

可以发现 `*sqlx.DB.Select` 方法用起来同样非常简单，它可以直接将查询结果扫描到 `[]User` 切片中。

其代码实现如下：

```go
func (db *DB) Select(dest interface{}, query string, args ...interface{}) error {
	return Select(db, dest, query, args...)
}

func Select(q Queryer, dest interface{}, query string, args ...interface{}) error {
	rows, err := q.Queryx(query, args...)
	if err != nil {
		return err
	}
	// if something happens here, we want to make sure the rows are Closed
	defer rows.Close()
	return scanAll(rows, dest, false)
}
```

根据源码可以看出，`*sqlx.DB.Select` 内部调用了 `*sqlx.DB.Queryx` 方法。

#### sqlx.In

在 `database/sql` 中如果想要执行 SQL IN 查询，由于 IN 查询参数长度不固定，我们不得不使用 `fmt.Sprintf` 来动态拼接 SQL 语句，以保证 SQL 中参数占位符的个数是正确的。

`sqlx` 提供了 `In` 方法来支持 SQL IN 查询，这极大的简化了代码，也使得代码更易维护和安全。

使用示例如下：

```go
func SqlxIn(db *sqlx.DB, ids []int64) ([]User, error) {
	query, args, err := sqlx.In("SELECT * FROM user WHERE id IN (?)", ids)
	if err != nil {
		return nil, err
	}

	query = db.Rebind(query)
	rows, err := db.Query(query, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var us []User
	for rows.Next() {
		var user User
		err = rows.Scan(&user.ID, &user.Name, &user.Email, &user.Age,
			&user.Birthday, &user.Salary, &user.CreatedAt, &user.UpdatedAt)
		if err != nil {
			return nil, err
		}
		us = append(us, user)
	}
	return us, nil
}
```

调用 `sqlx.In` 并传递 SQL 语句以及切片类型的参数，它将返回新的查询 SQL `query` 以及参数 `args`，这个 `query` 将会根据 `ids` 来动态调整。

比如我们传递 `ids` 为 `[]int64{1, 2, 3}`，则得到 `query` 为 `SELECT * FROM user WHERE id IN (?, ?, ?)`。

注意，我们接下来又调用 `db.Rebind(query)` 重新绑定了 `query` 变量的参数占位符。如果你使用 MySQL 数据库，这不是必须的，因为我们使用的 MySQL 驱动程序参数占位符就是 `?`。而如果你使用 PostgreSQL 数据库，由于 PostgreSQL 驱动程序参数占位符是 `$n`，这时就必须要调用 `db.Rebind(query)` 方法来转换参数占位符了。

它会将 `SELECT * FROM user WHERE id IN (?, ?, ?)` 中的参数占位符转换为 PostgreSQL 驱动程序能够识别的参数占位符 `SELECT * FROM user WHERE id IN ($1, $2, $3)`。

之后的代码就跟使用 `database/sql` 查询记录没什么两样了。

### 使用具名参数

`sqlx` 提供了两个方法 `NamedExec`、`NamedQuery`，它们能够支持具名参数 `:name`，这样就不必再使用 `?` 这种占位符的形式了。

这两个方法签名如下：

```go
func (db *DB) NamedExec(query string, arg interface{}) (sql.Result, error)
func (db *DB) NamedQuery(query string, arg interface{}) (*Rows, error)
```

其使用示例如下：

```go
func NamedExec(db *sqlx.DB) error {
	m := map[string]interface{}{
		"email": "jianghushinian007@outlook.com",
		"age":   18,
	}
	result, err := db.NamedExec(`UPDATE user SET age = :age WHERE email = :email`, m)
	if err != nil {
		return err
	}
	fmt.Println(result.RowsAffected())
	return nil
}

func NamedQuery(db *sqlx.DB) ([]User, error) {
	u := User{
		Email: "jianghushinian007@outlook.com",
		Age:   18,
	}
	rows, err := db.NamedQuery("SELECT * FROM user WHERE email = :email OR age = :age", u)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var users []User
	for rows.Next() {
		var user User
		err := rows.StructScan(&user)
		if err != nil {
			return nil, err
		}
		users = append(users, user)
	}
	return users, nil
}
```

我们可以使用 `:name` 的方式来命名参数，它能够匹配 `map` 或 `struct` 对应字段的参数值，这样的 SQL 语句可读性更强。

### 事务

在事务的支持上，`sqlx` 扩展出了 `Must` 版本的事务，使用示例如下：

```go
func MustTransaction(db *sqlx.DB) error {
	tx := db.MustBegin()
	tx.MustExec("UPDATE user SET age = 25 WHERE id = ?", 1)
	return tx.Commit()
}
```

不过这种用法不多，你知道就行。以下是事务的推荐用法：

```go

func Transaction(db *sqlx.DB, id int64, name string) error {
	tx, err := db.Begin()
	if err != nil {
		return err
	}
	defer tx.Rollback()

	res, err := tx.Exec("UPDATE user SET name = ? WHERE id = ?", name, id)
	if err != nil {
		return err
	}

	rowsAffected, err := res.RowsAffected()
	if err != nil {
		return err
	}
	fmt.Printf("rowsAffected: %d\n", rowsAffected)

	return tx.Commit()
}
```

我们使用 `defer` 语句来处理事务的回滚操作，这样就不必在每次处理错误时重复的编写调用 `tx.Rollback()` 的代码。

如果代码正常执行到最后，通过 `tx.Commit()` 来提交事务，此时即使再调用 `tx.Rollback()` 也不会对结果产生影响。

### 预处理语句

`sqlx` 针对 `*sql.DB.Prepare` 扩展出了 `*sqlx.DB.Preparex` 方法，返回 `*sqlx.Stmt` 类型。

`*sqlx.Stmt` 类型支持 `Queryx`、`QueryRowx`、`Get`、`Select` 这些 `sqlx` 特有的方法。

其使用示例如下：

```go
func PreparexGetUser(db *sqlx.DB) (User, error) {
	stmt, err := db.Preparex(`SELECT * FROM user WHERE id = ?`)
	if err != nil {
		return User{}, err
	}

	var u User
	err = stmt.Get(&u, 1)
	return u, err
}
```

`*sqlx.DB.Preparex` 方法定义如下：

```go
func Preparex(p Preparer, query string) (*Stmt, error) {
	s, err := p.Prepare(query)
	if err != nil {
		return nil, err
	}
	return &Stmt{Stmt: s, unsafe: isUnsafe(p), Mapper: mapperFor(p)}, err
}
```

实际上 `*sqlx.DB.Preparex` 内部还是调用的 `*sql.DB.Preapre` 方法，只不过将其返回结果构造成 `*sqlx.Stmt` 类型并返回。

### 不安全的扫描

在使用 `*sqlx.DB.Get` 等方法查询记录时，如果 SQL 语句查询出来的字段与要绑定的模型属性不匹配，则会报错。

示例如下：

```go
func GetUser(db *sqlx.DB) (User, error) {
	var user struct {
		ID    int
		Name  string
		Email string
		// 没有 Age 属性
	}
	err := db.Get(&user, "SELECT id, name, email, age FROM user WHERE id = ?", 1)
	if err != nil {
		return User{}, err
	}
	return User{
		ID:    user.ID,
		Name:  sql.NullString{String: user.Name},
		Email: user.Email,
	}, nil
}
```

以上示例代码中，SQL 语句中查询了 `id`、`name`、`email`、`age` 4 个字段，而 `user` 结构体则只有 `ID`、`Name`、`Email` 3 个属性，由于无法一一对应，执行以上代码，我们将得到如下报错信息：

```go
missing destination name age in *struct { ID int; Name string; Email string }
```

这种表现是合理的，符合 Go 语言的编程风格，尽早暴露错误有助于减少代码存在 BUG 的隐患。

不过，有些时候，我们就是为了方便想要让上面的示例代码能够运行，可以这样做：

```go
func UnsafeGetUser(db *sqlx.DB) (User, error) {
	var user struct {
		ID    int
		Name  string
		Email string
		// 没有 Age 属性
	}
	udb := db.Unsafe()
	err := udb.Get(&user, "SELECT id, name, email, age FROM user WHERE id = ?", 1)
	if err != nil {
		return User{}, err
	}
	return User{
		ID:    user.ID,
		Name:  sql.NullString{String: user.Name},
		Email: user.Email,
	}, nil
}
```

这里我们不再直接使用 `db.Get` 来查询记录，而是先通过 `udb := db.Unsafe()` 获取 `unsafe` 属性为 `true` 的 `*sqlx.DB` 对象，然后再调用它的 `Get` 方法。

`*sqlx.DB` 定义如下：

```go
type DB struct {
	*sql.DB
	driverName string
	unsafe     bool
	Mapper     *reflectx.Mapper
}
```

当 `unsafe` 属性为 `true` 时，`*sqlx.DB` 对象会忽略不匹配的字段，使代码能够正常运行，并将能够匹配的字段正确绑定到 `user` 结构体对象上。

通过这个属性的名称我们就知道，这是不安全的做法，不被推荐。

与未使用的变量一样，被忽略的列是对网络和数据库资源的浪费，并且这很容易导致出现模型与数据库表不匹配而不被感知的情况。

### Scan 变体

前文示例中，我们见过了 `*sqlx.Rows.Scan` 的变体 `*sqlx.Rows.StructScan` 的用法，它能够方便的将查询结果扫描到 `struct` 中。

`sqlx` 还提供了 `*sqlx.Rows.MapScan`、`*sqlx.Rows.SliceScan` 两个方法，能够将查询结果分别扫描到 `map` 和 `slice` 中。

使用示例如下：

```go
func MapScan(db *sqlx.DB) ([]map[string]interface{}, error) {
	rows, err := db.Queryx("SELECT * FROM user")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var res []map[string]interface{}
	for rows.Next() {
		r := make(map[string]interface{})
		err := rows.MapScan(r)
		if err != nil {
			return nil, err
		}
		res = append(res, r)
	}
	return res, err
}

func SliceScan(db *sqlx.DB) ([][]interface{}, error) {
	rows, err := db.Queryx("SELECT * FROM user")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var res [][]interface{}
	for rows.Next() {
		// cols is an []interface{} of all the column results
		cols, err := rows.SliceScan()
		if err != nil {
			return nil, err
		}
		res = append(res, cols)
	}
	return res, err
}
```

其中，`rows.MapScan(r)` 用法与 `rows.StructScan(&u)` 用法类似，都是将接收查询结果集的目标模型指针变量当作参数传递进来。

`rows.SliceScan()` 用法略有不同，它不接收参数，而是将结果保存在 `[]interface{}` 中并返回。

可以按需使用以上两个方法。

### 控制字段名称映射

讲到这里，想必不少同学心里可能存在一个疑惑，`rows.StructScan(&u)` 在将查询记录的字段映射到对应结构体属性时，是如何找到对应关系的呢？

答案就是 `db` 结构体标签。

回顾前文讲 [声明模型](#声明模型) 时，`User` 结构体中定义的 `CreatedAt`、`UpdatedAt` 两个字段，定义如下：

```go
CreatedAt time.Time `db:"created_at"`
UpdatedAt time.Time `db:"updated_at"`
```

这里显式的标明了结构体标签 `db`，`sqlx` 正是使用 `db` 标签来映射查询字段和模型属性。

默认情况下，结构体字段会被映射成全小写形式，如 `ID` 字段会被映射为 `id`，而 `CreatedAt` 字段会被映射为 `createdat`。

因为在 `user` 数据库表中，创建时间和更新时间两个字段分别为 `created_at`、`updated_at`，与 `sqlx` 默认字段映射规则不匹配，所以我才显式的为 `CreatedAt` 和 `UpdatedAt` 两个字段指明了 `db` 标签，这样 `sqlx` 的 `rows.StructScan` 就能正常工作了。

当然，数据库字段不一定都是小写，如果你的数据库字段为全大写，`sqlx` 提供了 `*sqlx.DB.MapperFunc` 方法来控制查询字段和模型属性的映射关系。

其使用示例如下：

```go
func MapperFuncUseToUpper(db *sqlx.DB) (User, error) {
	copyDB := sqlx.NewDb(db.DB, db.DriverName())
	copyDB.MapperFunc(strings.ToUpper)

	var user User
	err := copyDB.Get(&user, "SELECT id as ID, name as NAME, email as EMAIL FROM user WHERE id = ?", 1)
	if err != nil {
		return User{}, err
	}
	return user, nil
}
```

这里为了不改变原有的 `db` 对象，我们复制了一个 `copyDB`，调用 `copyDB.MapperFunc` 并将 `strings.ToUpper` 传递进来。

注意这里的查询语句中，查询字段全部通过 `as` 重新命名成了大写形式，而 `User` 模型字段 `db` 默认都为小写形式。

`copyDB.MapperFunc(strings.ToUpper)` 的作用，就是在调用 `Get` 方法将查询结果扫描到结构体时，把 `User` 模型的小写字段，通过 `strings.ToUpper` 方法转成大写，这样查询字段和模型属性就全为大写了，也就能够一一匹配上了。

还有一种情况，如果你的模型已存在 `json` 标签，并且不想重复的再抄一遍到 `db` 标签，我们可以直接使用 `json` 标签来映射查询字段和模型属性。

```go
func MapperFuncUseJsonTag(db *sqlx.DB) (User, error) {
	copyDB := sqlx.NewDb(db.DB, db.DriverName())
	// Create a new mapper which will use the struct field tag "json" instead of "db"
	copyDB.Mapper = reflectx.NewMapperFunc("json", strings.ToLower)

	var user User
	// json tag
	err := copyDB.Get(&user, "SELECT id, name as username, email FROM user WHERE id = ?", 1)
	if err != nil {
		return User{}, err
	}
	return user, nil
}
```

这里需要直接修改 `copyDB.Mapper` 属性，赋值为 `reflectx.NewMapperFunc("json", strings.ToLower)` 将模型映射的标签由 `db` 改为 `json`，并通过 `strings.ToLower` 方法转换为小写。

`reflectx` 按照如下方式导入：

```go
import "github.com/jmoiron/sqlx/reflectx"
```

现在，查询语句中 `name` 属性通过使用 `as` 被重命名为 `username`，而 `username` 刚好与 `User` 模型中 `Name` 字段的 `json` 标签相对应：

```go
Name     sql.NullString `json:"username"`
```

所以，以上示例代码能够正确映射查询字段和模型属性。

### 总结

`sqlx` 建立在 `database/sql` 包之上，用于简化和增强与关系型数据库的交互操作。

对常见数据库操作方法，`sqlx` 提供了 `Must` 版本，如 `sqlx.MustOpen` 用来连接数据库，`*sqlx.DB.MustExec` 用来执行 SQL 语句，当遇到 `error` 时将会直接 `panic`。

`sqlx` 还扩展了查询方法 `*sqlx.DB.Queryx`、`*sqlx.DB.QueryRowx`、`*sqlx.DB.Get`、`*sqlx.DB.Select`，并且这些查询方法支持直接将查询结果扫描到结构体。

`sqlx` 为 SQL IN 操作提供了便捷方法 `sqlx.In`。

为了使 SQL 更易阅读，`sqlx` 提供了 `*sqlx.DB.NamedExec`、`*sqlx.DB.NamedQuery` 两个方法支持具名参数。

调用 `*sqlx.DB.Unsafe()` 方法能够获取 `unsafe` 属性为 `true` 的 `*sqlx.DB` 对象，在将查询结果扫描到结构体使可以用来忽略不匹配的记录字段。

除了能够将查询结果扫描到 `struct`，`sqlx` 还支持将查询结果扫描到 `map` 和 `slice`。

`sqlx` 使用 `db` 结构体标签来映射查询字段和模型属性，如果不显式指定 `db` 标签，默认映射的模型属性为小写形式，可以通过 `*sqlx.DB.MapperFunc` 函数来修改默认行为。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sqlx) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- sqlx 源码：https://github.com/jmoiron/sqlx
- sqlx 文档：https://pkg.go.dev/github.com/jmoiron/sqlx
- sqlx 官网：http://jmoiron.github.io/sqlx/
- 本文示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/sqlx
