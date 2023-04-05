# 数据库简介


![db](assert/db.svg)

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

这里我们讨论的数据库是key-value类型的数据库，一般来说，key-value类型的数据库会提供最基本的键值对插入、删除功能，一些数据库会基于table构建，一些数据库有其它实现方式，本文讨论的是rust实现的`jammdb` 数据库，这是go语言实现的`boltdb` rust重写。此数据库的内部基于桶`bucket`实现。如上面所示，该数据库的基本结构由一个个处于全局空间的`bucket`组成，`bucket`可以存储普通的key-value数据，这里key和value都是[u8]数组，同时也可以存储嵌套的`bucket`数据结构。一个`bucket`是由一棵B+树构成，b+树将大小为4k或者其它大小的页面组织起来存储数据，这里的页面与传统的存储块类似，只是因为数据库使用内存映射而将存储结构设置为页大小。

## 为什么使用数据库构建文件系统

数据库和文件系统的功能逐渐交叉，两者存在很多借鉴，多数数据库依赖文件系统，但也存在一些数据库对文件系统的依赖没有那么强，即使那些依赖文件系统的数据库，在一些限定情况下，我们也可以消除这些依赖。基于数据库来实现文件系统：

1. 一方面可以大大简化文件系统的实现流程，因为数据库已经提供了一些丰富的功能，在强大的数据库中甚至包含了自己的缓存模块，而不去使用操作系统提供的缓存
2. 另一方面，数据库本身就包含一些有用的特性，例如完整的事务支持，要在文件系统中实现完整的事务支持是一件麻烦的事
3. 基于数据库的实现，可以在一定程度上统一目录和文件的差异。
4. 利用数据库的数据结构，可以对文件存储/访问/元数据扩展等开拓更有趣的功能