# 缓存层

## 如何为dbfs实现mmap

jammdb是使用mmap调用映射文件到内存中，而这块内存是只读并且操作系统会同步文件的更改到这块内存中。在内核中我们是否有必要去调用mmap这个接口呢？我们直接将设备抽象为了一个文件，这个过程中已经引入了缓存，那我们可以直接利用这个缓存，而不用使用mmap的接口，这样一来，既可以省去多余的内存映射开销，也可以省去同步的麻烦。

```rust
pub struct CacheLayer {
    device: Arc<QemuBlockDevice>,
    lru: LruCache<usize, FrameTracker>,
}
```

这个`CacheLayer` 使用`LRU` 的方式缓存数据，并直接从页帧分配器中分配物理页来进行缓存。

## 如何为fat32实现缓存

为了尽可能提高fat32的读写效率，也需要为fat32实现缓存。为了公平性，实验保证了两者的缓冲层实现几乎一致。