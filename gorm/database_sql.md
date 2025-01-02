# 一、抽象接口定义



标准库 database/sql 定义了 go 语言通用的结构化查询流程框架，其对接的数据库类型是灵活多变的，如 mysql、sqlite、oracle 等. 因此**在 database/sql 中，与具体数据库交互的细节内容统一托付给一个抽象的数据库驱动模块，在其中声明好一套适用于各类关系型数据库的统一规范，将各个关键节点定义成抽象的 interface，由具体类型的数据库完成数据库驱动模块的实现**，然后将其注入到 database/sql 的大框架之中.

database/sql 关于数据库驱动模块下各核心 interface 主要包括：

- **Connector：抽象的数据库连接器**，需要具备创建数据库连接以及返回从属的数据库驱动的能力
- **Driver：抽象的数据库驱动**，具备创建数据库连接的能力
- **Conn：抽象的数据库连接**，具备预处理 sql 以及开启事务的能力
- **Tx：抽象的事务**，具备提交和回滚的能力
- **Statement：抽象的请求预处理状态**. 具备实际执行 sql 并返回执行结果的能力
- **Result/Row**：**抽象的 sql 执行结果**

![database_sql接口关系](D:\gocode\github\blog\images\database_sql接口关系.png)

## 1、 抽象接口

由具体的数据库类型提供具体的实现版本

```go
// 抽象的数据库连接器
type Connector interface {
    // 获取一个数据库连接
    Connect(context.Context) (Conn, error)
    // 获取数据库驱动
    Driver() Driver
}

// 抽象的数据库连接
type Conn interface {
    // 预处理 sql
    Prepare(query string) (Stmt, error)

    // 关闭连接   
    Close() error

    // 开启事务
    Begin() (Tx, error)
}

// 抽象的请求预处理状态接口
type Stmt interface {
    // 关闭
    Close() error

    // 返回 sql 中存在的可变参数数量
    NumInput() int

    // 执行操作类型的 sql
    Exec(args []Value) (Result, error)

    // 执行查询类型的 sql
    Query(args []Value) (Rows, error)
}

// 抽象的执行结果接口
type Result interface {
    // 最后一笔插入数据的主键
    LastInsertId() (int64, error)

    // 操作影响的行数
    RowsAffected() (int64, error)
}
type Rows interface {
    // 返回所有列名
    Columns() []string

    // 关闭 rows 迭代器
    Close() error

    // 遍历
    Next(dest []Value) error
}

// 抽象的事务接口
type Tx interface {
    // 提交事务
    Commit() error
    // 回滚事务
    Rollback() error
}

// 抽象的数据库驱动
type Driver interface {
    // 开启一个新的数据库连接
    Open(name string) (Conn, error)
}
```



## 2、创建数据库实例

先看下 db 实例的结构体字段：

```go
type DB struct {
	// Total time waited for new connections.记录等待新连接的总时长,主要用于统计和调优连接池的性能
	waitDuration atomic.Int64

	connector driver.Connector	// 用于创建新连接的 Connector 对象,通常由数据库驱动实现
	
	numClosed atomic.Uint64	// 用于记录数据库连接池中已经关闭的连接数。Stmt.openStmt 会检查这个字段，以清理关闭的连接。

	mu           sync.Mutex    
	freeConn     []*driverConn // 连接池中的空闲连接 ordered by returnedAt oldest to newest
	connRequests connRequestSet	// 存储等待的连接请求
	numOpen      int // 表示当前打开（包括正在打开和已打开）的连接数
	
	openerCh          chan struct{}		// 通知连接池需要打开新连接,是一个不携带数据的通道，通常用于实现信号通知机制
	closed            bool				// db 实例是否已关闭
	dep               map[finalCloser]depSet	// 存储依赖于 finalCloser 类型的对象，管理对象之间的依赖关系，确保在关闭时正确清理资源
	lastPut           map[*driverConn]string // 最后一次将连接放入连接池时的堆栈信息；debug only
	maxIdleCount      int                    // 设置空闲连接池的最大连接数，为零表示使用默认值，如果为负数则表示不允许空闲连接
	maxOpen           int                    // 允许的最大打开连接数，<= 0 表示不限制最大连接数
	maxLifetime       time.Duration          // 连接的最大生命周期
	maxIdleTime       time.Duration          // 连接在空闲状态下的最长存活时间
	cleanerCh         chan struct{}
	waitCount         int64 // 请求连接时的等待次数
	maxIdleClosed     int64 // 因空闲连接超出最大数目而关闭的连接数
	maxIdleTimeClosed int64 // 因空闲时间过长而被关闭的连接数
	maxLifetimeClosed int64 // 因连接生命周期到期而被关闭的连接数

	stop func() 	// 关闭连接池时调用，用于清理相关资源并停止连接创建操作
}
```





以一个例子开始对 database/sql 的源码解读

```go
import (
    "context"
    "database/sql"
    "testing"


    // 注册 mysql 数据库驱动
    _ "github.com/go-sql-driver/mysql"
)


type user struct {
    UserID int64
}


func Test_sql(t *testing.T) {
    // 创建 db 实例
    db, err := sql.Open("mysql", "username:passpord@(ip:port)/database")
    if err != nil {
        t.Error(err)
        return
    }
    
    // 执行 sql
    ctx := context.Background()
    row := db.QueryRowContext(ctx, "SELECT user_id FROM user WHERE id = 1")
    if row.Err() != nil {
        t.Error(err)
        return
    }


    // 解析结果
    var u user
    if err = row.Scan(&u.UserID); err != nil {
        t.Error(err)
        return
    }
    t.Log(u.UserID)
}
```



从 `sql.Open()` 函数开始，`sql.Open` 不会立即建立与数据库的连接。连接是在实际需要时创建的，例如首次执行查询/或者调用 `DB.Ping`

```go
var driversMu sync.RWMutex // 全局变量，读写锁
//go:linkname drivers
var drivers = make(map[string]driver.Driver)

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

具体执行流程：

1、加读写锁访问全局资源 `drivers map`，从中取出对应的驱动器实现类，各个不同的数据库有其对应的实现类由，通过 `Register` 函数注册到 database/sql 的基础框架中。

2、若对应的驱动实现了连接器对象 `Connector` ，则获取并创建 db 实例

3、调用 OpenDB 函数，开始创建 db 实例

> Register 函数：
>
> ```GO
> func Register(name string, driver driver.Driver) {
> 	driversMu.Lock()
> 	defer driversMu.Unlock()
> 	if driver == nil {
> 		panic("sql: Register driver is nil")
> 	}
> 	if _, dup := drivers[name]; dup {
> 		panic("sql: Register called twice for driver " + name)
> 	}
> 	drivers[name] = driver
> }
> ```

其中 driver.DriverContext 也是一个接口类型，具体为:

```go
type DriverContext interface {
	// OpenConnector must parse the name in the same format that Driver.Open
	// parses the name parameter.
	OpenConnector(name string) (Connector, error)
}
```



接着看 OpenDB 函数的实现：

```GO
var connectionRequestQueueSize = 1000000
// OpenDB may just validate its arguments without creating a connection
// to the database. To verify that the data source name is valid, call
// [DB.Ping].
func OpenDB(c driver.Connector) *DB {
	ctx, cancel := context.WithCancel(context.Background())
	db := &DB{
		connector: c,
		openerCh:  make(chan struct{}, connectionRequestQueueSize),
		lastPut:   make(map[*driverConn]string),
		stop:      cancel,
	}

	go db.connectionOpener(ctx)

	return db
}

func (db *DB) connectionOpener(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
        //  等待新的连接请求到达 openerCh 通道。
		case <-db.openerCh:
			db.openNewConnection(ctx)
		}
	}
}
```

1、创建 DB 实例

`connector`: 实现了 `driver.Connector` 接口，用于创建数据库连接。

`openerCh`: 一个缓冲通道，作为连接请求的等待队列。容量由 `connectionRequestQueueSize` 定义，决定了最多可以同时等待的连接请求数

`lastPut`: 记录最后一次放回池中的连接，用于调试或监控。

`connRequests`: 用于管理连接请求，每个请求与唯一的 ID 关联。

`stop`: 通过 `context.CancelFunc` 控制 `connectionOpener` 的停止。

2、启动一个 connectionOpener 协程，注入 ctx，以此来控制该协程的生命周期



在 connectionOpener 协程中，通过 for select 循环监听 done 和 db.openerCh 通道，以此来打开一个新连接，其具体实现为：

```GO
func (db *DB) openNewConnection(ctx context.Context) {
	// maybeOpenNewConnections has already executed db.numOpen++ before it sent
	// on db.openerCh. This function must execute db.numOpen-- if the
	// connection fails or is closed before returning.
	ci, err := db.connector.Connect(ctx)	// 创建一个底层数据库连接
	db.mu.Lock()
	defer db.mu.Unlock()
	if db.closed {
		if err == nil {
			ci.Close()
		}
		db.numOpen--
		return
	}
	if err != nil {
		db.numOpen--
		db.putConnDBLocked(nil, err)
		db.maybeOpenNewConnections()
		return
	}
	dc := &driverConn{
		db:         db,
		createdAt:  nowFunc(),
		returnedAt: nowFunc(),
		ci:         ci,
	}
	if db.putConnDBLocked(dc, err) {
		db.addDepLocked(dc, dc)
	} else {
		db.numOpen--
		ci.Close()
	}
}
```

1）`db.connector.Connect(ctx)` 创建一个底层数据库连接

2）加锁

3）判断 db 实例是否已关闭，即使创建连接成功，也将其关闭，并且将 db 实例的连接打开数减一

4）db 实例未关闭但是连接创建未成功，调用 `putConnDBLocked` 记录错误，尝试触发新的连接创建（通过 `maybeOpenNewConnections`）

5）连接创建成功，将新创建的连接封装为 `driverConn`，包含了元信息（如创建时间）

6）调用 `putConnDBLocked` 尝试将连接放入空闲池中，成功则更新连接的依赖关系；如果放入失败（可能是空闲池已满或其他原因），关闭连接并减少计数



### 2.1 连接池的管理

那么连接是怎么被放入连接池的？连接池是怎么管理的？

`putConnDBLocked` 函数主要用于管理数据库连接池的行为。它的主要作用是处理一个连接请求（connRequest），或者将一个数据库连接（driverConn）放入空闲连接池中。如果既没有满足连接请求，也没有将连接放入空闲池，它将返回 false。

```go
// Satisfy a connRequest or put the driverConn in the idle pool and return true
// or return false.
// putConnDBLocked will satisfy a connRequest if there is one, or it will
// return the *driverConn to the freeConn list if err == nil and the idle
// connection limit will not be exceeded.
// If err != nil, the value of dc is ignored.
// If err == nil, then dc must not equal nil.
// If a connRequest was fulfilled or the *driverConn was placed in the
// freeConn list, then true is returned, otherwise false is returned.
func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
    // 检查 db 是否已关闭
	if db.closed {
		return false
	}
    // 检查是否超过 db 允许打开的最大连接数
	if db.maxOpen > 0 && db.numOpen > db.maxOpen {
		return false
	}
    // 从连接请求队列中随机取一个连接请求，若成功取到则标记 dc 连接已被实现，并通过 chan 发送给连接请求等待方
	if req, ok := db.connRequests.TakeRandom(); ok {
		if err == nil {
			dc.inUse = true
		}
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed {
		if db.maxIdleConnsLocked() > len(db.freeConn) {
			db.freeConn = append(db.freeConn, dc)
			db.startCleanerLocked()
			return true
		}
		db.maxIdleClosed++
	}
	return false
}
```

1）优先处理等待的连接请求

2）如果没有等待的连接请求，并且 `err == nil`（连接有效）且数据库未关闭；

检查空闲连接池（`db.freeConn`）是否未满：

- 调用 `db.maxIdleConnsLocked()` 获取允许的最大空闲连接数。
- 如果空闲池未满，将 `dc` 放入 `db.freeConn`，并启动连接清理器（`startCleanerLocked`），以确保空闲连接在过期时被清理。
- 返回 `true` 表示连接已成功放入空闲池。

如果空闲池已满：

- 增加 `db.maxIdleClosed` 计数，表示因空闲池已满而被关闭的连接数量。



通过 `maybeOpenNewConnections` 函数不断地对连接请求进行创建，直至全部满足

```go
// If there are connRequests and the connection limit hasn't been reached,
// then tell the connectionOpener to open new connections.
func (db *DB) maybeOpenNewConnections() {
	numRequests := db.connRequests.Len()
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
		db.openerCh <- struct{}{}	// 在 openNewConnections 函数监听，获取一个新连接
	}
}
```

