[解析](https://zhuanlan.zhihu.com/p/597591601)
# 一、bolt 基础使用



### 安装中

要开始使用 Bolt，请安装 Go 并运行`go get`：

```
$ go get github.com/boltdb/bolt/...
```

这将检索库并将`bolt`命令行实用程序安装到您的`$GOBIN`路径中。

### 打开数据库

Bolt 中的顶级对象是`DB`. 它表示为磁盘上的单个文件，并表示数据的一致快照。

要打开数据库，只需使用以下`bolt.Open()`函数：

```
package main

import (
	"log"

	"github.com/boltdb/bolt"
)

func main() {
	// Open the my.db data file in your current directory.
	// It will be created if it doesn't exist.
	db, err := bolt.Open("my.db", 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	...
}
```

请注意，Bolt 会获取数据文件的文件锁，因此多个进程无法同时打开同一个数据库。打开一个已经打开的 Bolt 数据库将导致它挂起，直到另一个进程将其关闭。为了防止无限期等待，您可以向函数传递超时选项`Open()`：

```
db, err := bolt.Open("my.db", 0600, &bolt.Options{Timeout: 1 * time.Second})
```

### 事务

==Bolt 一次只允许一个读写事务，但一次允许任意数量的只读事务。每个事务都具有与事务启动时存在的数据一致的视图。==

单个事务以及从它们创建的所有对象（例如存储桶、键）都不是线程安全的。要处理多个 Goroutine 中的数据，您必须为每个 Goroutine 启动一项事务，或者使用锁定来确保一次只有一个 Goroutine 访问事务。从创建事务`DB`是线程安全的。

只读事务和读写事务不应相互依赖，并且通常不应在同一个 goroutine 中同时打开。这可能会导致死锁，因为读写事务需要定期重新映射数据文件，但在只读事务打开时无法这样做。

#### 读写事务

要启动读写事务，可以使用以下`DB.Update()`函数：

```
err := db.Update(func(tx *bolt.Tx) error {
	...
	return nil
})
```

在闭包内，您可以看到一致的数据库视图。`nil`您通过最后返回来提交交易。您还可以通过返回错误随时回滚事务。所有数据库操作都允许在读写事务内进行。

请务必检查返回错误，因为它会报告任何可能导致事务无法完成的磁盘故障。如果您在闭包中返回错误，它将被传递。

#### 只读事务

要启动只读事务，您可以使用以下`DB.View()`函数：

```
err := db.View(func(tx *bolt.Tx) error {
	...
	return nil
})
```

您还可以在此闭包中获得数据库的一致视图，但是，只读事务中不允许进行任何更改操作。您只能在只读事务中检索存储桶、检索值和复制数据库。

#### 批量读写事务

每个`DB.Update()`等待磁盘提交写入。通过将多个更新与函数结合起来可以最大限度地减少这种开销`DB.Batch()` ：

```
err := db.Batch(func(tx *bolt.Tx) error {
	...
	return nil
})
```

并发批处理调用会机会性地组合成更大的事务。Batch 仅当有多个 goroutine 调用时才有用。

权衡是，`Batch`如果部分事务失败，则可以多次调用给定函数。该函数必须是幂等的，并且副作用必须仅在从 成功返回后才生效`DB.Batch()`。

例如：不要显示函数内部的消息，而是在封闭范围内设置变量：

```
var id uint64
err := db.Batch(func(tx *bolt.Tx) error {
	// Find last key in bucket, decode as bigendian uint64, increment
	// by one, encode back to []byte, and add new key.
	...
	id = newValue
	return nil
})
if err != nil {
	return ...
}
fmt.Println("Allocated ID %d", id)
```

#### 手动管理交易

和函数是函数`DB.View()`的`DB.Update()`包装器`DB.Begin()` 。这些辅助函数将启动事务、执行函数，然后在返回错误时安全地关闭事务。这是使用 Bolt 交易的推荐方式。

但是，有时您可能需要手动开始和结束交易。您可以`DB.Begin()`直接使用该功能，但**请**务必关闭交易。

```
// Start a writable transaction.
tx, err := db.Begin(true)
if err != nil {
    return err
}
defer tx.Rollback()

// Use the transaction...
_, err := tx.CreateBucket([]byte("MyBucket"))
if err != nil {
    return err
}

// Commit the transaction and check for error.
if err := tx.Commit(); err != nil {
    return err
}
```

第一个参数`DB.Begin()`是一个布尔值，说明事务是否应该可写。

### 使用水桶

存储桶是数据库中键/值对的集合。桶中的所有键必须是唯一的。您可以使用以下`DB.CreateBucket()` 函数创建存储桶：

```
db.Update(func(tx *bolt.Tx) error {
	b, err := tx.CreateBucket([]byte("MyBucket"))
	if err != nil {
		return fmt.Errorf("create bucket: %s", err)
	}
	return nil
})
```

仅当存储桶不存在时，您也可以使用该 `Tx.CreateBucketIfNotExists()`函数创建存储桶。打开数据库后，为所有顶级存储桶调用此函数是一种常见模式，这样您就可以保证它们在未来的事务中存在。

要删除存储桶，只需调用该`Tx.DeleteBucket()`函数即可。

### 使用键/值对

要将键/值对保存到存储桶，请使用以下`Bucket.Put()`函数：

```
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	err := b.Put([]byte("answer"), []byte("42"))
	return err
})
```

`"answer"`这会将键的值设置为存储桶`"42"`中的值`MyBucket` 。要检索该值，我们可以使用以下`Bucket.Get()`函数：

```
db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	v := b.Get([]byte("answer"))
	fmt.Printf("The answer is: %s\n", v)
	return nil
})
```

该`Get()`函数不会返回错误，因为它的操作保证有效（除非出现某种系统故障）。如果该键存在，那么它将返回其字节切片值。如果不存在则返回`nil`。请务必注意，您可以将零长度值设置为与不存在的密钥不同的密钥。

使用该`Bucket.Delete()`函数从存储桶中删除密钥。

请注意，返回的值`Get()`仅在交易打开时有效。如果您需要在事务之外使用值，则必须使用`copy()`将其复制到另一个字节片。

### 桶的自动递增整数

通过使用该`NextSequence()`函数，您可以让 Bolt 确定一个序列，该序列可用作键/值对的唯一标识符。请参阅下面的示例。

```
// CreateUser saves u to the store. The new user ID is set on u once the data is persisted.
func (s *Store) CreateUser(u *User) error {
    return s.db.Update(func(tx *bolt.Tx) error {
        // Retrieve the users bucket.
        // This should be created when the DB is first opened.
        b := tx.Bucket([]byte("users"))

        // Generate ID for the user.
        // This returns an error only if the Tx is closed or not writeable.
        // That can't happen in an Update() call so I ignore the error check.
        id, _ := b.NextSequence()
        u.ID = int(id)

        // Marshal user data into bytes.
        buf, err := json.Marshal(u)
        if err != nil {
            return err
        }

        // Persist bytes to users bucket.
        return b.Put(itob(u.ID), buf)
    })
}

// itob returns an 8-byte big endian representation of v.
func itob(v int) []byte {
    b := make([]byte, 8)
    binary.BigEndian.PutUint64(b, uint64(v))
    return b
}

type User struct {
    ID int
    ...
}
```

### 迭代键

Bolt 将其密钥按字节排序顺序存储在存储桶中。这使得对这些键的顺序迭代变得非常快。要迭代键，我们将使用 `Cursor`：

```
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	b := tx.Bucket([]byte("MyBucket"))

	c := b.Cursor()

	for k, v := c.First(); k != nil; k, v = c.Next() {
		fmt.Printf("key=%s, value=%s\n", k, v)
	}

	return nil
})
```

光标允许您移动到按键列表中的特定点，并一次向前或向后移动一个按键。

光标可以使用以下功能：

```
First()  Move to the first key.
Last()   Move to the last key.
Seek()   Move to a specific key.
Next()   Move to the next key.
Prev()   Move to the previous key.
```

这些函数中的每一个都有一个返回签名`(key []byte, value []byte)`。当您迭代到光标末尾时，`Next()`将返回一个 `nil`键。 在调用 或 之前，您必须使用`First()`、`Last()`、 或寻求职位。如果您不寻找某个位置，那么这些函数将返回一个键。`Seek()``Next()``Prev()``nil`

迭代过程中，如果 key 为 non-`nil`但 value 为`nil`，则表示 key 引用的是一个 Bucket，而不是一个 Value。用于`Bucket.Bucket()`访问子存储桶。

#### 前缀扫描

要迭代键前缀，您可以组合`Seek()`和`bytes.HasPrefix()`：

```
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	c := tx.Bucket([]byte("MyBucket")).Cursor()

	prefix := []byte("1234")
	for k, v := c.Seek(prefix); k != nil && bytes.HasPrefix(k, prefix); k, v = c.Next() {
		fmt.Printf("key=%s, value=%s\n", k, v)
	}

	return nil
})
```

#### 范围扫描

另一个常见的用例是扫描一定范围（例如时间范围）。如果您使用可排序的时间编码（例如 RFC3339），那么您可以像这样查询特定的日期范围：

```
db.View(func(tx *bolt.Tx) error {
	// Assume our events bucket exists and has RFC3339 encoded time keys.
	c := tx.Bucket([]byte("Events")).Cursor()

	// Our time range spans the 90's decade.
	min := []byte("1990-01-01T00:00:00Z")
	max := []byte("2000-01-01T00:00:00Z")

	// Iterate over the 90's.
	for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
		fmt.Printf("%s: %s\n", k, v)
	}

	return nil
})
```

请注意，虽然 RFC3339 是可排序的，但 RFC3339Nano 的 Golang 实现在小数点后不使用固定位数，因此不可排序。

#### 遍历

`ForEach()`如果您知道将迭代存储桶中的所有键，您也可以使用该函数：

```
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	b := tx.Bucket([]byte("MyBucket"))

	b.ForEach(func(k, v []byte) error {
		fmt.Printf("key=%s, value=%s\n", k, v)
		return nil
	})
	return nil
})
```

请注意，其中的键和值`ForEach()`仅在交易打开时有效。如果需要在事务之外使用键或值，则必须使用`copy()`将其复制到另一个字节片。

### 嵌套桶

您还可以将存储桶存储在键中以创建嵌套存储桶。该 API 与对象上的存储桶管理 API 相同`DB`：

```
func (*Bucket) CreateBucket(key []byte) (*Bucket, error)
func (*Bucket) CreateBucketIfNotExists(key []byte) (*Bucket, error)
func (*Bucket) DeleteBucket(key []byte) error
```

假设您有一个多租户应用程序，其中根级别存储桶是帐户存储桶。这个桶里面有一系列帐户，它们本身就是桶。在序列存储桶内，您可以有许多与帐户本身相关的存储桶（用户、注释等），将信息隔离为逻辑分组。

```
// createUser creates a new user in the given account.
func createUser(accountID int, u *User) error {
    // Start the transaction.
    tx, err := db.Begin(true)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Retrieve the root bucket for the account.
    // Assume this has already been created when the account was set up.
    root := tx.Bucket([]byte(strconv.FormatUint(accountID, 10)))

    // Setup the users bucket.
    bkt, err := root.CreateBucketIfNotExists([]byte("USERS"))
    if err != nil {
        return err
    }

    // Generate an ID for the new user.
    userID, err := bkt.NextSequence()
    if err != nil {
        return err
    }
    u.ID = userID

    // Marshal and save the encoded user.
    if buf, err := json.Marshal(u); err != nil {
        return err
    } else if err := bkt.Put([]byte(strconv.FormatUint(u.ID, 10)), buf); err != nil {
        return err
    }

    // Commit the transaction.
    if err := tx.Commit(); err != nil {
        return err
    }

    return nil
}
```

### 数据库备份

Bolt 是单个文件，因此很容易备份。您可以使用该`Tx.WriteTo()` 函数将数据库的一致视图写入写入器。如果您从只读事务中调用它，它将执行热备份并且不会阻止您的其他数据库读写。

默认情况下，它将使用常规文件句柄，该句柄将利用操作系统的页面缓存。[`Tx`](https://godoc.org/github.com/boltdb/bolt#Tx) 有关优化大于 RAM 数据集的信息，请参阅文档。

一种常见的用例是通过 HTTP 进行备份，因此您可以使用诸如`cURL`数据库备份之类的工具：

```
func BackupHandleFunc(w http.ResponseWriter, req *http.Request) {
	err := db.View(func(tx *bolt.Tx) error {
		w.Header().Set("Content-Type", "application/octet-stream")
		w.Header().Set("Content-Disposition", `attachment; filename="my.db"`)
		w.Header().Set("Content-Length", strconv.Itoa(int(tx.Size())))
		_, err := tx.WriteTo(w)
		return err
	})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

然后您可以使用以下命令进行备份：

```
$ curl http://localhost/backup > my.db
```

或者您可以打开浏览器`http://localhost/backup`，它会自动下载。

如果您想备份到另一个文件，可以使用`Tx.CopyFile()`辅助功能。

### 统计数据

数据库会对其执行的许多内部操作进行运行计数，以便您可以更好地了解正在发生的情况。通过在两个时间点获取这些统计数据的快照，我们可以看到在该时间范围内执行了哪些操作。

例如，我们可以启动一个 goroutine 每 10 秒记录一次统计数据：

```
go func() {
	// Grab the initial stats.
	prev := db.Stats()

	for {
		// Wait for 10s.
		time.Sleep(10 * time.Second)

		// Grab the current stats and diff them.
		stats := db.Stats()
		diff := stats.Sub(&prev)

		// Encode stats to JSON and print to STDERR.
		json.NewEncoder(os.Stderr).Encode(diff)

		// Save stats for the next loop.
		prev = stats
	}
}()
```

将这些统计信息通过管道传输到 statsd 等服务进行监控或提供执行固定长度样本的 HTTP 端点也很有用。

### 只读模式

有时创建共享的只读 Bolt 数据库很有用。为此，请`Options.ReadOnly`在打开数据库时设置标志。只读模式使用共享锁允许多个进程从数据库中读取数据，但会阻止任何进程以读写模式打开数据库。

```
db, err := bolt.Open("my.db", 0666, &bolt.Options{ReadOnly: true})
if err != nil {
	log.Fatal(err)
}
```



# 二、代码示例

```
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	bolt "go.etcd.io/bbolt"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	newBoltDB()
	writeData()
	boltDB()

}

// open的第三个参数说明
/*
	// Options represents the options that can be set when opening a database.
	type Options struct {
		// Timeout is the amount of time to wait to obtain a file lock.
		// When set to zero it will wait indefinitely. This option is only
		// available on Darwin and Linux.
		//超时是指等待获得文件锁定的时间。
		//当设置为零时，它将无限期等待。此选项仅
		//可在Darwin和Linux上使用
		Timeout time.Duration

		// Sets the DB.NoGrowSync flag before memory mapping the file.
		//在内存映射文件之前设置DB.NoGrowSync标志。
		NoGrowSync bool

		// Do not sync freelist to disk. This improves the database write performance
		// under normal operation, but requires a full database re-sync during recovery.
		//
		//不要将自由列表同步到磁盘。这提高了数据库写入性能
		//在正常操作下，但需要在恢复期间重新同步完整的数据库。
		NoFreelistSync bool

		// PreLoadFreelist sets whether to load the free pages when opening
		// the db file. Note when opening db in write mode, bbolt will always
		// load the free pages.
		// 设置打开时是否初始化空闲pages
		// 数据库文件。请注意，在写入模式下打开数据库时，bbolt将始终加载free pages
		PreLoadFreelist bool

		// FreelistType sets the backend freelist type. There are two options. Array which is simple but endures
		// dramatic performance degradation if database is large and fragmentation in freelist is common.
		// The alternative one is using hashmap, it is faster in almost all circumstances
		// but it doesn't guarantee that it offers the smallest page id available. In normal case it is safe.
		// The default type is array
		//FreelistType设置后端自由列表类型。有两种选择。简单而持久的阵列
		//若数据库很大并且自由列表中的碎片很常见，则性能会急剧下降。
		//另一种是使用hashmap，它在几乎所有情况下都更快
		//但它不能保证提供最小的可用页面id。在正常情况下是安全的。
		//默认类型为数组
		FreelistType FreelistType

		// Open database in read-only mode. Uses flock(..., LOCK_SH |LOCK_NB) to
		// grab a shared lock (UNIX).
		//以只读模式打开数据库。使用flock（…，LOCK_SH|LOCK_NB）
		//获取共享锁（UNIX）。
		ReadOnly bool

		// Sets the DB.MmapFlags flag before memory mapping the file.
		//在内存映射文件之前设置DB.MmapFlags标志。
		MmapFlags int

		// InitialMmapSize is the initial mmap size of the database
		// in bytes. Read transactions won't block write transaction
		// if the InitialMmapSize is large enough to hold database mmap
		// size. (See DB.Begin for more information)
		//
		// If <=0, the initial map size is 0.
		// If initialMmapSize is smaller than the previous database size,
		// it takes no effect.
		//InitialMmapSize是数据库的初始mmap大小
		//以字节为单位。读取事务不会阻止写入事务
		//如果InitialMmapSize足够大以容纳数据库mmap尺寸。（有关详细信息，请参阅DB.Begin）
		//如果<=0，则初始贴图大小为0。
		//如果initialMmapSize小于先前的数据库大小，它没有效果。
		InitialMmapSize int

		// PageSize overrides the default OS page size.
		////PageSize覆盖默认的操作系统页面大小。
		PageSize int

		// NoSync sets the initial value of DB.NoSync. Normally this can just be
		// set directly on the DB itself when returned from Open(), but this option
		// is useful in APIs which expose Options but not the underlying DB.
		////NoSync设置DB.NoSync的初
		//从Open（）返回时直接在DB本身上设置，但此选项
		//在公开Options但不公开底层DB的API中非常有用。
		NoSync bool

		// OpenFile is used to open files. It defaults to os.OpenFile. This option
		// is useful for writing hermetic tests.
		//OpenFile用于打开文件。默认为os.OpenFile。此选项
		//对于编写气密测试非常有用。
		OpenFile func(string, int, os.FileMode) (*os.File, error)

		// Mlock locks database file in memory when set to true.
		// It prevents potential page faults, however
		// used memory can't be reclaimed. (UNIX only)
		//当设置为true时，Mlock会锁定内存中的数据库文件。
		//但是，它可以防止潜在的页面错误
		//用过的内存无法回收。（仅限UNIX）
		Mlock bool
	}

*/
var world = []byte("world")
var db *bolt.DB

func newBoltDB() {
	var err error
	db, err = bolt.Open("./boltdb", 0666, nil)
	if err != nil {
		panic(err)
	}
	//defer db.Close()

}

func writeData() {
	key := []byte("hello")
	value := []byte("Hello World!")

	// store some data
	err := db.Update(func(tx *bolt.Tx) error {
		bucket, err := tx.CreateBucketIfNotExists(world)
		if err != nil {
			fmt.Println(111)
			panic(err)

		}

		err = bucket.Put(key, value)
		err = bucket.Put([]byte("hello1"), []byte("Hello World1!"))
		err = bucket.Put([]byte("hello2"), []byte("Hello World2!"))
		err = bucket.Put([]byte("ccccc"), []byte("aaaa"))
		if err != nil {
			panic(err)

		}
		return nil
	})

	if err != nil {
		panic(err)

	}

}

func boltDB() {
	var err error
	key := []byte("hello")

	// retrieve the data
	err = db.View(func(tx *bolt.Tx) error {
		bucket := tx.Bucket(world)
		if bucket == nil {
			panic("Bucket %q not found!")
		}

		val := bucket.Get(key)
		fmt.Println(string(val))

		return nil
	})

	if err != nil {
		panic(err)

	}

	//游标查询
	fmt.Println("游标查询")
	err = db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		b := tx.Bucket(world)

		c := b.Cursor()

		for k, v := c.First(); k != nil; k, v = c.Next() {
			fmt.Printf("key=%s, value=%s\n", k, v)
		}

		return nil
	})
	//游标查询
	//key=ccccc, value=aaaa
	//key=hello, value=Hello World!
	//key=hello1, value=Hello World1!
	//key=hello2, value=Hello World2!

	//前缀查询
	fmt.Println()
	fmt.Println("前缀查询")
	db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		c := tx.Bucket(world).Cursor()

		prefix := []byte("hello")
		for k, v := c.Seek(prefix); k != nil && bytes.HasPrefix(k, prefix); k, v = c.Next() {
			fmt.Printf("key=%s, value=%s\n", k, v)
		}

		return nil
	})
	//前缀查询
	//key=hello, value=Hello World!
	//key=hello1, value=Hello World1!
	//key=hello2, value=Hello World2!

	fmt.Println()
	//范围查询
	fmt.Println("范围查询")
	db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		c := tx.Bucket(world).Cursor()

		// Our time range spans the 90's decade.
		min := []byte("hello1")
		max := []byte("hello2")

		// Iterate over the 90's.
		for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
			fmt.Printf("%s: %s\n", k, v)
		}

		return nil

	})

	db.Batch(func(tx *bolt.Tx) error {
		// Find last key in bucket, decode as bigendian uint64, increment
		// by one, encode back to []byte, and add new key.
		return nil
	})

	fmt.Println()
	//范围查询
	fmt.Println("范围查询")
	db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		c := tx.Bucket(world).Cursor()

		// Our time range spans the 90's decade.
		min := []byte("hello1")
		max := []byte("hello2")

		// Iterate over the 90's.
		for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
			fmt.Printf("%s: %s\n", k, v)
		}

		return nil

	})

	//遍历buket
	fmt.Println()
	fmt.Println("遍历buket")
	db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		b := tx.Bucket(world)
		b.ForEach(func(k, v []byte) error {
			fmt.Printf("key=%s, value=%s\n", k, v)
			return nil
		})
		return nil
	})
	//遍历buket
	//key=ccccc, value=aaaa
	//key=hello, value=Hello World!
	//key=hello1, value=Hello World1!
	//key=hello2, value=Hello World2!

	//buket操作
	//func (*Bucket) CreateBucket(key []byte) (*Bucket, error)
	//func (*Bucket) CreateBucketIfNotExists(key []byte) (*Bucket, error)
	//func (*Bucket) DeleteBucket(key []byte) error

	//事务
	fmt.Println()
	fmt.Println("嵌套bucket")
	createUser(100, &User{})

	fmt.Println()
	fmt.Println("数据库备份")
	//copy()
	//BackupHandleFunc()

	fmt.Println()
	fmt.Println("统计数据")
	go func() {
		// Grab the initial stats.
		prev := db.Stats()

		for {
			// Wait for 10s.
			time.Sleep(10 * time.Second)

			// Grab the current stats and diff them.
			stats := db.Stats()
			diff := stats.Sub(&prev)

			// Encode stats to JSON and print to STDERR.
			json.NewEncoder(os.Stderr).Encode(diff)

			// Save stats for the next loop.
			prev = stats
		}
	}()
	//{"FreePageN":0,"PendingPageN":3,"FreeAlloc":49152,"FreelistInuse":40,"TxN":0,"OpenTxN":0,"TxStats":{"PageCount":0,"PageAlloc":0,"CursorCount":0,"NodeCount":0,"NodeDeref":0,"Rebalance":0,"RebalanceTime":0,"Split":0,"Spill":0,"SpillTime":0,"Write":0,"WriteTime":0}}
}

type User struct {
	ID uint64
}

// createUser creates a new user in the given account.
func createUser(accountID uint64, u *User) error {
	// Start the transaction.
	//是否开启写事务
	tx, err := db.Begin(true)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	// Retrieve the root bucket for the account.
	// Assume this has already been created when the account was set up.
	root := tx.Bucket(world)

	// Setup the users bucket.
	bkt, err := root.CreateBucketIfNotExists([]byte("USERS"))
	if err != nil {
		return err
	}

	// Generate an ID for the new user.
	//NextSequence为bucket返回一个自动递增的整数。
	userID, err := bkt.NextSequence()
	fmt.Println("递增的userId", userID)
	if err != nil {
		return err
	}
	u.ID = userID

	// Marshal and save the encoded user.
	if buf, err := json.Marshal(u); err != nil {
		return err
	} else if err := bkt.Put([]byte(strconv.FormatUint(u.ID, 10)), buf); err != nil {
		return err
	}

	// Commit the transaction.
	if err := tx.Commit(); err != nil {
		return err
	}

	//遍历嵌套的bucket
	fmt.Println()
	fmt.Println("遍历嵌套的bucket")
	db.View(func(tx *bolt.Tx) error {
		// Assume bucket exists and has keys
		b := tx.Bucket(world)
		b.ForEach(func(k, v []byte) error {
			fmt.Printf("key=%s, value=%s\n", k, v)
			return nil
		})
		//遍历嵌套的bucket
		//key=USERS, value=
		//key=ccccc, value=aaaa
		//key=hello, value=Hello World!
		//key=hello1, value=Hello World1!
		//key=hello2, value=Hello World2!

		fmt.Println()
		fmt.Println("遍历嵌套的bucket")
		u := b.Bucket([]byte("USERS"))
		u.ForEach(func(k, v []byte) error {
			fmt.Printf("嵌套的key=%s, value=%s\n", k, v)
			return nil
		})
		//遍历嵌套的bucket
		//嵌套的key=1, value={"ID":1}
		//嵌套的key=10, value={"ID":10}
		//嵌套的key=11, value={"ID":11}
		//嵌套的key=12, value={"ID":12}
		//嵌套的key=13, value={"ID":13}
		//嵌套的key=2, value={"ID":2}
		//嵌套的key=3, value={"ID":3}
		//嵌套的key=4, value={"ID":4}
		//嵌套的key=5, value={"ID":5}
		//嵌套的key=6, value={"ID":6}
		//嵌套的key=7, value={"ID":7}
		//嵌套的key=8, value={"ID":8}
		//嵌套的key=9, value={"ID":9}

		return nil
	})

	return nil
}

func BackupHandleFunc(w http.ResponseWriter, req *http.Request) {
	err := db.View(func(tx *bolt.Tx) error {
		w.Header().Set("Content-Type", "application/octet-stream")
		w.Header().Set("Content-Disposition", `attachment; filename="boltdb"`)
		w.Header().Set("Content-Length", strconv.Itoa(int(tx.Size())))
		_, err := tx.WriteTo(w)
		return err
	})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}

func copy() {
	err := db.View(func(tx *bolt.Tx) error {
		tx.CopyFile("./boltdb_back", 666)
		return nil
	})
	if err != nil {
		panic(err)
	}
}

```



# 三、K/V对比

https://blog.csdn.net/huxinglixing/article/details/116156322

https://github.com/huxinhuxin/kvtest

```
一万

[./kvtest -f ./ -a 10000]
2023/06/28 11:31:23 begin test bolt !!!!!!!!!!!!!
.//bolt.db
2023/06/28 11:31:23 sequence write begin
2023/06/28 11:31:26 sequence write end, cost time  3.069157471s
2023/06/28 11:31:26 randRead  begin
2023/06/28 11:31:26 randRead err count =  0
2023/06/28 11:31:26 randdel  begin
2023/06/28 11:31:30 randdel err count =  0
2023/06/28 11:31:30 randdel  end, cost time  3.683270353s
2023/06/28 11:31:30 randwrite  begin
2023/06/28 11:31:34 randwrite  end, cost time  4.478815804s
2023/06/28 11:31:34 readall  begin
2023/06/28 11:31:34 readall get count =  7708
2023/06/28 11:31:34 readall  end, cost time  1.052078ms
2023/06/28 11:31:34 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:34 db close ,total cost time  11.312932804s
2023/06/28 11:31:34 end test bolt !!!!!!!!!!!!!
2023/06/28 11:31:34 begin test pebble !!!!!!!!!!!!!
.//pebble
2023/06/28 11:31:34 sequence write begin
2023/06/28 11:31:35 sequence write end, cost time  853.149329ms
2023/06/28 11:31:35 randRead  begin
2023/06/28 11:31:35 randRead err count =  0
2023/06/28 11:31:35 randdel  begin
2023/06/28 11:31:36 randdel err count =  0
2023/06/28 11:31:36 randdel  end, cost time  570.763802ms
2023/06/28 11:31:36 randwrite  begin
2023/06/28 11:31:37 randwrite  end, cost time  722.175197ms
2023/06/28 11:31:37 readall  begin
2023/06/28 11:31:37 readall get count =  7629
2023/06/28 11:31:37 readall  end, cost time  73.831722ms
2023/06/28 11:31:37 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:37 db close ,total cost time  2.327886891s
2023/06/28 11:31:37 end test pebble !!!!!!!!!!!!!
2023/06/28 11:31:37 begin test badger !!!!!!!!!!!!!
.//badger
badger 2023/06/28 11:31:37 INFO: All 0 tables opened in 0s
badger 2023/06/28 11:31:37 INFO: Discard stats nextEmptySlot: 0
badger 2023/06/28 11:31:37 INFO: Set nextTxnTs to 0
2023/06/28 11:31:37 sequence write begin
2023/06/28 11:31:41 sequence write end, cost time  3.79018787s
2023/06/28 11:31:41 randRead  begin
2023/06/28 11:31:41 randRead err count =  0
2023/06/28 11:31:41 randdel  begin
2023/06/28 11:31:42 randdel err count =  0
2023/06/28 11:31:42 randdel  end, cost time  1.198206989s
2023/06/28 11:31:42 randwrite  begin
2023/06/28 11:31:46 randwrite  end, cost time  3.77515386s
2023/06/28 11:31:46 readall  begin
2023/06/28 11:31:46 readall get count =  7725
2023/06/28 11:31:46 readall  end, cost time  16.04366ms
badger 2023/06/28 11:31:46 INFO: Lifetime L0 stalled for: 0s
badger 2023/06/28 11:31:46 INFO:
Level 0 [ ]: NumTables: 01. Size: 489 KiB of 0 B. Score: 0.00->0.00 Target FileSize: 64 MiB
Level 1 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 2 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 3 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 4 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 5 [ ]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level 6 [B]: NumTables: 00. Size: 0 B of 10 MiB. Score: 0.00->0.00 Target FileSize: 2.0 MiB
Level Done
2023/06/28 11:31:46 10000 counts sequence write, rand read,rand del,rand write,get all
2023/06/28 11:31:46 db close ,total cost time  8.88882326s
2023/06/28 11:31:46 end test badger !!!!!!!!!!!!!
--------------- bolt --------------

seque write cost time:        3.069157471s
rand read cost time:          75.199069ms
rand del cost time:           3.683270353s
rand write cost:              4.478815804s
getall cost:                  1.052078ms
total cost:                   11.312932804s

-----------------------------------
--------------- pebble --------------

seque write cost time:        853.149329ms
rand read cost time:          91.731766ms
rand del cost time:           570.763802ms
rand write cost:              722.175197ms
getall cost:                  73.831722ms
total cost:                   2.327886891s

-----------------------------------
--------------- badger --------------

seque write cost time:        3.79018787s
rand read cost time:          73.084294ms
rand del cost time:           1.198206989s
rand write cost:              3.77515386s
getall cost:                  16.04366ms
total cost:                   8.88882326s

-----------------------------------


10万
--------------- pebble --------------

seque write cost time:        6.341589103s
rand read cost time:          1.342146869s
rand del cost time:           5.2483614s
rand write cost:              11.333176547s
getall cost:                  692.895844ms
total cost:                   25.07367853s

-----------------------------------
--------------- badger --------------

seque write cost time:        35.472496258s
rand read cost time:          705.603311ms
rand del cost time:           9.636414299s
rand write cost:              35.15595596s
getall cost:                  146.364165ms
total cost:                   1m21.403964226s

-----------------------------------
--------------- bolt --------------

seque write cost time:        34.574638108s
rand read cost time:          914.090331ms
rand del cost time:           2m16.546609363s
rand write cost:              2m9.629712722s
getall cost:                  13.770647ms
total cost:                   5m1.710829044s

-----------------------------------
```



# 三、boltdbweb

[github](https://github.com/evnix/boltdbweb)

下载代码 
