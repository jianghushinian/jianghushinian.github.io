---
title: 在 Go 中如何使用 database/sql 来操作数据库
date: 2023-06-05 12:11:15
tags:
- Golang
- SQL
categories:
- Golang
---

在现代软件开发中，数据库扮演着至关重要的角色，用于存储和管理应用程序的数据。针对不同的数据库系统，开发人员通常需要使用特定的数据库驱动来操作数据库，这往往需要开发人员掌握不同的驱动编程接口。在 Go 语言中，好在有一个名为 `database/sql` 的标准库，提供了统一的编程接口，使开发人员能够以一种通用的方式与各种关系型数据库进行交互。

<!-- more -->

### 概念

`database/sql` 包通过提供统一的编程接口，实现了对不同数据库驱动的抽象。

它的大致原理如下：

1. `Driver` 接口定义：`database/sql/driver` 包中定义了一个 `Driver` 接口，该接口用于表示一个数据库驱动。驱动开发者需要实现该接口来提供与特定数据库的交互能力。

2. `Driver` 注册：驱动开发者需要在程序初始化阶段，通过调用 `database/sql` 包提供的 `sql.Register()` 方法将自己的驱动注册到 `database/sql` 中。这样，`database/sql` 就能够识别和使用该驱动。

3. 数据库连接池管理：`database/sql` 维护了一个数据库连接池，用于管理数据库连接。当通过 `sql.Open()` 打开一个数据库连接时，`database/sql` 会在合适的时机调用注册的驱动来创建一个具体的连接，并将其添加到连接池中。连接池会负责连接的复用、管理和维护工作，并且这是并发安全的。

4. 统一的编程接口：`database/sql` 定义了一组统一的编程接口供用户使用，如 `Prepare()`、`Exec()` 和 `Query()` 等方法，用于准备 SQL 语句、执行 SQL 语句和执行查询等操作。这些方法会接收参数并调用底层驱动的相应方法来执行实际的数据库操作。

5. 接口方法的实现：驱动开发者需要实现 `database/sql/driver` 中定义的一些接口方法，以此来支持上层 `database/sql` 包提供的 `Prepare()`、`Exec()` 和 `Query()` 等方法，以提供底层数据库的具体实现。当 `database/sql` 调用这些方法时，实际上会调用注册的驱动的相应方法来执行具体的数据库操作。

通过以上的机制，`database/sql` 包能够实现对不同数据库驱动的统一封装和调用。用户可以使用相同的编程接口来进行数据库操作，无需关心底层驱动的具体细节。这种设计使得代码更具可移植性和灵活性，方便切换和适配不同的数据库。

### 特点

`database/sql` 具有如下特点：

- 统一的编程接口：`database/sql` 库提供了一组统一的接口，使得开发人员可以使用相同的方式操作不同的数据库，而不需要学习特定数据库的 API。

- 驱动支持：通过导入第三方数据库驱动程序，`database/sql` 可以与多种常见的关系型数据库系统进行交互，如 MySQL、PostgreSQL、SQLite 等。

- 预防 SQL 注入：`database/sql` 库通过使用预编译语句和参数化查询等技术，有效预防了 SQL 注入攻击。

- 支持事务：事务是一个优秀的 SQL 包必备功能。

### 准备

为了演示 `database/sql` 用法，我准备了如下 MySQL 数据库表：

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

要使用 `database/sql` 操作数据库，首先要建立与数据库的连接：

```go
package main

import (
	"database/sql"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql",
		"user:password@tcp(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=true&loc=Local")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```

因为我们要连接 MySQL 数据库，所以需要导入 MySQL 数据库驱动 `github.com/go-sql-driver/mysql`。`database/sql` 由 Go 语言官方团队设计，而数据库驱动程序则由社区维护，其他关系型数据库驱动列表可在[这里](https://github.com/golang/go/wiki/SQLDrivers)查看。

与数据库建立连接的代码非常简单，只需调用 `sql.Open()` 函数即可。它接收两个参数，驱动名称和 DSN。

这里驱动名称为 `mysql`，`database/sql` 之所以能够识别这个驱动名称，是因为在匿名导入 `github.com/go-sql-driver/mysql` 时，这个库内部调用了 `sql.Register` 将其注册给了 `database/sql`。

```go
func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```

在 Go 语言中，一个包的 `init` 方法会在导入时会被自动调用，这里完成了驱动程序的注册。这样在调用 `sql.Open()` 时才能找到 `mysql` 驱动。

第二个参数 DSN 全称 `Data Source Name`，数据库的源名称，其格式如下：

```
username:password@protocol(address)/dbname?param=value
```

下面是我们提供的 DSN 各部分解释：

- `user:password`：数据库的用户名和密码。根据实际情况，你需要使用你自己的用户名和密码。

- `tcp(127.0.0.1:3306)`：连接数据库服务器的协议、数据库服务器的地址和端口号。在这个例子中，使用的是本地主机 `127.0.0.1` 和 MySQL 默认端口号 `3306`。你可以根据实际情况修改为你自己的数据库服务器地址和端口号。

- `/demo`：数据库的名称。在这个例子中，数据库名称是 `demo`。你可以根据实际情况修改为你自己的数据库名称。

- `charset=utf8mb4`：指定数据库的字符集为 `UTF-8`。这里使用的是 `UTF-8` 的变体 `UTF-8mb4`，支持更广泛的字符范围。

- `parseTime=true`：启用时间解析。这个参数使得数据库驱动程序能够将数据库中的时间类型字段（`datetime`）解析为 Go 语言的 `time.Time` 类型。

- `loc=Local`：设置时区为本地时区。这个参数指定数据库驱动程序使用本地的时区。

`sql.Open()` 调用后将返回一个 `*sql.DB` 类型，可以用来操作数据库。

另外，我们调用 `defer db.Close()` 来释放数据库连接。其实这一步操作也可以不做，`database/sql` 底层连接池会帮我们处理。一旦关闭了连接，就不可以再继续使用这个 `db` 对象了。

`*sql.DB` 的设计是用来作为长连接使用的，所以不需要频繁的进行 `Open` 和 `Close` 操作。如果我们需要连接多个数据库，则可以为每个不同的数据库创建一个 `*sql.DB` 对象，保持这些对象为 `Open` 状态，不必频繁使用 `Close` 来切换连接。

值得注意的是，其实 `sql.Open()` 并没有真正建立数据库连接，它只是准备好了一切，以备后续使用，连接将在第一次被使用时延迟建立。

这样的设计虽然合理，可也有些违反直觉，`sql.Open()` 甚至不会校验 DSN 参数的合法性。不过我们可以使用 `db.Ping()` 方法来主动检查连接是否能被正确建立。

```go
if err := db.Ping(); err != nil {
    log.Fatal(err)
}
```

使用 `sql.Open()` 并不会建立一个唯一的数据库连接，事实上，`database/sql` 会维护一个连接池。

我们可以通过如下方法，控制连接池的一些参数：

```go
db.SetMaxOpenConns(25)                 // 设置最大的并发连接数（in-use + idle）
db.SetMaxIdleConns(25)                 // 设置最大的空闲连接数（idle）
db.SetConnMaxLifetime(5 * time.Minute) // 设置连接的最大生命周期
```

这些参数设置可以根据经验来修改，以上参数能够满足一些中小项目的使用，如果是大型项目，则可以适当调高参数。

### 声明模型

连接建立好后，理论上我们就可以操作数据库进行 CRUD 了。不过为了写出的代码更具可维护性，我们往往需要定义`模型`来映射数据库表。

`user` 表映射后的模型如下：

```go
type User struct {
	ID        int
	Name      sql.NullString
	Email     string
	Age       int
	Birthday  *time.Time
	Salary    Salary
	CreatedAt time.Time
	UpdatedAt string
}
```

模型在 Go 中使用 `struct` 表示，结构体字段同数据库表中的字段一一对应。

其中 `Salary` 类型定义如下：

```go
type Salary struct {
	Month int `json:"month"`
	Year  int `json:"year"`
}
```

关于 `Name`、`Salary` 两个字段的特殊性，我将分别在 [处理 NULL](#处理-NULL) 和 [自定义字段类型](#自定义字段类型) 部分讲解。

### 创建

`*sql.DB` 提供了 `Exec` 方法来执行一条 SQL 命令，可以用来创建、更新、删除表数据等。

这里使用 `Exec` 方法来实现创建一个用户：

```go
func CreateUser(db *sql.DB) (int64, error) {
	birthday := time.Date(2000, 1, 1, 0, 0, 0, 0, time.Local)
	user := User{
		Name:     sql.NullString{String: "jianghushinian007", Valid: true},
		Email:    "jianghushinian007@outlook.com",
		Age:      10,
		Birthday: &birthday,
		Salary: Salary{
			Month: 100000,
			Year:  10000000,
		},
	}
	res, err := db.Exec(`INSERT INTO user(name, email, age, birthday, salary) VALUES(?, ?, ?, ?, ?)`,
		user.Name, user.Email, user.Age, user.Birthday, user.Salary)
	if err != nil {
		return 0, err
	}
	return res.LastInsertId()
}
```

首先我们实例化了一个 `User` 对象 `user`，并对相应字段进行赋值。

接着使用 `db.Exec` 方法来执行 SQL 语句：

```SQL
INSERT INTO user(name, email, age, birthday, salary) VALUES(?, ?, ?, ?, ?)
```

其中 `?` 作为参数占位符，不同数据库驱动程序的占位符可能不同，可以参考数据库驱动的文档。

我们将这 5 个参数顺序传递给 `db.Exec` 方法，即可完成用户的创建。

`db.Exec` 方法调用后将返回 `sql.Result` 保存结果以及一个 `error` 来标记错误。

`sql.Result` 是一个接口，它包含两个方法：

- `LastInsertId() (int64, error)`：返回新插入的用户 ID。

- `RowsAffected() (int64, error)`：返回当前操作受影响的行数。

接口具体实现有数据库驱动程序来完成。

调用 `CreateUser` 函数即可创建一个新的用户：

```go
if id, err := CreateUser(db); err != nil {
	log.Fatal(err)
} else {
	log.Println("id:", id)
}
```

此外，`database/sql` 还提供了预处理方法 `*sql.DB.Prepare` 创建一个准备好的 SQL 语句，在循环中使用预处理，则可以减少与数据库的交互次数。

比如我们需要创建两个用户，则可以先使用 `db.Prepare` 创建一个 `*sql.Stmt` 对象，然后多次调用 `*sql.Stmt.Exec` 方法来插入数据：

```go
func CreateUsers(db *sql.DB) ([]int64, error) {
	stmt, err := db.Prepare("INSERT INTO user(name, email, age, birthday, salary) VALUES(?, ?, ?, ?, ?)")
	if err != nil {
		panic(err)
	}
	// 注意：预处理对象是需要关闭的
	defer stmt.Close()

	birthday := time.Date(2000, 2, 2, 0, 0, 0, 0, time.Local)
	users := []User{
		{
			Name:     sql.NullString{String: "", Valid: true},
			Email:    "jianghushinian007@gmail.com",
			Age:      20,
			Birthday: &birthday,
			Salary: Salary{
				Month: 200000,
				Year:  20000000,
			},
		},
		{
			Name:  sql.NullString{String: "", Valid: false},
			Email: "jianghushinian007@163.com",
			Age:   30,
		},
	}

	var ids []int64
	for _, user := range users {
		res, err := stmt.Exec(user.Name, user.Email, user.Age, user.Birthday, user.Salary)
		if err != nil {
			return nil, err
		}
		id, err := res.LastInsertId()
		if err != nil {
			return nil, err
		}
		ids = append(ids, id)
	}
	return ids, nil
}
```

`db.Prepare` 是预先将一个数据库连接和一个条 SQL 语句绑定并返回 `*sql.Stmt` 结构体，它代表了这个绑定后的连接对象，是并发安全的。

通过使用预处理，可以避免在循环中执行多次完整的 SQL 语句，从而显著减少了数据库交互次数，这可以提高应用程序的性能和效率。

使用预处理，会在 `db.Prepare` 时从连接池获取一个连接，之后循环执行 `stmt.Exec`，最终释放连接。

如果使用 `db.Exec`，则每次循环时都需要：获取连接-执行 SQL-释放连接，这几个步骤，大大增加了与数据库的交互次数。

不要忘记调用 `stmt.Close()` 关闭连接，这个方法是密等的，可以多次调用。

### 查询

现在数据库里已经有了数据，我们就可以查询数据了。

因为 `Exec` 方法只会执行 SQL，不会返回结果，所以不适用于查询数据。

`*sql.DB` 提供了 `Query` 方法执行查询操作：

```go
func GetUsers(db *sql.DB) ([]User, error) {
	rows, err := db.Query("SELECT * FROM user;")
	if err != nil {
		return nil, err
	}
	defer func() { _ = rows.Close() }()

	var users []User
	for rows.Next() {
		var user User
		if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.Age,
			&user.Birthday, &user.Salary, &user.CreatedAt, &user.UpdatedAt); err != nil {
			log.Println(err.Error())
			continue
		}
		users = append(users, user)
	}
	// 处理错误
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return users, nil
}
```

`db.Query` 返回查询结果集 `*sql.Rows`，这是一个结构体。

`rows.Next()` 方法用来判断是否还有下一条结果，可以用于 `for` 循环（题外话：这有点像 Python 的迭代器，只不过下一个值不是直接返回，而是通过 `Scan` 方法获取）。

如果存在下一条结果，`rows.Next()` 将返回 `true`。

`rows.Scan()` 方法可以将结果扫描到传递进来的指针对象。因为我们使用了 `SELECT *` 来查询，所以会返回全部字段的数据，按顺序将 `user` 对象相应的字段指针传递进来即可。

`rows.Scan()` 会将一行记录分别填入指定的变量中，并且会自动根据目标变量的类型处理类型转换的问题，比如数据库中是 `varchar` 类型，会映射成 Go 中的 `string`，但如果与之对应的目标变量是 `int`，那么转换失败就会返回 `error`。

`CreatedAt` 字段是 `time.Time` 类型，之所以能够被正确处理，是因为在调用 `sql.Open()` 时传递的 DSN 包含 `parseTime=true` 参数。

当 `rows.Next()` 返回为 `false` 时，即不再有下一条记录。我们也就将全部查询出来的用户都存储到 `users` 切片中了。

循环结束后，切记一定要调用 `rows.Err()` 来处理错误。

以上，我们查询了多条用户，`*sql.DB` 还提供了 `QueryRow` 方法可以查询单条记录：

```go
func GetUser(db *sql.DB, id int64) (User, error) {
	var user User
	row := db.QueryRow("SELECT * FROM user WHERE id = ?", id)
	err := row.Scan(&user.ID, &user.Name, &user.Email, &user.Age,
		&user.Birthday, &user.Salary, &user.CreatedAt, &user.UpdatedAt)
	switch {
	case err == sql.ErrNoRows:
		return user, fmt.Errorf("no user with id %d", id)
	case err != nil:
		return user, err
	}
	// 处理错误
	if err := row.Err(); err != nil {
		return user, err
	}
	return user, nil
}
```

查询单条记录会返回 `*sql.Row` 结构体，它实际上是对 `*sql.Rows` 的一层包装：

```go
type Row struct {
	// One of these two will be non-nil:
	err  error // deferred error for easy chaining
	rows *Rows
}
```

我们不再需要调用 `rows.Next()` 判断是否有下一条结果，调用 `row.Sca()` 时 `*sql.Row` 会自动帮我们处理好，返回查询结果集中的第一条数据。

如果 `row.Sca()` 返回的错误类型为 `sql.ErrNoRows` 说明没有查询到符合条件的数据，这对于判断错误类型特别有用。

但 `database/sql` 显然不能枚举出所有数据库的错误类型，有些针对不同数据库的指定错误类型，通常由数据库驱动程序来定义。

可以按照如下方式判断特定的 MySQL 错误类型：

```go
if driverErr, ok := err.(*mysql.MySQLError); ok {
    if driverErr.Number == 1045 {
        // 处理被拒绝的错误
    }
}
```

不过像 `1045` 这种魔法数字最好不要出现在代码中，[mysqlerr](https://github.com/VividCortex/mysqlerr) 包提供了 MySQL 错误类型的枚举。

以上代码可以改为：

```go
if driverErr, ok := err.(*mysql.MySQLError); ok  {
    if driverErr.Number == mysqlerr.ER_ACCESS_DENIED_ERROR {
        // 处理被拒绝的错误
    }
}
```

最后，同样不要忘记调用 `row.Err()` 处理错误。

### 更新

更新操作同创建一样可以使用 `*sql.DB.Exec` 方法来实现，不过这里我们将使用 `*sql.DB.ExecContext` 方法来实现。

`ExecContext` 方法与 `Exec` 方法在使用上没什么两样，只不过第一个参数需要接收一个 `context.Context`，它允许你控制和取消执行 SQL 语句的操作。使用上下文可以在需要的情况下设置超时时间、处理请求取消等操作。

```go
func UpdateUserName(db *sql.DB, id int64, name string) error {
	ctx := context.Background()
	res, err := db.ExecContext(ctx, "UPDATE user SET name = ? WHERE id = ?", name, id)
	if err != nil {
		return err
	}
	affected, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if affected == 0 {
		// 如果新的 name 等于原 name，也会执行到这里
		return fmt.Errorf("no user with id %d", id)
	}
	return nil
}
```

这里使用 `res.RowsAffected()` 获取了当前操作影响的行数。

注意，如果更新后的字段结果没有变化，`res.RowsAffected()` 返回 0。

### 删除

使用 `*sql.DB.ExecContext` 方法实现删除用户：

```go
func DeleteUser(db *sql.DB, id int64) error {
	ctx := context.Background()
	res, err := db.ExecContext(ctx, "DELETE FROM user WHERE id = ?", id)
	if err != nil {
		return err
	}
	affected, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if affected == 0 {
		return fmt.Errorf("no user with id %d", id)
	}
	return nil
}
```

### 事务

事务基本是开发 Web 项目时比不可少的数据库功能，`database/sql` 提供了对事务的支持。

如下示例使用事务来更新用户：

```go
func Transaction(db *sql.DB, id int64, name string) error {
	ctx := context.Background()
	tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
	if err != nil {
		return err
	}

	_, execErr := tx.ExecContext(ctx, "UPDATE user SET name = ? WHERE id = ?", name, id)
	if execErr != nil {
		if rollbackErr := tx.Rollback(); rollbackErr != nil {
			log.Fatalf("update failed: %v, unable to rollback: %v\n", execErr, rollbackErr)
		}
		log.Fatalf("update failed: %v", execErr)
	}
	return tx.Commit()
}
```

`*sql.DB.BeginTx` 用于开启一个事务，第一个参数为 `context.Context`，第二个参数为 `*sql.TxOptions` 对象，用来配置事务选项，`Isolation` 字段用来设置数据库隔离级别，关于隔离级别含义可以参考[这里](https://en.wikipedia.org/wiki/Isolation_%28database_systems%29#Isolation_levels)。

事务中执行的 SQL 语句需要放在 `tx` 对象的 `ExecContext` 方法中执行，而不是 `db.ExecContext`。

如果执行 SQL 过程中出现错误，可以使用 `tx.Rollback()` 进行事务回滚。

如果没有错误，则可以使用 `tx.Commit()` 提交事务。

`tx` 同样支持 `Prepare` 方法，可以点击[这里](https://pkg.go.dev/database/sql@go1.20.4#example-Tx.Prepare)查看使用示例。

### 处理 NULL

在创建 `User` 模型时，我们定义 `Name` 字段类型为 `sql.NullString`，而非普通的 `string` 类型，这是为了支持数据库中的 `NULL` 类型。

数据库中 `name` 字段定义如下：

```sql
`name` varchar(50) DEFAULT NULL COMMENT '用户名'
```

那么 `name` 在数据库中可能的值将有三种情况：`NULL`、空字符串 `''` 以及有值的字符串 `'n1'`。

我们知道，Go 语言中 `string` 类型的默认值即为空字符串 `''`，但是 `string` 无法表示 `NULL` 值。

这个时候，我们有两种方法解决此问题：

1. 使用指针类型。

2. 使用 `sql.NullString` 类型。

因为指针类型可能为 `nil`，所以可以使用 `nil` 来对应 `NULL` 值。这就是 `User` 模型中 `Birthday` 字段类型定义为 `*time.Time` 的缘故。

`sql.NullString` 类型定义如下：

```go
type NullString struct {
	String string
	Valid  bool // Valid is true if String is not NULL
}
```

`String` 用来记录值，`Valid` 用来标记是否为 `NULL`。

`NullString` 结构体的值和数据库中实际存储的值，有如下映射关系：

|          value           | value for MySQL  |
|:------------------------:|:----------------:|
| `{String:n1 Valid:true}` |      `'n1'`      |
| `{String: Valid:true}`   |       `''`       |
| `{String: Valid:false}`  |      `NULL`      |

此外，`sql.NullString` 类型还实现了 `sql.Scanner` 和 `driver.Valuer` 两个接口：

```go
// Scan implements the Scanner interface.
func (ns *NullString) Scan(value any) error {
	if value == nil {
		ns.String, ns.Valid = "", false
		return nil
	}
	ns.Valid = true
	return convertAssign(&ns.String, value)
}

// Value implements the driver Valuer interface.
func (ns NullString) Value() (driver.Value, error) {
	if !ns.Valid {
		return nil, nil
	}
	return ns.String, nil
}
```

这两个接口分别用在 `*sql.Row.Scan` 方法和 `*sql.DB.Exec` 方法。

即在使用 `*sql.DB.Exec` 方法执行 SQL 时，我们可能需要将 `Name` 字段的值存入 MySQL，此时 `database/sql` 会调用 `sql.NullString` 类型的 `Value()` 方法，获取其将要存储于数据库中的值。

在使用 `*sql.Row.Scan` 方法时，我们可能需要将从数据库获取到的 `name` 字段值映射到 `User` 结构体字段 `Name` 上，此时 `database/sql` 会调用 `sql.NullString` 类型的 `Scan()` 方法，把从数据库中查询的值赋值给 `Name` 字段。

如果你使用的字段不是 `string` 类型，`database/sql` 还提供了 `sql.NullBool`、`sql.NullFloat64` 等类型供用户使用。

但是，这并不能枚举出所有 MySQL 数据库支持的字段类型，所以如果能够尽量避免，还是不建议数据库字段允许 `NULL` 值。

### 自定义字段类型

有些时候，我们保存在数据库中的数据有着特定的格式，比如 `salary` 字段在数据库中存储的值为 `{"month":100000,"year":10000000}`。

数据库中 `salary` 字段定义如下：

```sql
`salary` varchar(128) DEFAULT NULL COMMENT '薪水'
```

如果只是将其映射为 Go 中的 `string`，则操作时要格外小心，如果忘记写一个 `"` 或 `,` 等，程序将可能报错。

因为 `salary` 值明显是一个 JSON 格式，我们可以定义一个 `struct` 来映射其内容：

```go
type Salary struct {
	Month int `json:"month"`
	Year  int `json:"year"`
}
```

这还不够，自定义类型无法支持 `*sql.Row.Scan` 方法和 `*sql.DB.Exec` 方法。

不过，我想你已经猜到了，我们可以参考 `sql.NullString` 类型让 `Salary` 同样实现 `sql.Scanner` 和 `driver.Valuer` 两个接口：

```go
// Scan implements sql.Scanner
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

// Value implements driver.Valuer
func (s Salary) Value() (driver.Value, error) {
	v, err := json.Marshal(s)
	return string(v), err
}
```

这样，存储和查询数据的操作，`Salary` 类型都能够支持。

### 未知列

极端情况下，我们可能不知道使用 `*sql.DB.Query` 方法查询的结果集中数据的列名以及字段个数。

此时，我们可以使用 `*sql.Rows.Columns` 方法获取所有列名，这将返回一个切片，这个切片长度，即为字段个数。

示例代码如下：

```go
func HandleUnknownColumns(db *sql.DB, id int64) ([]interface{}, error) {
	var res []interface{}
	rows, err := db.Query("SELECT * FROM user WHERE id = ?", id)
	if err != nil {
		return res, err
	}
	defer func() { _ = rows.Close() }()

	// 如果不知道列名称，可以使用 rows.Columns() 查找列名称列表
	cols, err := rows.Columns()
	if err != nil {
		return res, err
	}

	fmt.Printf("columns: %v\n", cols) // [id name email age birthday salary created_at updated_at]
	fmt.Printf("columns length: %d\n", len(cols))

	// 获取列类型信息
	types, err := rows.ColumnTypes()
	if err != nil {
		return nil, err
	}
	for _, typ := range types {
		// id: &{name:id hasNullable:true hasLength:false hasPrecisionScale:false nullable:false length:0 databaseType:INT precision:0 scale:0 scanType:0x1045d68a0}
		fmt.Printf("%s: %+v\n", typ.Name(), typ)
	}

	res = []interface{}{
		new(int),            // id
		new(sql.NullString), // name
		new(string),         // email
		new(int),            // age
		new(time.Time),      // birthday
		new(Salary),         // salary
		new(time.Time),      // created_at
		// 如果不知道列类型，可以使用 sql.RawBytes，它实际上是 []byte 的别名
		new(sql.RawBytes), // updated_at
	}

	for rows.Next() {
		if err := rows.Scan(res...); err != nil {
			return res, err
		}
	}
	return res, rows.Err()
}
```

除了获取列名和字段个数，我们还可以使用 `*sql.Rows.ColumnTypes` 方法获取每个 `column` 的详细信息。

如果我们不知道某个字段在数据库中的类型，则可以将其映射为 `sql.RawBytes` 类型，它实际上是 `[]byte` 的别名。

### 总结

`database/sql` 包统一了 Go 语言操作数据库的编程接口，避免了操作不同数据库需要学习多套 API 的窘境。

使用 `sql.Open()` 建立数据库连接并不会立刻生效，连接会在合适的时候延迟建立。

我们可以使用 `*sql.DB.Exec` / `*sql.DB.ExecContext` 来执行 SQL 命令。其实除了 `Exec` 有方法对应的 `ExecContext` 版本，文中提到的 `*sql.DB.Ping`、`*sql.DB.Query`、`*sql.DB.QueryRow`、`*sql.DB.Prepare` 方法也都有对应的 `XxxContext` 版本，你可以自行测试。

如果被执行的 SQL 语句中包含 MySQL 关键字，则需要使用反引号（\`）将关键字进行包裹，否则你将得到 `Error 1064 (42000): You have an error in your SQL syntax;` 错误。

`*sql.DB.BeginTx` 可以开启一个事务，事务需要显式的 `Commit` 或 `Rollback`，MySQL 驱动还支持使用 `*sql.TxOptions` 设置事务隔离级别。

对于 `NULL` 类型，`database/sql` 提供了 `sql.NullString` 等类型的支持。我们也可以为自定义类型实现 `sql.Scanner` 和 `driver.Valuer` 两个接口，来实现特定逻辑。

对于未知列和字段类型，我们可以使用 `*sql.Rows.Columns`、`sql.RawBytes` 等来解决，虽然这极大的增加了灵活性，不过不到万不得已不建议使用，使用更加明确的代码可以减少 BUG 的数量和提高可维护性。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/database/sql) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- database/sql 文档：https://pkg.go.dev/database/sql@go1.20.4
- SQLDrivers 驱动列表：https://github.com/golang/go/wiki/SQLDrivers
- MySQL Driver：https://github.com/go-sql-driver/mysql/
- Go database/sql tutorial：http://go-database-sql.org/index.html
- Configuring sql.DB for Better Performance：https://www.alexedwards.net/blog/configuring-sqldb
- 本文示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/database/sql
