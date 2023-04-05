# 问题与解决方案


上面提到，我们将存储的数据交给用户进行解释，因此系统调用中的参数是来自用户，但是这个方案会带来一些问题

- 安全问题，由用户定义的函数如何在内核安全运行？这里似乎没有方案可以检测用户函数的合法性和正确性。

## 解决方法1

这个问题尤为重要，因为非法的用户程序将会破坏内核环境。这里的一个想法是不允许用户自行实现处理函数，而是内核提供一些**基元函数**，用户只能使用这些基元函数或其组合对数据进行操作。因为从上面的讨论中可以看到，用户对bucket的存储只有叶子键值对和非叶子键值对的区别。而对叶子键值对的操作无非是delete/read/write/add ，对非叶子键值对的操作也是仅限于delete/step into/ add ，基于这个假设，操作系统提供这些基元函数，这些基元函数其实就是上文中提到的为用户默认提供的一些函数翻版。

```rust
pub struct RenameKeyOperate {
    pub old_key: String,
    pub new_key: String,
}
pub struct AddKeyOperate {
    pub map: BTreeMap<String, Vec<u8>>,
}
pub struct AddBucketOperate {
    pub key:String,
    pub other:Option<Box<OperateSet>>
}
pub struct StepIntoOperate {
    pub key:String,
    pub other:Option<Box<OperateSet>>
}
pub struct DeleteKeyOperate {
    pub keys: Vec<String>,
}
pub struct ReadOperate {
    pub keys: Vec<String>,
    pub buf_addr: usize,
    pub buf_size: usize,
}
pub enum Operate {
    RenameKey(RenameKeyOperate),
    AddKey(AddKeyOperate),
    AddBucket(AddBucketOperate),
    DeleteKey(DeleteKeyOperate),
    Read(ReadOperate),
    StepInto(StepIntoOperate),
}

pub struct OperateSet {
    pub operate: Vec<Operate>,
}
```

对于用户来说，处理数据的需求是复杂的，似乎没有办法在内核为其提供足够的基元函数来完成数据处理，所以我们只能提供一些粒度适中的函数，既能让用户可以对数据有更多的处理，又不需要运行用户自定义处理函数。**这种方案会大大降低用户对数据的掌控能力，在对数据进行更细的操作就不太行了。**

**example**

```rust
let addkey_operate = AddKeyOperate::new()
        .add_key("name", b"hello".to_vec())
        .add_key("data1", b"world".to_vec())
        .add_key("data2", b"world".to_vec());
let buf = [0u8; 20];
let read_operate = ReadOperate::new()
        .add_key("name")
        .add_key("data1")
        .add_key("data2")
        .set_buf(buf.as_ptr() as usize, 20);
let mut add_bucket = AddBucketOperate::new("dir1",None);
let add_operate = AddKeyOperate::new()
                .add_key("uid",b"111".to_vec())
                .add_key("gid",b"222".to_vec())
                .add_key("mode",b"333".to_vec());

let add_bucket1 = AddBucketOperate::new("dir2",None);
let operate_set = OperateSet::new()
                .add_operate(Operate::AddKey(add_operate))
                .add_operate(Operate::AddBucket(add_bucket1));
add_bucket.add_other(Box::new(operate_set));

let operate_set = OperateSet::new()
        .add_operate(Operate::AddKey(addkey_operate))
        .add_operate(Operate::AddBucket(add_bucket))
        .add_operate(Operate::Read(read_operate));

execute_operate("test", operate_set);
```

如果将上述这段代码等效为用户可能会实现的函数，其形式大概如下：

```rust
fn equal(buf:&mut [u8]){
    let db = DB::open::<FileOpenOptions, _>(Arc::new(FakeMap), "my-database.db").unwrap();
    let tx = db.tx(true).unwrap();
    let bucket = tx.get_bucket("test").unwrap();
    bucket.put("name", b"hello".to_vec()).unwrap();
    bucket.put("data1", b"world".to_vec()).unwrap();
    bucket.put("data2", b"world".to_vec()).unwrap();
    let mut start = 0;
    let v1 = bucket.get_kv("name").unwrap();
    let min = min(v1.len(), buf.len() - start);
    buf[start..start + min].copy_from_slice(&v1[..min]);
    start += min;
    let v2 = bucket.get_kv("data1").unwrap();
    let min = min(v2.len(), buf.len() - start);
    buf[start..start + min].copy_from_slice(&v2[..min]);
    start += min;
    let v3 = bucket.get_kv("data2").unwrap();
    let min = min(v3.len(), buf.len() - start);
    buf[start..start + min].copy_from_slice(&v3[..min]);
    start += min;

    let dir = bucket.create_bucket("dir1").unwrap();
    dir.put("uid", b"111".to_vec()).unwrap();
    dir.put("gid", b"222".to_vec()).unwrap();
    dir.put("mode", b"333".to_vec()).unwrap();
    dir.create_bucket("dir2").unwrap();
}
```



## 解决方法2

利用Linux中的一些特性，e-BPF以及其它一些项目似乎有办法在内核执行用户定义的函数。