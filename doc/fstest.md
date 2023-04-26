# 文件系统性能测试

1. 读写速度测试：测试文件系统的读写速度，包括顺序读写和随机读写，可以使用工具如IOzone、FIO、dd等。
2. 并发测试：测试文件系统在并发访问下的性能表现，可以使用工具如dbench、fs_mark、PostMark等。
3. 压力测试：测试文件系统在高负载下的性能表现，可以使用工具如stress、sysbench等。



要编写文件系统性能测试程序，您可以使用以下步骤：

1. 创建一个测试用的大型文件或多个小型文件。
2. 写入测试数据并记录时间，以测量写入性能。
3. 读取文件并记录时间，以测量读取性能。
4. 进行文件复制和移动操作，并记录时间以测量复制和移动性能。
5. 测试文件系统的并发读写性能。为此，可以同时运行多个线程或进程，并测量它们的执行时间。
6. 可以使用第三方工具来进行压力测试，例如使用IOmeter进行随机读写测试或使用FIO进行各种读写操作的测试。
7. 可以使用不同大小的文件来测试文件系统的处理能力，以确定其性能如何随着文件大小的变化而变化。
8. 可以在不同的存储设备（例如硬盘驱动器、闪存驱动器和固态硬盘）上运行测试，并比较它们的性能表现。
9. 最后，可以使用统计工具来分析性能测试结果，并确定瓶颈所在。这可以帮助您找到文件系统中的性能问题并解决它们。

文件系统性能测试可以使用多个指标来衡量，下面是一些常见的文件系统测试指标：

1. 延迟（Latency）：文件系统读取或写入数据所需的时间。延迟是衡量文件系统性能的重要指标，因为用户通常关心操作完成所需的时间。
2. 吞吐量（Throughput）：文件系统在一定时间内读取或写入数据的速率。吞吐量通常用MBps或GBps表示。
3. IOPS（每秒输入/输出操作数）：文件系统在一秒钟内处理的读写操作数。IOPS通常是衡量闪存存储设备性能的重要指标。
4. 并发性（Concurrency）：文件系统同时处理多个读写操作的能力。并发性是在高负载条件下测试文件系统性能的关键指标。
5. 可靠性（Reliability）：文件系统出现故障或意外情况时的恢复能力。例如，当系统意外断电时，文件系统应该能够在重新启动后恢复数据。
6. 一致性（Consistency）：文件系统保持数据一致性的能力。例如，当多个用户同时读取和写入文件时，文件系统应该能够正确处理数据并保持一致性。
7. 安全性（Security）：文件系统保护数据不被非授权用户访问的能力。例如，文件系统应该支持文件权限和访问控制列表（ACL）。

测试指标的选择取决于文件系统的具体用途和性能需求。在选择测试指标时，应该考虑到测试环境、应用场景和用户需求，以便正确地评估文件系统的性能。

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define FILE_SIZE_MB 1024 // 文件大小（MB）
#define BLOCK_SIZE_KB 4 // 块大小（KB）

// 生成指定大小的文件
void generate_file(const char* filename, int size_mb) {
    FILE* fp = fopen(filename, "wb");
    char* block = (char*)malloc(BLOCK_SIZE_KB * 1024);
    for (int i = 0; i < BLOCK_SIZE_KB * 1024; i++) {
        block[i] = i % 256; // 用0~255的随机数填充块数据
    }
    for (int i = 0; i < size_mb * 1024 / BLOCK_SIZE_KB; i++) {
        fwrite(block, BLOCK_SIZE_KB * 1024, 1, fp);
    }
    free(block);
    fclose(fp);
}

// 测试吞吐量
void test_throughput(const char* filename) {
    FILE* fp = fopen(filename, "rb");
    char* block = (char*)malloc(BLOCK_SIZE_KB * 1024);
    clock_t start = clock(); // 记录开始时间
    while (fread(block, BLOCK_SIZE_KB * 1024, 1, fp)) {} // 循环读取块数据
    clock_t end = clock(); // 记录结束时间
    free(block);
    fclose(fp);
    double duration = (double)(end - start) / CLOCKS_PER_SEC; // 计算读取时间（秒）
    double throughput = FILE_SIZE_MB / duration; // 计算吞吐量（MBps）
    printf("Throughput: %.2f MBps\n", throughput);
}

int main() {
    const char* filename = "testfile";
    generate_file(filename, FILE_SIZE_MB);
    test_throughput(filename);
    return 0;
}

```



## 顺序读性能

```rust
const FILE_SIZE: usize = 1024 * 1024 * 10;

#[cfg(feature = "8k")]
const BLOCK_SIZE: usize = 8192;
#[cfg(feature = "4k")]
const BLOCK_SIZE: usize = 4096;
#[cfg(feature = "1k")]
const BLOCK_SIZE: usize = 1024;
#[cfg(feature = "512")]
const BLOCK_SIZE: usize = 512;
#[cfg(feature = "256")]
const BLOCK_SIZE: usize = 256;
```

先往文件中写入10MB大小的数据，在将10MB大小的数据全部读出

### 测试结果

```rust
//fat32
time cost = 12777ms, read speed = 820KB/s
//dbfs
time cost = 22812ms, read speed = 459KB/s
```

## 顺序写性能

```c
SIZE:10MB
BLOCK_SIZE:1024B
```

#### 测试结果

```rust
time cost = 12632ms, write speed = 830KB/s
time cost = 25085ms, write speed = 418KB/s
```







## 随机读写性能

1. 打开文件：使用文件 I/O 函数打开测试文件，并将其指针保存在变量中。
2. 生成随机数：使用随机数生成函数生成随机数，用于确定读取或写入文件的位置和大小。
3. 进行随机读写：使用文件 I/O 函数对文件进行随机读写，根据生成的随机数确定读取或写入的位置和大小。要进行随机读写测试，应使用随机位置和随机大小进行读写操作，而不是顺序读写操作。
4. 记录测试结果：在测试期间，应记录每个操作的完成时间、读取或写入的位置和大小等信息，以便后续分析测试结果。
5. 分析测试结果：测试完成后，可以分析测试结果以了解文件系统的随机读写性能。可以计算测试期间每秒钟完成的操作数、延迟、带宽等指标，并将它们与其他文件系统进行比较。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <time.h>

#define TESTFILE "testfile"    // 测试文件名
#define TESTSIZE (1024 * 1024) // 测试文件大小（1MB）

int main()
{
    char buf[4096]; // 缓冲区大小（4KB）
    int fd, i, j, offset, ret;
    double start, end, elapsed, ops, throughput;

    srand(time(NULL)); // 初始化随机数生成器

    // 创建测试文件
    fd = open(TESTFILE, O_CREAT | O_TRUNC | O_WRONLY, 0666);
    if (fd == -1)
    {
        perror("open");
        exit(1);
    }
    for (i = 0; i < TESTSIZE / sizeof(buf); i++)
    {
        if (write(fd, buf, sizeof(buf)) == -1)
        {
            perror("write");
            exit(1);
        }
    }
    close(fd);

    // 打开测试文件
    fd = open(TESTFILE, O_RDONLY);
    if (fd == -1)
    {
        perror("open");
        exit(1);
    }

    // 随机读取测试
    start = clock();
    for (i = 0; i < 1000000; i++)
    {
        offset = rand() % TESTSIZE;
        if (lseek(fd, offset, SEEK_SET) == -1)
        {
            perror("lseek");
            exit(1);
        }
        if (read(fd, buf, sizeof(buf)) == -1)
        {
            perror("read");
            exit(1);
        }
    }
    end = clock();
    elapsed = (end - start) / CLOCKS_PER_SEC;
    ops = i / elapsed;
    throughput = ops * sizeof(buf) / (1024 * 1024);
    printf("Random read test:\n");
    printf("Elapsed time: %.3f s\n", elapsed);
    printf("Operations per second: %.0f\n", ops);
    printf("Throughput: %.2f MB/s\n", throughput);

    // 随机写入测试
    start = clock();
    for (i = 0; i < 1000000; i++)
    {
        offset = rand() % TESTSIZE;
        if (lseek(fd, offset, SEEK_SET) == -1)
        {
            perror("lseek");
            exit(1);
        }
        if (write(fd, buf, sizeof(buf)) == -1)
        {
            perror("write");
            exit(1);
        }
    }
    end = clock();
    elapsed = (end - start) / CLOCKS_PER_SEC;
    ops = i / elapsed;
    throughput = ops * sizeof(buf) / (1024 * 1024);
    printf("Random write test:\n");
    printf("Elapsed time: %.3f s\n", elapsed);
    printf("Operations per second: %.0f\n", ops);
    printf("Throughput: %.2f MB/s\n", throughput);

    // 关闭测试文件
    close(fd);

    // 删除测试文件
    ret = remove(TESTFILE);
    if (ret == -1)
    {
        perror("remove");
        exit(1);
    }

    return 0;
}

```

### 测试条件

```rust
const FILE_SIZE: usize = 1024 * 1024 * 16;
//16MB
const BLOCK_SIZE: usize = 4096;

const ITER: usize = 1000_0;
```



### 测试结果

#### fat32

```rust
Random read test:
Elapsed time: 155409ms
Read: 39997KB
Throughput: 257.367002152385KB/s
Operations: 64.3463377281882ops/s

Random write test:
Elapsed time: 153839ms
Write: 40000KB
Throughput: 260.01209056221114KB/s
Operations: 65.00302264055279ops/s
```

#### dbfs

```rust
Random read test:
Elapsed time: 111441ms
Read: 39992KB
Throughput: 358.8639392144722KB/s
Operations: 89.73358099801689ops/s

Random write test:
Elapsed time: 132381ms
Write: 40000KB
Throughput: 302.15816469130766KB/s
Operations: 75.53954117282692ops/s
```







```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>

#define THREAD_NUM 10 // 线程数
#define FILE_SIZE_MB 1024 // 文件大小（MB）
#define BLOCK_SIZE_KB 4 // 块大小（KB）
#define WRITE_FREQ 1000 // 写入频率（毫秒）
#define FAULT_FREQ 3000 // 故障模拟频率（毫秒）

char* block = NULL; // 块数据
pthread_mutex_t lock; // 互斥锁
int running = 1; // 是否继续运行
int faulting = 0; // 是否正在模拟故障

// 生成指定大小的文件
void generate_file(const char* filename, int size_mb) {
    FILE* fp = fopen(filename, "wb");
    for (int i = 0; i < size_mb * 1024 / BLOCK_SIZE_KB; i++) {
        fwrite(block, BLOCK_SIZE_KB * 1024, 1, fp);
    }
    fclose(fp);
}

// 随机生成一个块数据
void random_block(char* block) {
    for (int i = 0; i < BLOCK_SIZE_KB * 1024; i++) {
        block[i] = rand() % 256; // 用0~255的随机数填充块数据
    }
}

// 写入数据
void write_data(const char* filename, int thread_id) {
    int fd = open(filename, O_WRONLY | O_APPEND);
    if (fd < 0) {
        perror("open");
        return;
    }
    while (running) {
        random_block(block);
        pthread_mutex_lock(&lock); // 加锁
        if (write(fd, block, BLOCK_SIZE_KB * 1024) < 0) {
            perror("write");
        }
        pthread_mutex_unlock(&lock); // 解锁
        usleep(WRITE_FREQ * 1000); // 等待写入频率
    }
    close(fd);
}

// 读取数据
void read_data(const char* filename, int thread_id) {
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return;
    }
    while (running) {
        char buf[BLOCK_SIZE_KB * 1024];
        pthread_mutex_lock(&lock); // 加锁
        if (read(fd, buf, BLOCK_SIZE_KB * 1024) < 0) {
            perror("read");
        }
        pthread_mutex_unlock(&lock); // 解锁
    }
    close(fd);
}

```



## fuse测试

### pjdfstest

pjdfstest是一套简化版的文件系统POSIX兼容性测试套件，它可以工作在FreeBSD, Solaris, Linux上用于测试UFS, ZFS, ext3, XFS and the NTFS-3G等文件系统。目前pjdfstest包含八千多个测试用例，基本涵盖了文件系统的接口。

pjdfstest的项目地址与编译流程位于[pjdfstest](https://github.com/pjd/pjdfstest)

在挂载dbfs-fuse后，可以在挂载目录下运行测试程序

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
| rename          | rename/mkdir/             | 3239/4857 | 1618/4857 | 错误返回值处理与逻辑错误 |
| rmdir           | rmdir                     | 139/145   | 6/145     | 错误处理，权限检查       |
| symlink         | symlink                   | 95/95     | :x:       | :heavy_check_mark:       |
| truncate        | truncate                  | 84/84     | :x:       | :heavy_check_mark:       |
| unlink          | unlink/link               | 403/440   | 37/440    | mknod中的socket,错误处理 |
| utimensat       | utimens                   | 121/122   | 1/122     | 权限检查                 |
|                 |                           | 6724/8674 | 1950/8674 | 77%                      |



#### 结果说明

在pjdfstest中，错误主要集中在rename和chown的处理当中，rename是其中最复杂的部分，本项目并没有处理所有细节，chown是权限检查的主要部分，实现细节也有待斟酌。还有出现错误的位置主要是错误处理当中，一些错误值没有按照Posix标准进行返回.



#### TODO

- [ ] 更准确的错误处理
- [ ] chown的细节处理, 似乎需要处理用户和当前进程的的关系
- [ ] rename的完整实现



### LTP测试

[LTP](https://github.com/linux-test-project/ltp)（Linux Test Project）是一个由 IBM，Cisco 等多家公司联合开发维护的项目，旨在为开源社区提供一个验证 Linux 可靠性和稳定性的测试集。LTP 中包含了各种工具来检验 Linux 内核和相关特性

安装方法与项目说明请查看[ltp](https://github.com/linux-test-project/ltp)

```
$ git clone https://github.com/linux-test-project/ltp.git
$ cd ltp
$ make autotools
$ ./configure
```



```
./runltp -d /mnt/jfs -f fs_bind,fs_perms_simple,fsx,io,smoketest,fs,syscalls
```





### FIO测试(性能测试)

顺序写：

```shell
fio --name=dbfs-test --directory=./dbfuse --ioengine=libaio --rw=write --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
```

顺序读

```bash
fio --name=jfs-test --directory=./dbfuse --ioengine=libaio --rw=read --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
```

随机写

```shell
fio --name=jfs-test --directory=./dbfuse --ioengine=libaio --rw=randwrite --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
```

随机读

```shell
fio --name=jfs-test --directory=./dbfuse --ioengine=libaio --rw=randread --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
```

- `--name`：用户指定的测试名称，会影响测试文件名
- `--directory`：测试目录
- `--ioengine`：测试时下发 IO 的方式；通常用 libaio 即可
- `--rw`：常用的有 read，write，randread，randwrite，分别代表顺序读写和随机读写
- `--bs`：每次 IO 的大小
- `--size`：每个线程的 IO 总大小；通常就等于测试文件的大小
- `--numjobs`：测试并发线程数；默认每个线程单独跑一个测试文件
- `--direct`：在打开文件时添加 `O_DIRECT` 标记位，不使用系统缓冲，可以使测试结果更稳定准确

#### 测试结果

```
fio --name=dbfs-test --directory=./dbfuse  --rw=write --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
dbfs-test: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
fio-3.34-38-g0739
Starting 1 process
dbfs-test: Laying out IO file (1 file / 1024MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=18.0MiB/s][w=18 IOPS][eta 00m:00s]
dbfs-test: (groupid=0, jobs=1): err= 0: pid=183987: Wed Apr 26 23:46:37 2023
  write: IOPS=19, BW=19.6MiB/s (20.5MB/s)(1024MiB/52342msec); 0 zone resets
    clat (usec): min=40268, max=74446, avg=51065.55, stdev=2918.80
     lat (usec): min=40306, max=74486, avg=51109.32, stdev=2919.20
    clat percentiles (usec):
     |  1.00th=[45351],  5.00th=[46924], 10.00th=[47449], 20.00th=[48497],
     | 30.00th=[49546], 40.00th=[50070], 50.00th=[51119], 60.00th=[51643],
     | 70.00th=[52691], 80.00th=[53740], 90.00th=[54789], 95.00th=[55837],
     | 99.00th=[57410], 99.50th=[58459], 99.90th=[62653], 99.95th=[73925],
     | 99.99th=[73925]
   bw (  KiB/s): min=18432, max=22528, per=100.00%, avg=20046.77, stdev=1165.29, samples=104
   iops        : min=   18, max=   22, avg=19.58, stdev= 1.14, samples=104
  lat (msec)   : 50=38.48%, 100=61.52%
  cpu          : usr=0.28%, sys=0.23%, ctx=9220, majf=0, minf=8
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1024,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=19.6MiB/s (20.5MB/s), 19.6MiB/s-19.6MiB/s (20.5MB/s-20.5MB/s), io=1024MiB (1074MB), run=52342-52342msec
```

 

```
fio --name=dbfs-test --directory=./dbfuse  --rw=read --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
dbfs-test: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
fio-3.34-38-g0739
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=8192KiB/s][r=8 IOPS][eta 00m:00s]
dbfs-test: (groupid=0, jobs=1): err= 0: pid=184093: Wed Apr 26 23:49:26 2023
  read: IOPS=8, BW=8217KiB/s (8414kB/s)(1024MiB/127611msec)
    clat (msec): min=34, max=185, avg=124.61, stdev=31.51
     lat (msec): min=34, max=185, avg=124.62, stdev=31.51
    clat percentiles (msec):
     |  1.00th=[   35],  5.00th=[   39], 10.00th=[   44], 20.00th=[  129],
     | 30.00th=[  132], 40.00th=[  133], 50.00th=[  136], 60.00th=[  136],
     | 70.00th=[  138], 80.00th=[  140], 90.00th=[  144], 95.00th=[  146],
     | 99.00th=[  153], 99.50th=[  157], 99.90th=[  180], 99.95th=[  186],
     | 99.99th=[  186]
   bw (  KiB/s): min= 6144, max=26624, per=99.99%, avg=8216.09, stdev=3483.74, samples=255
   iops        : min=    6, max=   26, avg= 8.02, stdev= 3.40, samples=255
  lat (msec)   : 50=10.84%, 100=1.17%, 250=87.99%
  cpu          : usr=0.01%, sys=0.07%, ctx=8195, majf=0, minf=267
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=1024,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=8217KiB/s (8414kB/s), 8217KiB/s-8217KiB/s (8414kB/s-8414kB/s), io=1024MiB (1074MB), run=127611-127611msec
```



```
fio --name=dbfs-test --directory=./dbfuse  --rw=randwrite --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
dbfs-test: (g=0): rw=randwrite, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
fio-3.34-38-g0739
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=17.0MiB/s][w=17 IOPS][eta 00m:00s]
dbfs-test: (groupid=0, jobs=1): err= 0: pid=184411: Wed Apr 26 23:56:20 2023
  write: IOPS=17, BW=17.8MiB/s (18.7MB/s)(1024MiB/57570msec); 0 zone resets
    clat (usec): min=52612, max=63588, avg=56167.62, stdev=1490.51
     lat (usec): min=52653, max=63625, avg=56212.19, stdev=1491.04
    clat percentiles (usec):
     |  1.00th=[53216],  5.00th=[53740], 10.00th=[54264], 20.00th=[54789],
     | 30.00th=[55313], 40.00th=[55837], 50.00th=[56361], 60.00th=[56361],
     | 70.00th=[56886], 80.00th=[57410], 90.00th=[57934], 95.00th=[58459],
     | 99.00th=[60031], 99.50th=[61080], 99.90th=[63177], 99.95th=[63701],
     | 99.99th=[63701]
   bw (  KiB/s): min=16384, max=20480, per=100.00%, avg=18218.30, stdev=684.85, samples=115
   iops        : min=   16, max=   20, avg=17.79, stdev= 0.67, samples=115
  lat (msec)   : 100=100.00%
  cpu          : usr=0.17%, sys=0.24%, ctx=9218, majf=0, minf=7
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1024,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=17.8MiB/s (18.7MB/s), 17.8MiB/s-17.8MiB/s (18.7MB/s-18.7MB/s), io=1024MiB (1074MB), run=57570-57570msec
```



```
fio --name=dbfs-test --directory=./dbfuse  --rw=randread --bs=1m --size=1g --numjobs=1 --direct=1 --group_reporting
dbfs-test: (g=0): rw=randread, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
fio-3.34-38-g0739
Starting 1 process
Jobs: 1 (f=1): [r(1)][98.1%][r=24.0MiB/s][r=24 IOPS][eta 00m:01s]
dbfs-test: (groupid=0, jobs=1): err= 0: pid=184224: Wed Apr 26 23:50:37 2023
  read: IOPS=19, BW=19.8MiB/s (20.8MB/s)(1024MiB/51676msec)
    clat (msec): min=34, max=166, avg=50.46, stdev=27.74
     lat (msec): min=34, max=166, avg=50.46, stdev=27.74
    clat percentiles (msec):
     |  1.00th=[   35],  5.00th=[   35], 10.00th=[   35], 20.00th=[   36],
     | 30.00th=[   38], 40.00th=[   39], 50.00th=[   41], 60.00th=[   43],
     | 70.00th=[   45], 80.00th=[   49], 90.00th=[  107], 95.00th=[  129],
     | 99.00th=[  144], 99.50th=[  148], 99.90th=[  155], 99.95th=[  167],
     | 99.99th=[  167]
   bw (  KiB/s): min= 8192, max=28672, per=99.85%, avg=20261.28, stdev=4013.99, samples=103
   iops        : min=    8, max=   28, avg=19.79, stdev= 3.92, samples=103
  lat (msec)   : 50=81.64%, 100=6.35%, 250=12.01%
  cpu          : usr=0.01%, sys=0.18%, ctx=8194, majf=0, minf=267
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=1024,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=19.8MiB/s (20.8MB/s), 19.8MiB/s-19.8MiB/s (20.8MB/s-20.8MB/s), io=1024MiB (1074MB), run=51676-51676msec
```





### mdtest(元数据处理能力)

mdtest是一款针对服务器元数据处理能力的基准测试工具，可以用来模拟对文件或目录的open/stat/close操作，然后返回报告。它支持MPI，可以用来协调大量客户端对服务器发起请求。

```
./mdtest -d dbfuse -b 6 -I 8 -z 4
```

​	

#### 测试结果

DBFS

```
UMMARY rate: (of 1 iterations)
   Operation                     Max            Min           Mean        Std Dev
   ---------                     ---            ---           ----        -------
   Directory creation            434.840        434.840        434.840          0.000
   Directory stat              48396.356      48396.356      48396.356          0.000
   Directory rename              478.060        478.060        478.060          0.000
   Directory removal             427.715        427.715        427.715          0.000
   File creation                 408.680        408.680        408.680          0.000
   File stat                   43582.450      43582.450      43582.450          0.000
   File read                    5484.167       5484.167       5484.167          0.000
   File removal                  559.039        559.039        559.039          0.000
   Tree creation                 469.566        469.566        469.566          0.000
   Tree removal                 1989.708       1989.708       1989.708          0.000
-- finished at 04/26/2023 22:12:00 --
```

其他文件系统结果

S3FS

```
mdtest-3.4.0+dev was launched with 1 total task(s) on 1 node(s)
Command line used: ./mdtest '-d' '/s3fs/mdtest' '-b' '6' '-I' '8' '-z' '2'
WARNING: Read bytes is 0, thus, a read test will actually just open/close.
Path                : /s3fs/mdtest
FS                  : 256.0 TiB   Used FS: 0.0%   Inodes: 0.0 Mi   Used Inodes: -nan%
Nodemap: 1
1 tasks, 344 files/directories

SUMMARY rate: (of 1 iterations)
   Operation                      Max            Min           Mean        Std Dev
   ---------                      ---            ---           ----        -------
   Directory creation        :          5.977          5.977          5.977          0.000
   Directory stat            :        435.898        435.898        435.898          0.000
   Directory removal         :          8.969          8.969          8.969          0.000
   File creation             :          5.696          5.696          5.696          0.000
   File stat                 :         68.692         68.692         68.692          0.000
   File read                 :         33.931         33.931         33.931          0.000
   File removal              :         23.658         23.658         23.658          0.000
   Tree creation             :          5.951          5.951          5.951          0.000
   Tree removal              :          9.889          9.889          9.889          0.000
```

EFS

```
mdtest-3.4.0+dev was launched with 1 total task(s) on 1 node(s)
Command line used: ./mdtest '-d' '/efs/mdtest' '-b' '6' '-I' '8' '-z' '4'
WARNING: Read bytes is 0, thus, a read test will actually just open/close.
Path                : /efs/mdtest
FS                  : 8388608.0 TiB   Used FS: 0.0%   Inodes: 0.0 Mi   Used Inodes: -nan%
Nodemap: 1
1 tasks, 12440 files/directories

SUMMARY rate: (of 1 iterations)
   Operation                      Max            Min           Mean        Std Dev
   ---------                      ---            ---           ----        -------
   Directory creation        :        192.301        192.301        192.301          0.000
   Directory stat            :       1311.166       1311.166       1311.166          0.000
   Directory removal         :        213.132        213.132        213.132          0.000
   File creation             :        179.293        179.293        179.293          0.000
   File stat                 :        915.230        915.230        915.230          0.000
   File read                 :        371.012        371.012        371.012          0.000
   File removal              :        217.498        217.498        217.498          0.000
   Tree creation             :        187.906        187.906        187.906          0.000
   Tree removal              :        218.357        218.357        218.357          0.000
```



JuiceFs

```
mdtest-3.4.0+dev was launched with 1 total task(s) on 1 node(s)
Command line used: ./mdtest '-d' '/jfs/mdtest' '-b' '6' '-I' '8' '-z' '4'
WARNING: Read bytes is 0, thus, a read test will actually just open/close.
Path                : /jfs/mdtest
FS                  : 1024.0 TiB   Used FS: 0.0%   Inodes: 10.0 Mi   Used Inodes: 0.0%
Nodemap: 1
1 tasks, 12440 files/directories

SUMMARY rate: (of 1 iterations)
   Operation                      Max            Min           Mean        Std Dev
   ---------                      ---            ---           ----        -------
   Directory creation        :       1416.582       1416.582       1416.582          0.000
   Directory stat            :       3810.083       3810.083       3810.083          0.000
   Directory removal         :       1115.108       1115.108       1115.108          0.000
   File creation             :       1410.288       1410.288       1410.288          0.000
   File stat                 :       5023.227       5023.227       5023.227          0.000
   File read                 :       3487.947       3487.947       3487.947          0.000
   File removal              :       1163.371       1163.371       1163.371          0.000
   Tree creation             :       1503.004       1503.004       1503.004          0.000
   Tree removal              :       1119.806       1119.806       1119.806          0.000
```



#### 结果说明





#### TODO

- [ ] remove/rename太慢





## Reference

https://www.qiniu.com/qfans/qnso-21565865#comments tools

https://blog.csdn.net/pwl999/article/details/106787042  LTP

https://juicefs.com/docs/zh/community/posix_compatibility/#%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83

https://www.cnblogs.com/raykuan/p/6914748.html FIO

https://www.jianshu.com/p/e6892d808e94

https://github.com/hpc/ior ior+mdtest
