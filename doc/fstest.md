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

## 顺序读写性能

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



### 测试结果

```rust
//fat32
time cost = 12777ms, read speed = 820KB/s
//dbfs
time cost = 22812ms, read speed = 459KB/s
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

