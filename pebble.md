[github](https://iswade.github.io/articles/pebble/#1-pebble)
# 一、pebble参数说明

**BytesPerSync int**

//定期同步sstables，以平滑对磁盘的写入。这选项不提供任何持久性保证，但用于避免如果操作系统自动决定写出一个大区块，则延迟会激增 脏文件系统缓冲区。此选项仅控制SSTable同步；瓦尔同步由WALBytesPerSync控制。默认值为512KB。

**Cache *cache.Cache**

Cache用于缓存sstables中的未压缩块。

默认缓存大小为8MB。



**Cleaner**

清洁器会清理过时的文件。

默认的清洁器使用DeleteCleaner。

有默认值

**Comparer *Comparer**

比较器定义[]字节键空间上的总排序：“小于”关系。在数据库的整个生命周期内，读取和写入必须使用相同的比较算法。

//默认值使用与字节相同的顺序。比较。



**DebugCheck**

//只要数据有新版本，就会调用DebugCheck（如果不是nil）

//已安装。通常，这在测试中设置为pebble.DebugCheckLevels

//或者仅使用工具来检查数据库中所有数据的不变量。



**DisableWAL**

//禁用预写日志（WAL）。禁用预写日志禁止

//崩溃恢复，但如果不进行崩溃恢复，则可以提高性能

//需要（例如，当数据库中仅存储临时状态时）。

//TODO（peter）：未经测试



**ErrorIfExists**

//如果数据库已经存在，ErrorIfExists会在打开时导致错误。

//可以使用错误检查错误。Is（err，ErrDBAlreadyExists）。

//默认值为false。



**ErrorIfNotExists**

//如果数据库尚未打开，ErrorIfNotExists会导致打开时出错

//存在。可以使用错误检查错误。Is（err，ErrDBDoesNotExist）。

//默认值为false，如果不存在。



**ErrorIfNotPristine**

//如果数据库已存在，ErrorIfNotPristine会在打开时导致错误

//并且已经对数据库执行了任何操作。错误可能是

//检查时出现错误。Is（err，ErrDBNotPristine）。

//请注意，包含随后全部被删除的密钥的数据库

//可以触发也可以不触发错误。目前，我们检查是否有任何直播

//要重播的SST或日志记录。



==EventListener==

//EventListener提供了监听重要DB事件的挂钩，例如

//刷新、压缩和表删除。



**Experimental**

//实验包含默认情况下处于禁用状态的实验选项。

//这些选项是临时的，最终要么被删除，要么被移动

//出实验组，或设为不可调默认值。这些

//选项可能随时更改，因此不要依赖它们。

{

**L0CompactionConcurrency**

//压缩并发时L0读取放大的阈值

//已启用（如果尚未超过CompactionDebtCurrency）。

//此值的每一个倍数都会启用另一个并发

//压缩到MaxConcurrentCompactions。

**CompactionDebtConcurrency**

//压缩线程并发控制压缩线程的阈值

//在该位置添加额外的压缩并发槽。对于每个

//压缩债务字节中该值的倍数

//添加了并发压缩。这是在

//L0CompactionConcurrent，因此压实计数越高

//选择由这两个选项确定的并发槽。



**ReadCompactionRate**

//ReadCompactionRate控制触发读取的频率

//通过调整manifest.FileMetadata中的“AllowedSeeks”进行压缩：

//AllowedSeeks=文件大小/读取压缩率

//来自LevelDB：

//```

//我们安排在之后自动压缩此文件

//一定数量的搜索。让我们假设：

//（1）一次搜索花费10ms

//（2）写入或读取1MB需要10ms（100MB/s）

//（3）1MB的压缩会产生25MB的IO：

//从该级别读取1MB

//从下一级别读取10-12MB（边界可能未对齐）

//10-12MB写入下一级别

//这意味着25寻求与压实相同的成本

//1MB的数据。也就是说，一次搜索的成本约为

//与40KB数据的压缩相同。我们有点

//保守，并允许大约每16KB进行一次搜索

//触发压缩之前的数据。



==ReadSamplingMultiplier==

//ReadSamplingMultiplier是中readSamplingPeriod的乘数

//迭代器.maybeSampleRead（）来控制读取采样的频率

//以触发读取触发的压缩。值为-1会阻止采样

//并禁用读取触发的压缩。默认值为1<<4。哪一个

//与常数1<<16相乘，得到1<<20（1MB）。



**TableCacheShards**

//TableCacheShards是每个表缓存的碎片数。

//减少该值可以减少每个DB的空闲goroutine的数量

//实例，这在具有大量DB实例的场景中非常有用

//和大量的CPU，但这样做可能会导致更高的争用

//在表缓存中，性能降低。

//默认值是逻辑CPU的数量，可以是

//受运行时间GOMAXPROCS限制。



**KeyValidationFunc**

//KeyValidationFunc是一个用于验证SSTable中用户密钥的函数。

//目前，此函数用于验证最小和最大

//正在压缩的SSTable中的键。在这种情况下，返回

//来自验证函数的错误将在运行时导致死机，

//考虑到很少有任何方法可以从格式错误的密钥中恢复

//存在于压缩文件中。默认情况下，不执行验证。

//未来可能会添加其他用例。

//注意：调用方应注意不要更改正在验证的密钥。



**ValidateOnIngest**

//ValidateOnIngest计划在sstables完成后对其进行验证

//被摄入。

//默认情况下，此值为false。



**LevelMultiplier**

//LevelMultiplier配置用于确定

//LSM的每个级别的期望大小。默认值为10。



**MultiLevelCompactionHueristic**

//MultiLevelCompactionHueristic确定是否添加附加

//水平至传统的两级压实。如果为零，则为多级

//压缩永远不会被触发。

多级压缩启发式多级启发式



**MaxWriterConcurrency**

//MaxWriterConcurrent用于指示

//允许压缩队列使用的压缩工作者。如果

//MaxWriterConcurrent>0，则Writer将使用并行性，以

//压缩块并将其写入磁盘。否则，作者将

//同步压缩块并将其写入磁盘。

最大写入电流int



**ForceWriterParallelism**

//ForceWriterParallelism用于强制sstable中的并行性

//变形测试的作者。即使使用MaxWriterConcurrent

//选项集，如果存在

//是否有足够的CPU可用，而此选项将绕过此选项。



**CPUWorkPermissionGranter**

//如果Pebble应获得

//有选择地安排额外CPU的能力。请参阅文档

//有关更多详细信息，请参阅CPUWorkPermissionGranter。



**EnableValueBlocks**

//EnableValueBlocks用于决定是否启用写入

//表格式Pebblev3 sstables。此设置仅受

//格式主要版本的特定子集：FormatSSTableValueBlocks，

//FormatFlushableIngest和FormatPrePebblv1MarkedCompacted。在下部

//格式化主要版本，值块永远不会启用。在更高

//格式主要版本，值块始终处于启用状态。



**ShortAttributeExtractor**

//如果EnableValueBlocks（）返回true，则使用ShortAttributeExtractor

//（其他已忽略）。如果不是nil，则可以从

//值并与密钥一起存储，当该值存储在其他位置时。

ShortAttributeExtractor短属性提取器



**RequiredInPlaceValueBound**

//RequiredInPlaceValueBound指定用户密钥的可选范围

//不是MVCC但有后缀的前缀。对于这些值

//必须与密钥一起存储，因为“旧版本”的概念是

//未定义。对于静态已知的排除值，它也很有用

//分离。在蟑螂数据库中，这将用于锁表密钥

//具有非空后缀的空间，但这些锁不表示

//实际的MVCC版本（后缀顺序是任意的）。我们也会

//需要添加对动态配置排除的支持（我们希望

//默认为允许Pebble决定是否分离值

//是否，因此这被构造为排除），例如，对于用户

//以动态排除某些表。

//

//排除行为的任何变化仅在未来书面

//sstables，并且不开始重写现有的sstables。

//

//即使忽略此设置中的更改，排除也被解释为

//Pebble的指导，不一定受到尊重。具体来说，用户

//具有多个Pebble版本*的密钥可能*存储了旧版本

//在值块中。



//DisableIngestAsFlushable通过禁用sstables的惰性摄取

//WAL写入和可记忆旋转。只有当格式

//主要版本至少为“FormatFlushableIngest”。

禁用IngestAsFlushable函数（）布尔



//SharedStorage是第二种可以共享的类似FS的存储介质

//在多个Pebble实例之间。它仅用于存储sstables，并且

//由objstorage.Provider管理。每个sstable只能写入

//一个Pebble的例子，但其他



**Filters**

//过滤器是从过滤器策略名称到过滤器策略的映射。它用于

//调试工具，可用于配置有

//不同的过滤策略。没有必要填充此筛选器

//在数据库的正常使用期间映射。

筛选器映射[string]FilterPolicy



**FlushDelayDeleteRange**

//FlushDelayDeleteRange配置数据库在

//强制刷新包含范围删除的memtable。磁盘空间

//在清除范围删除之前无法回收。无自动

//如果为零，则发生冲洗。

FlushDelayDeleteRange时间。持续时间



**FlushDelayRangeKey**

//FlushDelayRangeKey配置数据库在

//强制刷新包含范围键的memtable。范围关键点在

//memtable可以防止懒惰的组合迭代，因此需要刷新

//范围键迅速。如果为零，则不会发生自动冲洗。

FlushDelayRangeKey时间。持续时间



**FlushSplitBytes**

//FlushSplitBytes表示中每个子级别的目标字节数

//每个齐平拆分间隔（即两个齐平拆分键之间的范围）

//在L0表中。当设置为零时，只生成一个sstable

//通过每次冲洗。当设置为非零值时，刷新将在

//点以满足L0的TargetFileSize，任何与祖父母相关的重叠

//选项，以及L0齐平分割间隔的边界关键点（

//目标是在每个子级别中包含大约FlushSplitBytes字节

//在边界键对之间）。冲洗期间拆分表

//允许在以下情况下提高压缩灵活性和并发性

//表格被压缩到较低的级别。

FlushSplitBytes int64格式



//FormatMajorVersion设置磁盘上文件的格式。是的

//建议将格式主版本设置为显式

//版本，因为默认值可能会随时间变化。

//

//在打开时，如果使用稍后的

//格式化这个版本的Pebble已知的主要版本，

//Pebble将继续使用后期的格式主要版本。如果

//现有数据库的版本未知，调用方可以使用

//FormatMost兼容并且将能够打开数据库

//而不管其实际版本如何。

//

//如果使用格式化专业对现有数据库进行格式化

//版本早于指定版本，打开将自动

//将数据库逐步调整为指定格式的主版本。

格式主要版本格式主要版本



//FS为持久文件存储提供了接口。

//

//默认值使用基础操作系统的文件系统。

财政司司长财政司司长



//锁（如果设置）必须是通过的LockDirectory获取的数据库锁

//传递给Open的相同目录。如果提供，Open将跳过锁定

//目录。关闭数据库不会释放锁，而且

//调用方在关闭后释放锁的责任

//数据库。

//

//Open将强制传递的Lock锁定传递到的同一目录

//打开。检测到使用相同锁对Open的并发调用，并且

//禁止。

锁定*锁定



//触发L0压缩所需的L0文件数。

L0压缩文件阈值int



//触发L0压缩所需的L0读取放大量。

L0压缩阈值int



//L0读取放大的硬限制，计算为L0的数量

//子级别。达到此阈值时将停止写入。

L0停止写入阈值int



//LBase的最大字节数。基准面是指

//L0被压实成。基准水平是根据

//LSM中的现有数据。其他级别的最大字节数

//是基于基本级别的最大大小动态计算的。当

//超过一个级别的最大字节数，则请求压缩。

L基本最大字节数int64



//按级别选项。必须至少指定一个级别的选项。这个

//最后一个级别的选项用于所有后续级别。

级别[]级别选项



//将使用LoggerAndTracer，如果不是nil，则将使用Logger，并且

//追踪将是一个小问题。



//用于写入日志消息的记录器。

//

//默认记录器使用Go标准库日志包。

记录器记录器

//LoggerAndTracer用于写入日志消息和跟踪。

记录器和跟踪器记录器和跟踪器



//MaxManifestFileSize是允许MANIFEST文件的最大大小

//成为。当MANIFEST超过这个尺寸时，它会被翻转

//MANIFEST已创建。

最大清单文件大小int64



//MaxOpenFiles是对可以

//由数据库使用。

//

//默认值为1000。

最大打开文件数



//处于稳定状态的MemTable的大小。MemTable的实际大小起始于

//最小值（256KB，MemTableSize），并为每个后续MemTable加倍，最多可达

//MemTableSize（内存表大小）。这降低了MemTables对

//短期（测试）DB实例。请注意，可以有多个MemTable

//现有的 

# 二、代码示例
```
package main

import (
	"fmt"
	"log"

	"github.com/cockroachdb/pebble"
)

var db *pebble.DB

func main() {
	//打开数据库
	var err error
	db, err = pebble.Open("demo", &pebble.Options{DebugCheck: func(db *pebble.DB) error {

		//fmt.Println(db.Metrics())
		fmt.Println(1111)
		return nil
	}})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("写入key")
	key := []byte("hello")
	key1 := []byte("hello1")
	if err := db.Set(key, []byte("world"), pebble.Sync); err != nil {
		log.Fatal(err)
	}
	if err := db.Set(key1, []byte("world1"), pebble.Sync); err != nil {
		log.Fatal(err)
	}

	fmt.Println()
	fmt.Println("获取key")
	value, closer, err := db.Get(key)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%s %s\n", key, value)

	fmt.Println()
	fmt.Println("获取全部key")
	GetAll()

	fmt.Println()
	fmt.Println("将内存刷如磁盘")
	//将内存表刷新到稳定存储。
	//db.Flush()
	///AsyncFlush将memtable异步刷新到稳定存储。
	////如果没有返回错误，调用方可以在中从返回的通道接收
	////以便等待刷新完成。
	//db.AsyncFlush()

	//b := db.NewBatch()
	//b.

	///////////////////////////////////////////////////////////
	fmt.Println()
	fmt.Println("关闭")
	if err := closer.Close(); err != nil {
		fmt.Println(1)
		log.Fatal(err)
	}
	if err := db.Close(); err != nil {
		fmt.Println(2)
		log.Fatal(err)
	}
}

func GetAll() (int, error) {
	var cout int
	iter := db.NewIter(&pebble.IterOptions{})
	defer iter.Close()
	//iter.First()

	for iter.First(); iter.Valid(); iter.Next() {
		v, e := iter.ValueAndErr()
		fmt.Println(string(v), e)
		fmt.Println(string(iter.Key()))
		cout++
	}
	//获取全部key
	//world <nil>
	//hello
	//world1 <nil>
	//hello1

	return cout, nil
}

```
