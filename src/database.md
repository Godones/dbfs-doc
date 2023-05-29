# 数据库简介

jammdb是一个嵌入式、单文件的key-value数据库，其提供ACID特性，支持多个并发读取和单个写入。所有的数据被组织成一棵B+树，随机和顺序读取速度很快。其对文件的操作基于内存映射。选择这个数据库作为dbfs实现的原因是其结构比较简单，因为复杂的数据库不仅难以移植到裸机平台，要将其放到内核态需要大量的工作，但简单带来的一个坏处就是数据库的功能不够强大，并且数据库本身没有对磁盘设备作出相应的优化，这可能会导致最终的DBFS性能不能达到最好状态。

数据库的数据结构如下所示:

![db](assert/db-1683515704725-3.svg)





```
key1-bucket
		|key1-value
		|key2-value
		|key-bucket |key1-value
					|key2-value
					|key3-value
key-bucket
		|key-value
		|key-value
```

```rust
let db = DB::open::<FileOpenOptions, _>(Arc::new(FakeMap), "my-database.db").unwrap();
let tx = db.tx(true).unwrap();
let bucket = tx.create_bucket("file").unwrap();
bucket.put("name", "hello").unwrap();
bucket.put("data1", "world").unwrap();
bucket.put("data2", "world").unwrap();
let n_bucket = bucket.create_bucket("dir1").unwrap();
```





数据库的内部基于桶`bucket`实现。如图所示，该数据库的基本结构由一个个处于全局空间的`bucket`组成，`bucket`可以存储普通的key-value数据，这里key和value都是[u8]数组，同时也可以存储嵌套的`bucket`数据结构。一个`bucket`是由一棵B+树构成，b+树将大小为4k或者其它大小的页面组织起来存储数据，这里的页面与传统文件系统的磁盘块类似，只是因为数据库使用内存映射而将存储结构描述为页。

数据库使用mmap系统调用来实现其缓存结构和事务特性，所有的只读操作发生在内存映射区域，当发生写操作时，数据库会及时刷新磁盘并同步更新内存映射。

