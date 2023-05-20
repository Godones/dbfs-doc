# 文件系统性能测试

[TOC]



## 性能测试内容

1. 读写速度测试：测试文件系统的读写速度，包括顺序读写和随机读写
2. 并发测试：测试文件系统在并发访问下的性能表现
3. 压力测试：测试文件系统在高负载下的性能表现

### 测试指标

文件系统性能测试可以使用多个指标来衡量，下面是一些常见的文件系统测试指标：

1. 延迟（Latency）：文件系统读取或写入数据所需的时间。延迟是衡量文件系统性能的重要指标，因为用户通常关心操作完成所需的时间。
2. 吞吐量（Throughput）：文件系统在一定时间内读取或写入数据的速率。吞吐量通常用MBps或GBps表示。
3. IOPS（每秒输入/输出操作数）：文件系统在一秒钟内处理的读写操作数。IOPS通常是衡量闪存存储设备性能的重要指标。
4. 并发性（Concurrency）：文件系统同时处理多个读写操作的能力。并发性是在高负载条件下测试文件系统性能的关键指标。
5. 可靠性（Reliability）：文件系统出现故障或意外情况时的恢复能力。例如，当系统意外断电时，文件系统应该能够在重新启动后恢复数据。
6. 一致性（Consistency）：文件系统保持数据一致性的能力。例如，当多个用户同时读取和写入文件时，文件系统应该能够正确处理数据并保持一致性。
7. 安全性（Security）：文件系统保护数据不被非授权用户访问的能力。例如，文件系统应该支持文件权限和访问控制列表（ACL）。

测试指标的选择取决于文件系统的具体用途和性能需求。在选择测试指标时，应该考虑到测试环境、应用场景和用户需求，以便正确地评估文件系统的性能。

## Alien OS的性能测试

本文不仅将DBFS移植到Alien OS中，为了可以更直接查看内核中DBFS的性能，还移植了一个fat32文件系统，因为Alien OS中有一个类似linux 的VFS，因此fat32的移植会比较简单。本文编写了几个测试程序以测试两个文件系统是否正确工作，实验证明确实如此。在实现正确的基础上，本文编写了几个读写测试的程序。在顺序写测试中本文写入10MB大小的数据，在第一次写入时，DBFS会处理缺页异常将磁盘块的内容加载到内存中，fat32同样也会加载数据到内存中。在顺序读中，先写入10MB大小的数据，再将数据顺序读出。在随机读写测试中，首先写入16MB大小的数据，随后进行1000次的随机读取操作和1000次的随机写操作。



| (MB/s)/FS   |     DBFS |   FAT32 |
| :---------- | -------: | ------: |
| 顺序写(1MB) |   32MB/s |  15MB/s |
| 顺序写(Re)  |  384MB/s | 104MB/s |
| 顺序读(1MB) | 1000MB/s | 110MB/s |
| 顺序读(4KB) |   27MB/s |  50MB/s |
| 随机写(1MB) |  386MB/s |  56MB/s |
| 随机读(1MB) | 1066MB/s |  58MB/s |







## fuse测试

在完成fuse接口的适配工作后，可以在linux上对dbfs的正确性和性能进行测试, 对于功能性验证，我们主要使用pjdfstest这个比较流行的测试集,对于性能测试，我们主要使用FIO、mdtest、filebench三个工具或者测试集。

### Juice fs + Fuse2fs [e2fs](https://e2fsprogs.sourceforge.net/)

fuse2fs - FUSE file system client for ext2/ext3/ext4 file systems。

为了与其他文件系统作比较，这里选取了ext系列文件系统，这个系列的文件系统有fuse的客户端，因此可以与dbfs进行对比,同时我们还选取了另一个使用了数据库技术的`juice fs` 参与对比。这里我们简要介绍这几个文件系统的主要特征。

#### ext2/ext3/ext4

ext2是Linux中最早的文件系统之一，也是最简单的文件系统。它具有可靠性和稳定性，但不支持日志记录和热插拔。这意味着如果系统异常关机，文件系统可能会损坏。ext2文件系统的优点是速度快，适合用于嵌入式系统和旧硬件设备。ext3是在ext2基础上发展而来的文件系统，增加了日志记录功能，可以避免文件系统损坏。当系统异常关机时，ext3会使用日志记录来恢复文件系统，保证数据的一致性。ext3的日志功能对磁盘的驱动器读写头进行了优化,所以，文件系统的读写性能较之Ext2文件系统并来说，性能并没有降低。ext4是在ext3基础上发展改进的文件系统，增加了一些新功能和性能改进。它支持更大的文件和分区，支持快速恢复和热插拔。此外，ext4还采用了一些新的技术来提高文件系统性能，如延迟分配、多块分配和快速预读取。实验中，我们选择使用ext2与ext4两个主要版本进行测试。

### 测试条件

机器信息:

- 8  Intel(R) Core(TM) i5-8265U CPU @ 1.60GHz
- 8GB memory
- Ubuntu 22.04.2 LTS

程序运行条件:

1. dbfs使用release模式运行, mount 参数为 

```
-o auto_unmount allow_other default_permissions rw async
```

2. ext系列使用fuse2fs工具进行挂载,具体步骤为:

   1. 使用dd生成4GB大小的文件
   2. 使用mkfs工具对文件进行格式化生成对应的文件系统
   3. 使用fuse2fs工具将文件系统挂载到任意一个空目录上

   这个系列的挂载参数与dbfs的相同

### pjdfstest

pjdfstest是一套简化版的文件系统POSIX兼容性测试套件，它可以工作在FreeBSD, Solaris, Linux上用于测试UFS, ZFS, ext3, XFS and the NTFS-3G等文件系统。目前pjdfstest包含八千多个测试用例，基本涵盖了文件系统的接口。

pjdfstest的项目地址与编译流程位于[pjdfstest](https://github.com/pjd/pjdfstest)

在挂载dbfs-fuse后，可以在挂载目录下运行测试程序得到dbfs的测试结果。

#### 测试结果

| test-set        | interface                 | pass/all  | error/all | error                    |
| --------------- | ------------------------- | --------- | --------- | ------------------------ |
| chflags         | chflags(FreeBSD)          | 14/14     | :x:       | linux下不工作            |
| chmod           | chmod/stat/symlink/chown  | 321/327   | 26/327    | chmod实现有误            |
| chown           | chown/chmod/stat          | 1280/1540 | 260/1540  | chmod实现有误            |
| ftruncate       | truncate/stat             | 88/89     | 1/89      | Access判断出错           |
| granular        | 未知                      | 7/7       | :x:       | linux下不工作            |
| link            | link/unlink/mknod         | 359/359   | :x:       | :heavy_check_mark:       |
| mkdir           | mkdir                     | 118/118   | :x:       | :heavy_check_mark:       |
| mkfifo          | mknod/link/unlink         | 120/120   | :x:       | :heavy_check_mark:       |
| mknod           | mknod                     | 186/186   | :x:       | :heavy_check_mark:       |
| open            | open/chmod/unlink/symlink | 328/328   | :x:       | :heavy_check_mark:       |
| posix_fallocate | fallocate                 | 21/22     | 1/22      | Access判断错误           |
| rename          | rename/mkdir/             | 4458/4857 | 399/4857  | 错误返回值处理与逻辑错误 |
| rmdir           | rmdir                     | 139/145   | 6/145     | 错误处理，权限检查       |
| symlink         | symlink                   | 95/95     | :x:       | :heavy_check_mark:       |
| truncate        | truncate                  | 84/84     | :x:       | :heavy_check_mark:       |
| unlink          | unlink/link               | 403/440   | 37/440    | mknod中的socket,错误处理 |
| utimensat       | utimens                   | 121/122   | 1/122     | 权限检查                 |
|                 |                           | 7943/8674 | 731/8674  | 92%                      |



#### 结果说明

在pjdfstest中，错误主要集中在rename和chown的处理当中，rename是其中最复杂的部分，本项目并没有处理所有细节，chown是权限检查的主要部分，实现细节也有待斟酌。还有出现错误的位置主要是错误处理当中，一些错误值没有按照Posix标准进行返回.



#### TODO

- [ ] 更准确的错误处理
- [ ] chown的细节处理, 似乎需要处理用户和当前进程的的关系
- [ ] rename的完整实现



### FIO测试(性能测试)

fio是一款基于命令行的磁盘I/O性能测试工具，可以测试各种不同类型的磁盘I/O工作负载，例如随机读写、顺序读写、混合读写等。fio可以在Linux、FreeBSD、macOS等操作系统上运行，支持多线程、异步I/O和多种I/O引擎，具有高度的可定制性和可扩展性。

fio的主要优势在于其灵活性和可配置性。用户可以通过fio的配置文件或者命令行选项，自定义测试工作负载、块大小、I/O引擎、线程数、测试时间等参数。此外，fio还可以输出详细的测试结果，包括吞吐量、IOPS、延迟等指标，方便用户进行性能分析和比较。

fio的主要参数有：

- `--name`：用户指定的测试名称，会影响测试文件名
- `--directory`：测试目录
- `--ioengine`：测试时下发 IO 的方式；通常用 libaio 即可
- `--rw`：常用的有 read，write，randread，randwrite，分别代表顺序读写和随机读写
- `--bs`：每次 IO 的大小
- `--size`：每个线程的 IO 总大小；通常就等于测试文件的大小
- `--numjobs`：测试并发线程数；默认每个线程单独跑一个测试文件
- `--direct`：在打开文件时添加 `O_DIRECT` 标记位，不使用系统缓冲，可以使测试结果更稳定准确

本文使用fio工具测试几个文件系统的顺序读，顺序写，随机读，随机写性能。在实验过程中，发现在进行读测试时由于需要先往目录下写入文件再读取，因此第一次读测试的结果与后面的测试会有较大波动，因此本文对第一次读的测试不做记录，而是在第一次读往目录写入文件后再测试，为了提高正确性，测量了三次取其平均值。



#### 顺序写

```shell
sudo fio --name=$(fs) --directory=$(fs)  --rw=write --bs=1m --size=$(size) --numjobs=$(jobs) --direct=1 --group_reporting
```

#### 顺序读

```bash
sudo fio --name=$(fs) --directory=$(fs)  --rw=read --bs=1m --size=$(size) --numjobs=$(jobs) --direct=1 --group_reporting
```

#### 随机写

```shell
sudo fio --name=$(fs) --directory=$(fs)  --rw=randwrite --bs=1m --size=$(size) --numjobs=$(jobs) --direct=1 --group_reporting
```

#### 随机读

```shell
sudo fio --name=$(fs) --directory=$(fs)  --rw=randread --bs=1m --size=$(size) --numjobs=$(jobs) --direct=1 --group_reporting
```



### mdtest(元数据处理能力)

mdtest是一款针对服务器元数据处理能力的基准测试工具，可以用来模拟对文件或目录的open/stat/close操作，然后返回报告。它支持MPI，可以用来协调大量客户端对服务器发起请求。

- -b: 目录树的分支参数
- -d: 指出测试运行的目录
- -I: 每个树节点包含的项目
- -z: 目录树的深度

各个文件系统的测试命令如下所示：

```
sudo mdtest -d dbfuse -b 6 -I 8 -z 3
```

![image-20230502222234499](assert/image-20230502222234499.png)

#### 结果说明

#### TODO

- [ ] remove/rename太慢



## 性能改进

### 1. flush+sync_all

在数据库中, 每当一个写事务发生，根据`jammdb` 的机制，会调用`flush` 和 `sync_all` 来将文件的元数据以及缓存中的数据写回磁盘，并更新`mmap`中的只读缓存，由于缺乏写缓存且fuse位于用户态，这导致了非常大的性能开销，因此我们需要在这两个函数中作一定的优化。

具体而言, 我们在实现`jammdb`的 `File` 的接口时，并不直接将`flush` 和 `sync_all` 映射到`File`的这两个接口。原来这两个接口的实现是这样的: 

```rust
fn flush(&mut self) -> core2::io::Result<()> {
        self.file
            .flush()
            .map_err(|_x| core2::io::Error::new(core2::io::ErrorKind::Other, "flush error"))
}
fn sync_all(&self) -> IOResult<()> {
        self.file
            .sync_all()
            .map_err(|_x| core2::io::Error::new(core2::io::ErrorKind::Other, "sync_all error"))
}
```

对这两个接口的初步改进方案是我们直接将对文件的调用忽略，因为操作系统会为我们更新文件的数据。

WARN： 破坏事务一致性



### 2. 固定文件大小

在原来的实现中，本文使用本机的文件模拟dbfs的镜像文件，但这个实现是按照数据库的申请而逐渐增大文件大小的，对于ext文件系统，一开始就分配了一个固定大小的极限文件，因此在读写过程中并不会向系统再次申请文件，而对于juicefs来说，其本身不需要读取镜像文件，而是直接使用linux的文件接口。

在这个改进中，我们在创建文件时，直接初始化这个文件的大小为一个常数值，因为所有的性能测试文件大小都在一个可预测的范围，因此整个过程中数据库只会调用这个分配接口而不需要真正的去分配。同时，对于mmap系统调用，在数据库的原实现中，每一次增大文件都会重新进行映射，然后我们的改进是固定了文件的大小，因此对这个调用也同时进行了修改，我们只在第一次进行映射时记录映射的数据，后面调用这个接口时，返回的仍然是第一个数据。

```rust
static MMAP:Once<Arc<IndexByPageIDImpl>> = Once::new();

impl MemoryMap for FakeMMap {
    fn do_map(&self, file: &mut File) -> IOResult<Arc<dyn IndexByPageID>> {
        if !MMAP.is_completed(){
            let file = &file.file;
            let fake_file = file.downcast_ref::<FakeFile>().unwrap();
            let res = mmap(&fake_file.file, false);
            if res.is_err() {
                return Err(core2::io::Error::new(
                    core2::io::ErrorKind::Other,
                    "not support",
                ));
            }
            let map = res.unwrap();
            let map = Arc::new(IndexByPageIDImpl{map});
            MMAP.call_once(||map);
        }
        Ok(MMAP.get().unwrap().clone())
    }
}
```



### 3.调整块大小

对于dbfs来说，本质上是没有块大小的说法的，这里其实指的就是我们在使用key-value键值对保存文件数据时做的优化手段，这在设计文档中也有说明。

在原实现中，我们规定了这个块大小为512字节，但是在实验中发现，其他几个文件系统使用的块大小都为4kb大小，在进行fio测试时，我们设定了每次读取的大小为1MB，对于其他几个文件系统来说，每次只需要读256个块，但是对于dbfs来说就需要读取2048个键值对，而且这些键值对是需要依次查找的，其他文件系统则会尝试分配在一个连续的空间，因此增大dbfs的块大小，理论上是可以增大吞吐量的。这里改进我们增加了几个可选的块大小。

```rust
#[cfg(feature = "sli512")]
pub const SLICE_SIZE:usize = 512;

#[cfg(feature = "sli1k")]
pub const SLICE_SIZE:usize = 1024;

#[cfg(feature = "sli4k")]
pub const SLICE_SIZE:usize = 4096;

#[cfg(feature = "sli8k")]
pub const SLICE_SIZE:usize = 8192;

```



### 4.修改数据库存储数据的方式（TODO)

在这个方案中，需要分析数据库在存储数据时使用的策略，再根据一般存储文件的需求做相应改进。





## 改进测试

上文中我们已经提出了三个当前可行的解决方案，因此，在本节中我们会对这些改进方法进行测试，检查这些改进是否会真正地提升性能。具体而言，我们对每一个改进依次运行mdtest测试，fio的读写测试。

### mdtest测试

![mdtest-opt](assert/mdtest-opt-1683162685692-11.svg)

解析： 从图中可以看到，没有优化的原始实现与优化后的实现差距很大，有的操作达到了数十倍的性能提升。





## 改进后的最终测试

在对DBFS进行改进后，将其再次进行数据的测试。为了屏蔽操作系统缓存带来的影响，在进行读写吞吐量进行测试时，将读写的文件设置为内存容量的2倍。并且在测试读性能时，不能紧跟着写测试，因为写测试会填充操作系

### mdtest

![mdtest](assert/mdtest-1684205988754-1.svg)



在mdtest中，主要衡量的是文件系统对元数据操作的性能



### FIO

#### 顺序写

```rust
sudo fio --name=dbfs-test1 --directory=./ext4  --rw=write --bs=1mb --size=15g --numjobs=1 --direct=1 --group_reporting
```

```
EXT4

write: IOPS=116, BW=117MiB/s (122MB/s)(15.0GiB/131780msec); 0 zone resets
clat (usec): min=6599, max=26157, avg=8553.45, stdev=856.22
lat (usec): min=6611, max=26173, avg=8577.07, stdev=858.87

write: IOPS=114, BW=115MiB/s (121MB/s)(15.0GiB/133601msec); 0 zone resets
clat (usec): min=6572, max=28297, avg=8667.75, stdev=840.62
lat (usec): min=6588, max=28323, avg=8694.96, stdev=841.80

```



```
EXT3

write: IOPS=390, BW=391MiB/s (410MB/s)(15.0GiB/39302msec); 0 zone resets
clat (usec): min=884, max=124814, avg=2535.59, stdev=6198.27
lat (usec): min=900, max=124829, avg=2556.79, stdev=6197.98

write: IOPS=379, BW=379MiB/s (397MB/s)(15.0GiB/40521msec); 0 zone resets
    clat (usec): min=879, max=35251, avg=2613.24, stdev=5033.51
     lat (usec): min=895, max=35266, avg=2635.87, stdev=5032.91

```



```
DBFS

write: IOPS=159, BW=160MiB/s (168MB/s)(15.0GiB/96004msec); 0 zone resets
    clat (usec): min=396, max=142123, avg=6214.68, stdev=14140.60
     lat (usec): min=413, max=142184, avg=6247.11, stdev=14141.96
write: IOPS=144, BW=144MiB/s (151MB/s)(15.0GiB/106458msec); 0 zone resets
    clat (usec): min=474, max=283761, avg=6891.01, stdev=14275.28
     lat (usec): min=493, max=283794, avg=6927.63, stdev=14276.67


```



|        | EXT4    | EXT3    | DBFS    |
| ------ | ------- | ------- | ------- |
| 第一次 | 117MB/s | 391MB/s | 160MB/s |
| 第二次 | 115MB/s | 379MB/s | 144MB/s |
| 平均值 | 116MB/s | 385MB/s | 152MB/s |

```
EXT4

read: IOPS=268, BW=269MiB/s (282MB/s)(15.0GiB/57196msec)
    clat (usec): min=2969, max=31592, avg=3721.25, stdev=699.31
     lat (usec): min=2970, max=31592, avg=3721.54, stdev=699.32
read: IOPS=268, BW=269MiB/s (282MB/s)(15.0GiB/57161msec)
    clat (usec): min=2886, max=32557, avg=3719.47, stdev=689.94
     lat (usec): min=2886, max=32557, avg=3719.68, stdev=689.94
     
     
EXT3 
read: IOPS=301, BW=301MiB/s (316MB/s)(15.0GiB/50983msec)
    clat (usec): min=2604, max=30999, avg=3317.00, stdev=920.70
     lat (usec): min=2604, max=30999, avg=3317.29, stdev=920.72

read: IOPS=302, BW=302MiB/s (317MB/s)(15.0GiB/50785msec)
    clat (usec): min=2592, max=34444, avg=3304.34, stdev=919.04
     lat (usec): min=2593, max=34444, avg=3304.60, stdev=919.03

DBFS

read: IOPS=25, BW=25.7MiB/s (26.9MB/s)(15.0GiB/597984msec)
    clat (usec): min=183, max=493284, avg=38927.23, stdev=16815.88
     lat (usec): min=183, max=493284, avg=38927.69, stdev=16815.98
read: IOPS=22, BW=22.7MiB/s (23.8MB/s)(3514MiB/154772msec)
    clat (msec): min=37, max=511, avg=44.04, stdev=16.56
     lat (msec): min=37, max=511, avg=44.04, stdev=16.56
```



#### 顺序读

|        | EXT4      | EXT3      | DBFS     |
| ------ | --------- | --------- | -------- |
| 第一次 | 269MB/s   | 301MB/s   | 26MB/s   |
| 第二次 | 268MB/s   | 302MB/s   | 23MB/s   |
| 平均值 | 268.5MB/s | 301.5MB/s | 24.5MB/s |



```
EXT4

write: IOPS=69, BW=69.9MiB/s (73.3MB/s)(15.0GiB/219802msec); 0 zone resets
    clat (usec): min=6400, max=31651, avg=14274.43, stdev=1958.85
     lat (usec): min=6427, max=31687, avg=14306.05, stdev=1960.58

write: IOPS=70, BW=70.7MiB/s (74.1MB/s)(15.0GiB/217331msec); 0 zone resets
    clat (msec): min=6, max=110, avg=14.12, stdev= 2.17
     lat (msec): min=6, max=110, avg=14.15, stdev= 2.17
     
     
EXT3

write: IOPS=371, BW=372MiB/s (390MB/s)(15.0GiB/41310msec); 0 zone resets
    clat (usec): min=893, max=34289, avg=2663.62, stdev=4958.01
     lat (usec): min=904, max=34300, avg=2686.59, stdev=4957.54
write: IOPS=379, BW=380MiB/s (398MB/s)(15.0GiB/40446msec); 0 zone resets
    clat (usec): min=887, max=34733, avg=2606.85, stdev=4995.64
     lat (usec): min=902, max=34752, avg=2630.44, stdev=4995.14


DBFS
write: IOPS=139, BW=139MiB/s (146MB/s)(15.0GiB/110384msec); 0 zone resets
    clat (usec): min=436, max=199773, avg=7146.47, stdev=15380.03
     lat (usec): min=454, max=199813, avg=7182.56, stdev=15381.95
 write: IOPS=138, BW=138MiB/s (145MB/s)(15.0GiB/111210msec); 0 zone resets
    clat (usec): min=434, max=197632, avg=7199.87, stdev=14425.79
     lat (usec): min=451, max=197674, avg=7236.69, stdev=14427.02

```

#### 随机写

|        | EXT4   | EXT3    | DBFS      |
| ------ | ------ | ------- | --------- |
| 第一次 | 70MB/s | 372MB/s | 139MB/s   |
| 第二次 | 70MB/s | 380MB/s | 138MB/s   |
| 平均值 | 70MB/s | 376MB/s | 138.5MB/s |



#### 随机读

```
EXT4
read: IOPS=155, BW=155MiB/s (163MB/s)(15.0GiB/99083msec)
    clat (usec): min=2941, max=49300, avg=6447.97, stdev=2150.72
     lat (usec): min=2941, max=49300, avg=6448.27, stdev=2150.75
read: IOPS=155, BW=155MiB/s (163MB/s)(15.0GiB/98853msec)
    clat (usec): min=3342, max=34196, avg=6432.91, stdev=998.69
     lat (usec): min=3342, max=34196, avg=6433.21, stdev=998.72



EXT3
read: IOPS=198, BW=198MiB/s (208MB/s)(15.0GiB/77535msec)
    clat (usec): min=2658, max=51951, avg=5044.67, stdev=1892.24
     lat (usec): min=2658, max=51952, avg=5045.00, stdev=1892.26
read: IOPS=191, BW=192MiB/s (201MB/s)(15.0GiB/80110msec)
    clat (usec): min=2594, max=46084, avg=5212.13, stdev=2284.28
     lat (usec): min=2594, max=46084, avg=5212.50, stdev=2284.30


DBFS
read: IOPS=23, BW=23.9MiB/s (25.1MB/s)(4409MiB/184194msec)
    clat (msec): min=35, max=230, avg=41.77, stdev= 4.59
     lat (msec): min=35, max=230, avg=41.77, stdev= 4.59
read: IOPS=27, BW=27.3MiB/s (28.6MB/s)(2354MiB/86259msec)
clat (usec): min=744, max=90048, avg=36638.69, stdev=15654.76
 lat (usec): min=744, max=90049, avg=36639.17, stdev=15654.75


```



|        | EXT4    | EXT3    | DBFS     |
| ------ | ------- | ------- | -------- |
| 第一次 | 155MB/s | 198MB/s | 24MB/s   |
| 第二次 | 155MB/s | 192MB/s | 27MB/s   |
| 平均值 | 155MB/s | 195MB/s | 25.5MB/s |







![fio-test-1job](assert/fio-test-1job.svg)





在顺序写中，ext3的性能最好，几乎达到了原生读写的速度。而ext4的性能只是ext3的25%，DBFS的性能是EXT3的40%左右，是ext4的1.3倍，说明DBFS的写性能在一定程度上是可以追赶上ext系列文件系统的。在顺序读中，ext3 的读取速度仍然是最快的，而ext4的读取速度是其89%左右，对于DBFS来说，其读性能相较于两个系统都非常差，甚至只达到了ext4的10%左右。在随机写测试中，ext3与dbfs都保持了与顺序写差不多的性能，性能损失在10%以内。对于ext4,其性能损失达到了35%左右。dbfs的随机写性能是EXT4的2倍，是ext3的37%。在随机读测试中，EXT3和EXT4都有较大程度的性能下降，但两者的差距并不是很大，EXT4是ext3的80%左右。对于DBFS，其性能与顺序读有相同的性能数据。由于ext3和ext4的性能下降，反而让DBFS的性能超过了两者的10%，但整体性能依然非常落后。







### 四个线程下的读写测试

3g/file 4thread

```
EXT4
write: IOPS=134, BW=135MiB/s (141MB/s)(12.0GiB/91306msec); 0 zone resets
    clat (usec): min=8423, max=77694, avg=29646.97, stdev=3125.62
     lat (usec): min=8462, max=77731, avg=29704.45, stdev=3125.18
  
read: IOPS=350, BW=351MiB/s (368MB/s)(12.0GiB/35048msec)
    clat (usec): min=4673, max=72146, avg=11401.04, stdev=5831.51
     lat (usec): min=4673, max=72147, avg=11401.35, stdev=5831.55
     
write: IOPS=78, BW=78.3MiB/s (82.1MB/s)(12.0GiB/156994msec); 0 zone resets
    clat (usec): min=11804, max=90177, avg=51002.34, stdev=9122.14
     lat (usec): min=11854, max=90249, avg=51062.56, stdev=9121.17
read: IOPS=229, BW=230MiB/s (241MB/s)(12.0GiB/53511msec)
    clat (msec): min=3, max=158, avg=17.41, stdev=10.02
     lat (msec): min=3, max=158, avg=17.41, stdev=10.02


EXT3
write: IOPS=396, BW=397MiB/s (416MB/s)(12.0GiB/30954msec); 0 zone resets
    clat (usec): min=1435, max=52828, avg=10018.82, stdev=8974.53
     lat (usec): min=1451, max=52871, avg=10065.48, stdev=8973.86
read: IOPS=370, BW=370MiB/s (388MB/s)(12.0GiB/33172msec)
    clat (usec): min=1126, max=39107, avg=10780.17, stdev=1648.60
     lat (usec): min=1126, max=39108, avg=10780.48, stdev=1648.61
write: IOPS=392, BW=393MiB/s (412MB/s)(12.0GiB/31273msec); 0 zone resets
    clat (usec): min=1297, max=235502, avg=10097.67, stdev=12704.50
     lat (usec): min=1324, max=235538, avg=10142.58, stdev=12703.03
read: IOPS=280, BW=280MiB/s (294MB/s)(12.0GiB/43815msec)
    clat (usec): min=5538, max=45706, avg=14246.54, stdev=2855.10
     lat (usec): min=5538, max=45707, avg=14246.87, stdev=2855.10


DBFS
write: IOPS=169, BW=170MiB/s (178MB/s)(12.0GiB/72293msec); 0 zone resets
    clat (usec): min=1476, max=340835, avg=23480.21, stdev=44725.19
     lat (usec): min=1515, max=340875, avg=23528.20, stdev=44725.66
read: IOPS=25, BW=25.1MiB/s (26.3MB/s)(12.0GiB/490429msec)
    clat (usec): min=1712, max=691379, avg=159618.30, stdev=40715.53
     lat (usec): min=1713, max=691379, avg=159618.84, stdev=40715.58
write: IOPS=138, BW=139MiB/s (146MB/s)(12.0GiB/88527msec); 0 zone resets
    clat (msec): min=2, max=540, avg=28.76, stdev=51.55
     lat (msec): min=2, max=540, avg=28.81, stdev=51.55
read: IOPS=23, BW=23.5MiB/s (24.6MB/s)(3743MiB/159492msec)
    clat (msec): min=41, max=598, avg=170.36, stdev=14.08
     lat (msec): min=41, max=598, avg=170.36, stdev=14.08

```

|      | 顺序写  | 顺序读    | 随机写    | 随机读   |
| ---- | ------- | --------- | --------- | -------- |
| EXT4 | 116MB/s | 268.5MB/s | 70MB/s    | 155MB/s  |
| EXT3 | 385MB/s | 301.5MB/s | 376MB/s   | 195MB/s  |
| DBFS | 152     | 24.5MB/s  | 138.5MB/s | 25.5MB/s |



|      | 顺序写  | 顺序读  | 随机写  | 随机读   |
| ---- | ------- | ------- | ------- | -------- |
| EXT4 | 135MB/s | 351MB/s | 78MB/s  | 230Mb/s  |
| EXT3 | 397MB/s | 370MB/s | 393MB/s | 280MB/s  |
| DBFS | 170MB/s | 25MB/s  | 139MB/s | 23.6MB/s |





![fio-test-1job-4job](assert/fio-test-1job-4job.svg)

![fio-test-4job](assert/fio-test-4job.svg)



图三显示了并发读写的结果。在并发读写测试中，从图三右图可以看到，不管是对于顺序读写还是随机读写，EXT3和EXT4都有一定的性能提升，在写测试中两者的性能提升在10%以内，在读测试中两者有显著的性能提升，甚至达到了单线程下的1.4倍。对于DBFS来说，四种操作的性能都没有明显提升，因此其与ext3的性能差距被放大，与ext4的性能优势被缩小，说明DBFS在办法控制方面的能力比较弱。



### 读写测试分析

在读写测试中可以看到，只有在写操作下DBFS才可以赶得上ext4的性能，并且与ext3的性能差距较远。而在读操作下DBFS的性能会急剧下降，以至于可能只有两者的零头那么多。造成这个性能差距的原因涉及到了诸多方面，其一，DBFS的fuse实现本身就与ext的实现存在实现细节的差异，DBFS对fuse没有做更多的优化，而ext3和ext4的fuse实现是有针对性的优化措施的，比如在进行数据读写时减少数据的拷贝，直接在内核和用户态传递数据，这在读操作时性能提升尤为明显，在DBFS中，一次读操作需要两次拷贝数据，一次从数据库中拷贝到申请的内存中，一次从申请的内存中传递到内核中，对于读15GB数据的测试来说，相当于DBFS需要读取30GB大小的数据。其二，在DBFS中，使用了键值对来存储文件数据，这导致这些数据可能会分散在不同的页面中，而且这些数据可能跨过多个页面，而在ext文件系统中，文件在存储时倾向于将这些数据存放在连续的块中，在进行读取操作时，ext文件系统可以直接快速查找到文件的数据存储区域，而在DBFS中，每一次的查找都会数据库可能都会便利存储的所有键值对，即使使用树结构可以加速查找过程，但频繁的查找相比只需要几次查找仍然带来了巨大的开销。同时，由于ext的数据存储在连续的块中，这可以减少缓存失效，而DBFS，由于需要进行索引，因此需要频繁地进行缺页处理，而这些页面会在缓存中不断地切换，同样这也会造成性能下降，在写测试中，数据产生的波动应该来自于次。其三，在并发测试中，DBFS的性能几乎没有变化，在实现中，每一次写操作都会变成一个写事务，而数据库的写事务将会使得文件被加锁，由于数据库的加锁粒度过大，导致在并发情况下数据库的写操作变成了跟单线程一样的串行操作。对于读操作来说，按理来说读事务不没有对文件加锁，因此其性能应该有较大的提升，但实验中不没有发生，其原因有待分析。



### Filebench 测试



#### webserver

EXT3

```
appendlog            22830ops      380ops/s   3.0mb/s    200.0ms/op [0.02ms - 713.35ms]
closefile10          22734ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.06ms]
readfile10           22734ops      379ops/s   5.9mb/s      0.7ms/op [0.01ms - 80.55ms]
openfile10           22735ops      379ops/s   0.0mb/s      5.3ms/op [0.05ms - 199.64ms]
closefile9           22735ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.04ms]
readfile9            22736ops      379ops/s   5.9mb/s      0.8ms/op [0.01ms - 253.92ms]
openfile9            22739ops      379ops/s   0.0mb/s      5.2ms/op [0.05ms - 381.75ms]
closefile8           22739ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.05ms]
readfile8            22739ops      379ops/s   5.9mb/s      0.8ms/op [0.01ms - 68.11ms]
openfile8            22743ops      379ops/s   0.0mb/s      5.3ms/op [0.05ms - 219.65ms]
closefile7           22743ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.08ms]
readfile7            22743ops      379ops/s   6.0mb/s      0.8ms/op [0.01ms - 124.87ms]
openfile7            22744ops      379ops/s   0.0mb/s      5.2ms/op [0.06ms - 127.65ms]
closefile6           22744ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.08ms]
readfile6            22744ops      379ops/s   6.0mb/s      0.8ms/op [0.01ms - 81.59ms]
openfile6            22746ops      379ops/s   0.0mb/s      5.2ms/op [0.06ms - 174.31ms]
closefile5           22746ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.08ms]
readfile5            22746ops      379ops/s   5.9mb/s      0.8ms/op [0.01ms - 95.46ms]
openfile5            22748ops      379ops/s   0.0mb/s      5.2ms/op [0.05ms - 117.25ms]
closefile4           22748ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
readfile4            22748ops      379ops/s   5.9mb/s      0.8ms/op [0.01ms - 218.10ms]
openfile4            22750ops      379ops/s   0.0mb/s      5.3ms/op [0.05ms - 351.52ms]
closefile3           22750ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.06ms]
readfile3            22750ops      379ops/s   5.9mb/s      0.9ms/op [0.01ms - 206.68ms]
openfile3            22753ops      379ops/s   0.0mb/s      5.3ms/op [0.05ms - 447.46ms]
closefile2           22753ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.03ms]
readfile2            22753ops      379ops/s   6.0mb/s      0.9ms/op [0.01ms - 383.29ms]
openfile2            22755ops      379ops/s   0.0mb/s      5.3ms/op [0.05ms - 412.19ms]
closefile1           22755ops      379ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.06ms]
readfile1            22755ops      379ops/s   5.9mb/s      1.1ms/op [0.01ms - 332.51ms]
openfile1            22758ops      379ops/s   0.0mb/s      6.6ms/op [0.05ms - 502.00ms]
151.484: IO Summary: 705196 ops 11751.459 ops/s 3790/380 rd/wr  62.3mb/s  23.9ms/op
151.484: Shutting down processes

```

EXT4

```
appendlog            21001ops      350ops/s   2.7mb/s    208.1ms/op [0.00ms - 1038.81ms]
closefile10          20905ops      348ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
readfile10           20905ops      348ops/s   5.5mb/s      0.8ms/op [0.01ms - 50.58ms]
openfile10           20907ops      348ops/s   0.0mb/s      6.8ms/op [0.10ms - 110.10ms]
closefile9           20907ops      348ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
readfile9            20907ops      348ops/s   5.5mb/s      0.8ms/op [0.01ms - 56.07ms]
openfile9            20910ops      348ops/s   0.0mb/s      6.8ms/op [0.09ms - 110.09ms]
closefile8           20910ops      348ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.05ms]
readfile8            20910ops      348ops/s   5.5mb/s      0.8ms/op [0.01ms - 48.68ms]
openfile8            20913ops      349ops/s   0.0mb/s      6.9ms/op [0.11ms - 155.75ms]
closefile7           20913ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
readfile7            20913ops      349ops/s   5.5mb/s      0.8ms/op [0.01ms - 76.55ms]
openfile7            20914ops      349ops/s   0.0mb/s      6.9ms/op [0.10ms - 151.70ms]
closefile6           20914ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
readfile6            20914ops      349ops/s   5.4mb/s      0.8ms/op [0.01ms - 81.33ms]
openfile6            20917ops      349ops/s   0.0mb/s      6.8ms/op [0.10ms - 156.10ms]
closefile5           20917ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.06ms]
readfile5            20917ops      349ops/s   5.5mb/s      0.9ms/op [0.01ms - 63.98ms]
openfile5            20920ops      349ops/s   0.0mb/s      6.7ms/op [0.11ms - 146.69ms]
closefile4           20920ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.15ms]
readfile4            20920ops      349ops/s   5.5mb/s      0.9ms/op [0.01ms - 57.11ms]
openfile4            20924ops      349ops/s   0.0mb/s      6.8ms/op [0.10ms - 154.38ms]
closefile3           20924ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.04ms]
readfile3            20924ops      349ops/s   5.5mb/s      0.9ms/op [0.01ms - 70.68ms]
openfile3            20926ops      349ops/s   0.0mb/s      6.7ms/op [0.10ms - 149.38ms]
closefile2           20926ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.08ms]
readfile2            20927ops      349ops/s   5.5mb/s      0.9ms/op [0.01ms - 60.42ms]
openfile2            20934ops      349ops/s   0.0mb/s      6.7ms/op [0.10ms - 156.18ms]
closefile1           20934ops      349ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.03ms]
readfile1            20935ops      349ops/s   5.4mb/s      0.8ms/op [0.01ms - 92.46ms]
openfile1            20939ops      349ops/s   0.0mb/s      7.4ms/op [0.12ms - 331.73ms]
185.355: IO Summary: 648547 ops 10808.340 ops/s 3486/350 rd/wr  57.3mb/s  26.0ms/op
185.355: Shutting down processes
```



```
appendlog            5040ops       84ops/s   0.6mb/s    413.1ms/op [0.70ms - 1105.60ms]
closefile10          4949ops       82ops/s   0.0mb/s      6.8ms/op [0.60ms - 117.72ms]
readfile10           4949ops       82ops/s   1.3mb/s     13.7ms/op [2.26ms - 297.85ms]
openfile10           4953ops       83ops/s   0.0mb/s     57.1ms/op [2.66ms - 508.19ms]
closefile9           4954ops       83ops/s   0.0mb/s      6.7ms/op [0.61ms - 51.54ms]
readfile9            4954ops       83ops/s   1.3mb/s     13.5ms/op [2.76ms - 296.21ms]
openfile9            4955ops       83ops/s   0.0mb/s     57.3ms/op [3.58ms - 491.47ms]
closefile8           4957ops       83ops/s   0.0mb/s      6.6ms/op [0.30ms - 24.02ms]
readfile8            4957ops       83ops/s   1.3mb/s     13.6ms/op [2.37ms - 128.06ms]
openfile8            4963ops       83ops/s   0.0mb/s     58.2ms/op [2.98ms - 430.99ms]
closefile7           4964ops       83ops/s   0.0mb/s      6.7ms/op [0.04ms - 205.43ms]
readfile7            4966ops       83ops/s   1.3mb/s     13.6ms/op [0.23ms - 123.46ms]
openfile7            4969ops       83ops/s   0.0mb/s     57.9ms/op [0.08ms - 369.27ms]
closefile6           4970ops       83ops/s   0.0mb/s      6.7ms/op [0.02ms - 81.96ms]
readfile6            4970ops       83ops/s   1.3mb/s     13.6ms/op [1.51ms - 320.62ms]
openfile6            4972ops       83ops/s   0.0mb/s     58.4ms/op [2.46ms - 494.66ms]
closefile5           4973ops       83ops/s   0.0mb/s      6.8ms/op [0.02ms - 278.91ms]
readfile5            4974ops       83ops/s   1.3mb/s     13.6ms/op [0.17ms - 312.75ms]
openfile5            4976ops       83ops/s   0.0mb/s     58.6ms/op [0.06ms - 417.86ms]
closefile4           4980ops       83ops/s   0.0mb/s      6.8ms/op [0.02ms - 112.18ms]
readfile4            4982ops       83ops/s   1.3mb/s     13.7ms/op [0.14ms - 251.37ms]
openfile4            4991ops       83ops/s   0.0mb/s     58.5ms/op [0.07ms - 508.75ms]
closefile3           4994ops       83ops/s   0.0mb/s      6.9ms/op [0.02ms - 275.97ms]
readfile3            4995ops       83ops/s   1.3mb/s     13.9ms/op [0.18ms - 320.57ms]
openfile3            5002ops       83ops/s   0.0mb/s     57.7ms/op [0.09ms - 441.49ms]
closefile2           5006ops       83ops/s   0.0mb/s      6.8ms/op [0.01ms - 236.64ms]
readfile2            5008ops       83ops/s   1.3mb/s     13.7ms/op [0.14ms - 322.59ms]
openfile2            5015ops       84ops/s   0.0mb/s     56.3ms/op [0.07ms - 472.81ms]
closefile1           5022ops       84ops/s   0.0mb/s      6.5ms/op [0.02ms - 74.03ms]
readfile1            5023ops       84ops/s   1.3mb/s     13.3ms/op [0.15ms - 297.82ms]
openfile1            5028ops       84ops/s   0.0mb/s     59.2ms/op [0.13ms - 429.64ms]
143.139: IO Summary: 154411 ops 2573.299 ops/s 830/84 rd/wr  13.6mb/s 109.1ms/op
143.139: Shutting down processes
```

| DBFS                     | EXT3                         | EXT4                        |
| ------------------------ | ---------------------------- | --------------------------- |
| 2573.299 ops/s  13.6mb/s | 11751.459 ops/s     62.3mb/s | 10808.340 ops/s    57.3mb/s |

在webserver测试中，主要是模拟简单的网络服务器 I/O 活动，在目录树中的多个文件上生成一系列打开-读取-关闭以及日志文件追加操作，默认使用 100 个线程。

在这个测试中，EXT3与EXT4表现出相当的性能，在60s的时间内，两者的IO次数与吞吐量差距在10%以内，ext3较ext4略微领先。说明这两种文件系统在这种元数据密集型且主要是打开和读取的操作的应用中性能几乎相同。





#### Mailserver

EXT3

```

finish               14285ops       39ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.02ms]
closefile4           14285ops       39ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.04ms]
readfile4            14285ops       39ops/s   0.5mb/s      4.1ms/op [0.02ms - 38.55ms]
openfile4            14285ops       39ops/s   0.0mb/s      8.9ms/op [0.13ms - 49.74ms]
closefile3           14285ops       39ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.05ms]
fsyncfile3           14285ops       39ops/s   0.0mb/s     12.0ms/op [0.67ms - 58.41ms]
appendfilerand3      14285ops       39ops/s   0.3mb/s     14.1ms/op [0.08ms - 63.41ms]
readfile3            14285ops       39ops/s   0.5mb/s      3.8ms/op [0.00ms - 34.98ms]
openfile3            14287ops       39ops/s   0.0mb/s      8.1ms/op [0.09ms - 59.44ms]
closefile2           14287ops       39ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.05ms]
fsyncfile2           14287ops       39ops/s   0.0mb/s     12.0ms/op [6.58ms - 75.29ms]
appendfilerand2      14287ops       39ops/s   0.3mb/s     13.2ms/op [0.12ms - 76.02ms]
createfile2          14294ops       39ops/s   0.0mb/s    168.3ms/op [1.88ms - 341.38ms]
deletefile1          14298ops       39ops/s   0.0mb/s    166.4ms/op [1.77ms - 363.28ms]
716.520: IO Summary: 185735 ops 504.671 ops/s 78/78 rd/wr   1.7mb/s 102.8ms/op
716.520: Shutting down processes

```



EXT4

```

finish               14284ops       43ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.03ms]
closefile4           14284ops       43ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.06ms]
readfile4            14284ops       43ops/s   0.6mb/s      3.9ms/op [0.02ms - 43.67ms]
openfile4            14287ops       43ops/s   0.0mb/s     18.4ms/op [0.12ms - 231.25ms]
closefile3           14287ops       43ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.08ms]
fsyncfile3           14289ops       43ops/s   0.0mb/s      7.3ms/op [0.72ms - 37.11ms]
appendfilerand3      14289ops       43ops/s   0.3mb/s     14.3ms/op [0.18ms - 57.98ms]
readfile3            14289ops       43ops/s   0.6mb/s      3.5ms/op [0.00ms - 28.31ms]
openfile3            14290ops       43ops/s   0.0mb/s     17.6ms/op [0.10ms - 237.23ms]
closefile2           14290ops       43ops/s   0.0mb/s      0.0ms/op [0.00ms - 18.36ms]
fsyncfile2           14290ops       43ops/s   0.0mb/s      7.3ms/op [2.78ms - 92.69ms]
appendfilerand2      14291ops       43ops/s   0.3mb/s     12.3ms/op [0.14ms - 65.55ms]
createfile2          14293ops       43ops/s   0.0mb/s    145.8ms/op [2.44ms - 265.70ms]
deletefile1          14299ops       43ops/s   0.0mb/s    145.5ms/op [3.32ms - 279.19ms]
771.564: IO Summary: 185762 ops 552.815 ops/s 85/85 rd/wr   1.9mb/s  94.0ms/op
771.564: Shutting down processes



```

```

finish               14285ops       64ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.02ms]
closefile4           14285ops       64ops/s   0.0mb/s      2.5ms/op [0.01ms - 35.56ms]
readfile4            14285ops       64ops/s   0.9mb/s      5.9ms/op [0.10ms - 57.46ms]
openfile4            14285ops       64ops/s   0.0mb/s     17.6ms/op [0.02ms - 224.65ms]
closefile3           14285ops       64ops/s   0.0mb/s      2.2ms/op [0.01ms - 24.20ms]
fsyncfile3           14285ops       64ops/s   0.0mb/s      1.9ms/op [0.00ms - 24.81ms]
appendfilerand3      14285ops       64ops/s   0.5mb/s      4.4ms/op [0.07ms - 35.74ms]
readfile3            14285ops       64ops/s   0.9mb/s      5.4ms/op [0.13ms - 57.31ms]
openfile3            14288ops       64ops/s   0.0mb/s     17.3ms/op [0.02ms - 238.93ms]
closefile2           14288ops       64ops/s   0.0mb/s      2.3ms/op [0.01ms - 26.09ms]
fsyncfile2           14288ops       64ops/s   0.0mb/s      1.8ms/op [0.01ms - 13.46ms]
appendfilerand2      14289ops       64ops/s   0.5mb/s      5.0ms/op [0.09ms - 32.31ms]
createfile2          14293ops       64ops/s   0.0mb/s     91.3ms/op [0.21ms - 247.15ms]
deletefile1          14300ops       64ops/s   0.0mb/s     90.4ms/op [0.33ms - 238.99ms]
274.165: IO Summary: 185741 ops 836.593 ops/s 129/129 rd/wr   2.8mb/s  62.0ms/op

```



| DBFS             | EXT3                     | EXT4                   |
| ---------------- | ------------------------ | ---------------------- |
| 836.593  2.8MB/S | 504.671 ops/s    1.7mb/s | 552.815 ops/s  1.9mb/s |



#### FileServer



EXT3

```

finish               16143ops      269ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.01ms]
statfile1            16143ops      269ops/s   0.0mb/s      5.6ms/op [0.02ms - 221.09ms]
deletefile1          16142ops      269ops/s   0.0mb/s     27.2ms/op [0.07ms - 489.94ms]
closefile3           16145ops      269ops/s   0.0mb/s      0.0ms/op [0.00ms - 60.44ms]
readfile1            16145ops      269ops/s  33.8mb/s      5.4ms/op [0.00ms - 138.55ms]
openfile2            16147ops      269ops/s   0.0mb/s      7.9ms/op [0.04ms - 370.82ms]
closefile2           16147ops      269ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.04ms]
appendfilerand1      16148ops      269ops/s   2.1mb/s     10.8ms/op [0.04ms - 367.31ms]
openfile1            16150ops      269ops/s   0.0mb/s      8.0ms/op [0.05ms - 417.57ms]
closefile1           16151ops      269ops/s   0.0mb/s      0.0ms/op [0.00ms - 53.40ms]
wrtfile1             16153ops      269ops/s  33.6mb/s     96.2ms/op [1.76ms - 1377.86ms]
createfile1          16186ops      270ops/s   0.0mb/s     24.4ms/op [0.10ms - 418.93ms]
135.936: IO Summary: 177657 ops 2960.717 ops/s 269/538 rd/wr  69.6mb/s  61.8ms/op
```



EXT4

```

finish               12455ops      208ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.02ms]
statfile1            12456ops      208ops/s   0.0mb/s      6.9ms/op [0.00ms - 249.00ms]
deletefile1          12455ops      208ops/s   0.0mb/s     34.9ms/op [0.12ms - 385.00ms]
closefile3           12460ops      208ops/s   0.0mb/s      0.0ms/op [0.00ms - 83.63ms]
readfile1            12461ops      208ops/s  26.0mb/s      5.6ms/op [0.00ms - 109.02ms]
openfile2            12461ops      208ops/s   0.0mb/s     10.1ms/op [0.07ms - 384.07ms]
closefile2           12462ops      208ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.24ms]
appendfilerand1      12462ops      208ops/s   1.6mb/s     13.6ms/op [0.05ms - 156.82ms]
openfile1            12464ops      208ops/s   0.0mb/s     10.2ms/op [0.06ms - 240.18ms]
closefile1           12464ops      208ops/s   0.0mb/s      0.0ms/op [0.00ms - 18.56ms]
wrtfile1             12464ops      208ops/s  26.0mb/s    127.1ms/op [0.67ms - 544.96ms]
createfile1          12497ops      208ops/s   0.0mb/s     31.7ms/op [0.18ms - 412.89ms]
92.499: IO Summary: 137106 ops 2284.932 ops/s 208/415 rd/wr  53.6mb/s  80.1ms/op
92.499: Shutting down processes

```

```

finish               4256ops       71ops/s   0.0mb/s      0.0ms/op [0.00ms -  0.07ms]
statfile1            4256ops       71ops/s   0.0mb/s     41.8ms/op [0.01ms - 771.14ms]
deletefile1          4259ops       71ops/s   0.0mb/s    216.2ms/op [0.19ms - 1371.18ms]
closefile3           4274ops       71ops/s   0.0mb/s     16.5ms/op [1.02ms - 258.23ms]
readfile1            4275ops       71ops/s   8.9mb/s     40.0ms/op [1.78ms - 602.01ms]
openfile2            4275ops       71ops/s   0.0mb/s     58.8ms/op [0.12ms - 1215.71ms]
closefile2           4280ops       71ops/s   0.0mb/s     17.8ms/op [0.02ms - 407.46ms]
appendfilerand1      4280ops       71ops/s   0.5mb/s     20.1ms/op [1.68ms - 411.84ms]
openfile1            4280ops       71ops/s   0.0mb/s     59.4ms/op [2.33ms - 1215.94ms]
closefile1           4291ops       72ops/s   0.0mb/s     17.1ms/op [0.04ms - 556.84ms]
wrtfile1             4292ops       72ops/s   8.9mb/s     35.3ms/op [3.96ms - 683.96ms]
createfile1          4295ops       72ops/s   0.0mb/s    176.4ms/op [3.16ms - 2169.02ms]
113.576: IO Summary: 47057 ops 784.215 ops/s 71/143 rd/wr  18.4mb/s 232.8ms/op
113.576: Shutting down processes

```

|            | DBFS                  | EXT3                    | EXT4                   |
| ---------- | --------------------- | ----------------------- | ---------------------- |
| fileserver | 784ops/s   18.4MB/s   | 2960ops/s   70MB/s      | 2284ops/s  54MB/s      |
| webserver  | 2573ops/s  13.6MB/s   | 11751ops/s    62.3MB/s  | 10808ops/s    57.3MB/s |
| mailserver | 836.593ops/s  2.8MB/S | 504.671ops/s    1.7MB/s | 552.815ops/s  1.9MB/s  |



## Reference

https://www.qiniu.com/qfans/qnso-21565865#comments tools

https://blog.csdn.net/pwl999/article/details/106787042  LTP

https://juicefs.com/docs/zh/community/posix_compatibility/#%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83

https://www.cnblogs.com/raykuan/p/6914748.html FIO

https://www.jianshu.com/p/e6892d808e94

https://github.com/hpc/ior ior+mdtest

https://blog.csdn.net/Neutionwei/article/details/108437857  mkfs.ext



https://fio.readthedocs.io/en/latest/fio_doc.html#running-fio

https://www.cnblogs.com/augusite/p/16178852.html  fio

https://brinnatt.com/projects/%E7%AC%AC-3-%E7%AB%A0-fio-%E7%A3%81%E7%9B%98%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95/



https://szp2016.github.io/linux/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7filebench/文件系统测试工具filebench
