---
title: Go 语言中 database/sql 是如何设计的
date: 2023-06-21 21:47:11
tags:
- Go
- SQL
categories:
- Go
---

常见的关系型数据库都支持标准的 SQL 语言，所以无论是 MySQL、PostgreSQL 还是 SQL Server，我们都可以使用相同的 SQL 语句来对其进行操作。这种思想同样体现在 Go 语言的数据库操作中，在 Go 语言中内置了 `database/sql` 包，它只对外暴露了一套统一的编程接口，便可以操作不同数据库。

<!-- more -->

本文重点讲解 `database/sql` 设计思想，默认读者已经有了 `database/sql` 使用经验，对于 `database/sql` 功能则不会详细介绍。如果你对 `database/sql` 不熟悉，可以查看我的另一篇文章[《在 Go 中如何使用 database/sql 来操作数据库》](https://jianghushinian.cn/2023/06/05/how-to-use-database-sql-to-operate-database-in-go/)。

### 接口设计

首先我们来看下 `database/sql` 目录结构长什么样：

```go
go1.20.1/src/database
└── sql
    ├── driver
    │   ├── driver.go
    │   └── types.go
    ├── convert.go
    ├── ctxutil.go
    ├── sql.go
    ...
```

> 笔记：这里没有列出测试文件。

可以发现，`database/sql` 实际上包含了 `sql` 包及其子包 `driver`。

`sql` 包为我们提供了操作数据库的对象以及方法，`driver` 包则定义了数据库驱动的编程接口，这些接口都是第三方驱动包需要实现的。

现在我们一起来看下 `driver` 包是如何设计的。

在 `driver` 包中定义了一个 `Driver` 接口：

```go
type Driver interface {
	Open(name string) (Conn, error)
}
```

这个接口只有一个 `Open` 方法，用来建立一个数据库连接并返回。

`Open` 方法的 `name` 参数即为 DSN，返回值中的 `Conn` 接口则代表了一个数据库连接，定义如下：

```go
type Conn interface {
	Prepare(query string) (Stmt, error)
	Close() error
	Begin() (Tx, error)
}
```

`Conn` 接口包含三个方法：

`Prepare` 用来预处理 SQL，返回一个准备好的 SQL 语句。

`Close` 用来关闭数据库连接。

`Begin` 显然是对事务的支持。

其中 `Prepare` 返回 `Stmt` 类型，这也是一个接口，定义如下：

```go
type Stmt interface {
	Close() error
	NumInput() int
	Exec(args []Value) (Result, error)
	Query(args []Value) (Rows, error)
}
```

`Close` 用来关闭该预处理语句。

`NumInput` 返回 SQL 中占位符参数的数量。

`Exec` 和 `Query` 两个方法我们再熟悉不过了，分别用来执行 SQL 命令以及查询记录。这两个方法都接收参数 `[]Value`，`Value` 其实是 `any` 类型，也就是 `interface{}`，定义如下：

```go
type Value any
```

`Exec` 方法返回的 `Result` 接口定义如下：

```go
type Result interface {
	LastInsertId() (int64, error)
	RowsAffected() (int64, error)
}
```

`LastInsertId` 返回 INSERT SQL 插入记录的 ID。

`RowsAffected` 返回受影响记录的行数。

`Query` 方法返回的 `Rows` 接口定义如下：

```go
type Rows interface {
	Columns() []string
	Close() error
	Next(dest []Value) error
}
```

当我们执行 SQL 查询时，如果不知道列名，可以使用 `rows.Columns()` 查看所有列名称列表。

`Close` 用来关闭 `Rows` 的迭代器，关闭后无法再继续调用 `Next` 查询下一条记录。

调用 `Next` 可以将下一行数据填充到提供的 `dest` 切片中。

`Value` 在上面已经介绍过了，是 `any` 类型。

现在 `Conn` 接口中定义的 `Prepare` 方法这条线所涉及到的类型，我们已经追查到底了，是时候回过头来看下 `Begin` 方法返回的 `Tx` 类型定义了：

```go
type Tx interface {
	Commit() error
	Rollback() error
}
```

`Tx` 不出所料，同样是一个接口，包含两个方法：

`Commit` 用来提交事务。

`Rollback` 用来回滚事务。

至此，`Driver` 接口的设计就清晰的摆在眼前了：

![Driver](driver.png)

除了 `Driver` 接口，在 `database/sql/driver` 包中，还有几个常用接口定义如下：

```go
type Connector interface {
	Connect(context.Context) (Conn, error)
	Driver() Driver
}

type Pinger interface {
	Ping(ctx context.Context) error
}

type Execer interface {
	Exec(query string, args []Value) (Result, error)
}

type ExecerContext interface {
	ExecContext(ctx context.Context, query string, args []NamedValue) (Result, error)
}

type Queryer interface {
	Query(query string, args []Value) (Rows, error)
}

type QueryerContext interface {
	QueryContext(ctx context.Context, query string, args []NamedValue) (Rows, error)
}
```

`Connector` 接口用来连接数据库。

`Pinger` 接口用来检查连接是否能被正确建立。

还有 `Execer`、`ExecerContext`、`Queryer`、`QueryerContext` 这 4 个接口，正好对应了我们在利用 `database/sql` 时操作数据库所使用的方法。

所有这些接口，都是第三方数据库驱动包要实现的接口（有些接口是可选的）。

看到这里，你可能有个疑惑，为什么这些接口都只定义为只有一个方法的小接口？

这其实是 Go 语言中的惯用法，越小的接口抽象程度越高，易于解耦，也越容易被实现，并且非常适用于 Go 语言的组合机制。

好了，关于 `driver` 包中接口的定义部分就讲解到这里，其他用的比较少接口的我就不在这里介绍了，感兴趣的同学可以自行尝试阅读源码学习。

以上介绍的这些接口全部定义在 `driver/driver.go` 文件中，而 `driver/types.go` 文件中则用来定义类型，如 `Bool`、`Int32` 等方便用来类型转换，由于不是本文重点，这里也就不多介绍了。

### 代码实现

看了以上关于接口定义的讲解，你可能会觉得有些云里雾里，有种学了一身功夫却又无从下手的感觉。

没关系，接下来我将根据一段示例代码，带你深入到 `database/sql` 的源码中，加深你对 `database/sql` 包的理解。

以下示例是我们使用 `database/sql` 操作 MySQL 最典型的场景：

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=true&loc=Local")
	if err != nil {
		panic(err.Error())
	}
	defer db.Close()

	rows, err := db.Query("SELECT id, name FROM user")
	if err != nil {
		panic(err.Error())
	}
	defer rows.Close()

	for rows.Next() {
		var (
			id   int
			name string
		)
		if err := rows.Scan(id, name); err != nil {
			panic(err.Error())
		}
		fmt.Printf("id: %d, name: %s\n", id, name)
	}
}
```

这段代码最让初学者摸不着头脑的是我们以匿名的方式导入了 MySQL 驱动包：

```go
import _ "github.com/go-sql-driver/mysql"
```

但在实际的代码中并没有使用它。

其实，这看似有些奇怪的代码导入的作用，就隐藏在 `go-sql-driver/mysql` 包的 `init` 函数中：

```go
import "database/sql"

func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```

`go-sql-driver/mysql` 包导入并使用 `database/sql` 包的 `sql.Register` 函数，将自己实现的驱动程序注册到 `database/sql` 中。

调用注册函数的代码写在了 `init` 函数中，`go-sql-driver/mysql` 包正是利用了匿名导入时 Go 语言会自动调用被导入包的 `init` 方法所产生的副作用，来实现驱动注册。

注册驱动函数 `sql.Register` 实现如下：

```go
var (
	driversMu sync.RWMutex
	drivers   = make(map[string]driver.Driver)
)

func Register(name string, driver driver.Driver) {
	driversMu.Lock()
	defer driversMu.Unlock()
	if driver == nil {
		panic("sql: Register driver is nil")
	}
	if _, dup := drivers[name]; dup {
		panic("sql: Register called twice for driver " + name)
	}
	drivers[name] = driver
}
```

可以发现，`Register` 内部通过全局互斥锁变量 `driversMu` 保证了并发操作的安全性。在加锁的条件下，将 `mysql` 驱动保存在 `drivers` 这个全局的 map 类型变量中，以 `mysql` 为 `key`，驱动对象为 `value`。

这就是为什么，我们能够使用 `sql.Open` 函数与数据库建立连接的原因。

```go
func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}

	if driverCtx, ok := driveri.(driver.DriverContext); ok {
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
	}

	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

`Open` 函数接收两个参数，驱动名称和 DSN。

在 `Open` 函数内部，首先从全局变量 `drivers` 中获取驱动对象。而我们调用 `sql.Open` 函数时，传递的第一个参数是 `mysql`，这刚好与 `go-sql-driver/mysql` 包中注册的驱动名称对应，所以能够获取到 MySQL 驱动程序。

接着，代码中通过类型断言，判断驱动对象 `driveri` 是否为 `driver.DriverContext` 类型。

是的话就先调用驱动对象的 `OpenConnector` 方法得到 `Connector` 类型的对象，然后再使用 `OpenDB` 打开数据库连接。

`driver.DriverContext` 接口定义如下：

```go
type DriverContext interface {
	OpenConnector(name string) (Connector, error)
}
```

只包含了 `OpenConnector` 方法，这个方法返回 `Connector` 接口类型。

`Connector` 接口前文已经讲过，我们可以再回顾下它的定义：

```go
type Connector interface {
	Connect(context.Context) (Conn, error)
	Driver() Driver
}
```

这个接口定义了两个方法分别用来连接数据库和获取驱动对象。

而如果 `driveri` 不是 `driver.DriverContext` 类型，则需要先构造一个 `dsnConnector` 对象，然后再使用 `OpenDB` 函数打开数据库连接。

`dsnConnector` 是一个结构体，定义非常简单：

```go
type dsnConnector struct {
	dsn    string
	driver driver.Driver
}
```

只包含了 DSN 和驱动对象。

并且它同时也实现了 `Connector` 接口：

```go
func (t dsnConnector) Connect(_ context.Context) (driver.Conn, error) {
	return t.driver.Open(t.dsn)
}

func (t dsnConnector) Driver() driver.Driver {
	return t.driver
}
```

接下来，我们看看 `OpenDB` 函数是如何定义的：

```go
func OpenDB(c driver.Connector) *DB {
	ctx, cancel := context.WithCancel(context.Background())
	db := &DB{
		connector:    c,
		openerCh:     make(chan struct{}, connectionRequestQueueSize),
		lastPut:      make(map[*driverConn]string),
		connRequests: make(map[uint64]chan connRequest),
		stop:         cancel,
	}

	go db.connectionOpener(ctx)

	return db
}
```

在 `OpenDB` 函数内部实例化了一个 `*sql.DB` 结构体指针并返回。这个结构体由 `database/sql` 包提供，是统一用户编程接口的关键结构体，我们后续的查询操作，就是调用了这个对象上的方法。

这里实例化 `*sql.DB` 对象时，并不不会立即建立数据库连接，连接会在需要时被延迟建立。

在 `sql.DB` 结构体中，我们需要关注的是 `openerCh` 属性，这是一个 Channel 对象，是一个用来保存连接请求的队列，稍后我们将见到它的关键作用。

`db` 对象创建后，通过 `go db.connectionOpener(ctx)` 单独启用了一个协程，用来处理建立连接的请求。

函数最终返回了这个 `*sql.DB` 类型的 `db` 对象，此对象是并发安全的，支持多个 Goroutine 同时操作，并且维护了自己的空闲连接池。

`OpenDB` 函数只应被调用一次，且很少需要用户主动关闭连接。

`db.connectionOpener` 方法实现如下：

```go
func (db *DB) connectionOpener(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-db.openerCh:
			db.openNewConnection(ctx)
		}
	}
}
```

这里仅包含一个永不退出的无限循环，当 `db.openerCh` 这个 Channel 有值时，代码会进入 `db.openNewConnection` 函数的调用。

```go
func (db *DB) openNewConnection(ctx context.Context) {
	ci, err := db.connector.Connect(ctx)
	...
	dc := &driverConn{
		db:         db,
		createdAt:  nowFunc(),
		returnedAt: nowFunc(),
		ci:         ci,
	}
	if db.putConnDBLocked(dc, err) {
		db.addDepLocked(dc, dc)
	}
	...
}
```

在 `db.openNewConnection` 函数的第一行代码中，`db.connector` 属性是在之前调用 `OpenDB` 时进行赋值的一个 `driver.Connector` 接口类型对象（还记得前文讲的 `dsnConnector` 吗），调用它的 `Connect` 方法就可以与数据库建立连接了。

之后调用的 `db.putConnDBLocked(dc, err)` 方法作用是将这个连接放入数据库空闲连接池中（`db.freeConn` 属性）。

至此，我们得到了两条函数调用线：

![database/sql](database-sql.png)

在驱动包 `go-sql-driver/mysql` 中，通过 `sql.Register` 进行驱动程序注册。

在 `database/sql` 中，我们调用 `sql.OpenDB` 来开启数据库连接，这不会立刻建立连接，而是通过开启新的 Goroutine 阻塞在 `db.openerCh` Channel 上，等待建立连接请求。

那么接下来，何时触发 `*sql.DB.connectionOpener` 函数中 `<-db.openerCh` 这个 `case`，就是我们要研究的重点了。

我们可以顺着示例代码继续往下看。

在示例中，接下来使用 `db.Query("SELECT id, name FROM user")` 方法来查询 `user` 记录。

`*sql.DB.Query` 方法定义如下：

```go
func (db *DB) Query(query string, args ...any) (*Rows, error) {
	return db.QueryContext(context.Background(), query, args...)
}
```

它直接调用了 `*sql.DB.QueryContext`：

```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...any) (*Rows, error) {
	var rows *Rows
	var err error

	err = db.retry(func(strategy connReuseStrategy) error {
		rows, err = db.query(ctx, query, args, strategy)
		return err
	})

	return rows, err
}
```

`*sql.DB.QueryContext` 方法内部又调用了 `db.query` 方法：

```go
func (db *DB) query(ctx context.Context, query string, args []any, strategy connReuseStrategy) (*Rows, error) {
	dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
	}

	return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
}
```

在 `db.query` 方法内部，首先调用了 `db.conn` 方法。`db.conn` 顾名思义，就是用来建立数据库连接的，定义如下：

```go
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
	last := len(db.freeConn) - 1
	if strategy == cachedOrNewConn && last >= 0 {
		conn := db.freeConn[last]
		...
		return conn, nil
	}
	...
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:         db,
		createdAt:  nowFunc(),
		returnedAt: nowFunc(),
		ci:         ci,
		inUse:      true,
	}
	db.addDepLocked(dc, dc)
	db.mu.Unlock()
	return dc, nil
}
```

这里我省略了一些代码，只列出了比较重要的逻辑。

在函数内部，首先会尝试从 `db.freeConn` 空闲连接池中获取连接。

如果没有空闲连接，则调用 `db.connector.Connect` 来获取新的数据库连接。

当获取连接失败，会调用 `db.maybeOpenNewConnections()` 方法并返回错误。

这个 `db.maybeOpenNewConnections()` 方法是我们要关注的重点，定义如下：

```go
func (db *DB) maybeOpenNewConnections() {
	numRequests := len(db.connRequests)
	if db.maxOpen > 0 {
		numCanOpen := db.maxOpen - db.numOpen
		if numRequests > numCanOpen {
			numRequests = numCanOpen
		}
	}
	for numRequests > 0 {
		db.numOpen++ // optimistically
		numRequests--
		if db.closed {
			return
		}
		db.openerCh <- struct{}{}
	}
}
```

可以发现，正是在这个方法内部，调用了 `db.openerCh <- struct{}{}` 为 Channel 发送数据。

当 `db.openerCh` 有值时，会被前文讲解的通过子协程调用的 `*sql.DB.connectionOpener` 函数消费，以此来触发异步获取数据库连接操作。

前文有提到，异步创建的数据库连接会被放入空闲连接池 `db.freeConn` 中。

此时，我们再次回到 `db.query` 方法被调用的地方，来重新审视下 `*sql.DB.QueryContext` 方法的定义：

```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...any) (*Rows, error) {
	var rows *Rows
	var err error

	err = db.retry(func(strategy connReuseStrategy) error {
		rows, err = db.query(ctx, query, args, strategy)
		return err
	})

	return rows, err
}
```

这里并不是简单的直接调用 `db.query`，而是将其放入了 `db.retry` 方法中调用。

顾名思义，`db.retry` 方法是用来进行重试操作的，如果 `db.query` 调用失败，则会重试一次。

这就体现了当调用 `db.connector.Connect(ctx)` 失败时，调用 `db.maybeOpenNewConnections()` 方法异步建立连接的意义。

因为如果第一次创建连接失败，则 `db.retry` 会进行重试，下次重试的时候，再次进入 `db.conn` 方法，如果异步建立连接已经完成，则可以直接从空闲连接池 `db.freeConn` 中获取数据库连接。即使异步建立连接来不及完成，那么空闲连接池也会有一个新的连接被创建，下次有另外一个请求进来，也能够从空闲连接池中获取连接。这个操作能够提升程序的性能。

至此，`database/sql` 包中 `sql.Open` 和 `*sql.DB.Query` 两条函数调用线，我们就搞清楚了：

![database/sql](database-sql-1.png)

这两条函数调用线通信的关键，就是 `db.openerCh` 所在。

现在，上图中 `*sql.DB.Query` 这条函数调用线我们唯独没有搞清楚的就只剩下 `*sql.DB.queryDC` 的调用了。

`*sql.DB.queryDC` 定义如下：

```go
func (db *DB) queryDC(ctx, txctx context.Context, dc *driverConn, releaseConn func(error), query string, args []any) (*Rows, error) {
	queryerCtx, ok := dc.ci.(driver.QueryerContext)
	var queryer driver.Queryer
	if !ok {
		queryer, ok = dc.ci.(driver.Queryer)
	}
	if ok {
		var nvdargs []driver.NamedValue
		var rowsi driver.Rows
		var err error
		withLock(dc, func() {
			nvdargs, err = driverArgsConnLocked(dc.ci, nil, args)
			if err != nil {
				return
			}
			rowsi, err = ctxDriverQuery(ctx, queryerCtx, queryer, query, nvdargs)
		})
		...
	}
	...
}
```

这里断言了 `*driverConn` 中携带的查询对象是 `driver.QueryerContext` 还是 `driver.Queryer`，并将断言结果传递给 `ctxDriverQuery` 函数。

`ctxDriverQuery` 定义如下：

```go
func ctxDriverQuery(ctx context.Context, queryerCtx driver.QueryerContext, queryer driver.Queryer, query string, nvdargs []driver.NamedValue) (driver.Rows, error) {
	if queryerCtx != nil {
		return queryerCtx.QueryContext(ctx, query, nvdargs)
	}
	dargs, err := namedValueToValue(nvdargs)
	if err != nil {
		return nil, err
	}

	select {
	default:
	case <-ctx.Done():
		return nil, ctx.Err()
	}
	return queryer.Query(query, dargs)
}
```

在 `ctxDriverQuery` 函数内部，根据查询对象类型的不同，调用了 `queryerCtx.QueryContext` 或 `queryer.Query`。

这个操作正是在调用驱动程序对应的 `QueryContext` 或 `Query` 方法。

不管是 `driver.QueryerContext` 还是 `driver.Queryer`，都是 `database/sql/driver` 中定义的接口类型，`database/sql` 内部正是通过使用接口类型，来实现跟驱动程序 `go-sql-driver/mysql` 的解耦。

这样，`database/sql` 不直接跟 `go-sql-driver/mysql` 中定义的具体类型打交道，二者通过 `database/sql/driver` 这个中间层来交互，这便是 Go 语言接口用法的精髓所在。

现在，我们的函数调用线路图就已经完整了。

![database/sql](database-sql-2.png)

### 总结

本文带大家一起学习了 `database/sql` 包的设计思想，`database/sql/driver` 用来定义定义驱动需要实现的接口，`database/sql` 则为用户提供了操作数据库的方法。

这里涉及了一个使用 `init` 函数的技巧，利用 `init` 函数的副作用，可以实现不改 `database/sql` 任何代码的情况下，只需要 `import` 驱动程序，就能注册驱动程序的所有功能。

通过一个简单的示例程序，我们一起阅读了 `database/sql` 包的部分源码，以 `*sql.DB.Query` 方法作为示例，查看了 `database/sql` 最终是在何处调用驱动程序对应方法的。抛砖引玉，如果你对其他方法源码也感兴趣，可以顺着我讲解的思路继续深入学习。

`database/sql` 包统一了 Go 语言操作数据库的编程接口，避免了操作不同数据库需要学习多套 API 的窘境。

记住，在 Go 语言中使用接口来解耦是惯用方法，你一定要掌握。未来我们讲解如何编写单元测试代码的时候，还会用到。

注意：本文讲解的 `database/sql` 包源码版本为 [Go 1.20.1](https://github.com/golang/go/tree/go1.20.1/src/database/sql)，其他版本可能有所不同。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- database/sql 源码：https://github.com/golang/go/tree/go1.20.1/src/database/sql
- MySQL Driver：https://github.com/go-sql-driver/mysql/
- 在 Go 中如何使用 database/sql 来操作数据库：https://jianghushinian.cn/2023/06/05/how-to-use-database-sql-to-operate-database-in-go/
