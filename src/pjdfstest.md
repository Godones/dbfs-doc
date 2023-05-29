# pjdfstest

pjdfstest是一套简化版的文件系统POSIX兼容性测试套件，它可以工作在FreeBSD, Solaris, Linux上用于测试UFS, ZFS, ext3, XFS and the NTFS-3G等文件系统。目前pjdfstest包含八千多个测试用例，基本涵盖了文件系统的接口。

pjdfstest的项目地址与编译流程位于[pjdfstest](https://github.com/pjd/pjdfstest)

在挂载dbfs-fuse后，可以在挂载目录下运行测试程序得到dbfs的测试结果。

#### 测试结果

| test-set        | interface                 | pass/all  | error/all | error                    |
| --------------- | ------------------------- | --------- | --------- | ------------------------ |
| chflags         | chflags(FreeBSD)          | 14/14     | no        | linux下不工作            |
| chmod           | chmod/stat/symlink/chown  | 321/327   | 26/327    | chmod实现有误            |
| chown           | chown/chmod/stat          | 1280/1540 | 260/1540  | chmod实现有误            |
| ftruncate       | truncate/stat             | 88/89     | 1/89      | Access判断出错           |
| granular        | 未知                      | 7/7       | no        | linux下不工作            |
| link            | link/unlink/mknod         | 359/359   | no        | ok                       |
| mkdir           | mkdir                     | 118/118   | no        | ok                       |
| mkfifo          | mknod/link/unlink         | 120/120   | no        | ok                       |
| mknod           | mknod                     | 186/186   | no        | ok                       |
| open            | open/chmod/unlink/symlink | 328/328   | no        | ok                       |
| posix_fallocate | fallocate                 | 21/22     | 1/22      | Access判断错误           |
| rename          | rename/mkdir/             | 4458/4857 | 399/4857  | 错误返回值处理与逻辑错误 |
| rmdir           | rmdir                     | 139/145   | 6/145     | 错误处理，权限检查       |
| symlink         | symlink                   | 95/95     | no        | ok                       |
| truncate        | truncate                  | 84/84     | no        | ok                       |
| unlink          | unlink/link               | 403/440   | 37/440    | mknod中的socket,错误处理 |
| utimensat       | utimens                   | 121/122   | 1/122     | 权限检查                 |
|                 |                           | 7943/8674 | 731/8674  | 92%                      |



表显示了测试结果。在pjdfstest测试中，DBFS通过了92%的测例，这说明DBFS的整体实现正确，并且符合POSIX语义。在各个单独的测设集合中，DBFS全部通过了包括link、mkdir、open、truncate等在内的所有测试。DBFS发生错误的位置主要集中在rename和chown的处理当中，rename是文件系统中最复杂的操作，在各个文件系统中都是代码量很多的部分，DBFS实现并没有处理到位所有的细节，导致这部分出现了一部分错误，chown是权限检查的主要函数，linux中文件权限管理细节比较多，DBFS在实现中简化和忽略了一些判断，从而导致这部分部分测例也未通过。其他测试集合中未通过的测例较少，这里推测是由于其它测试集合中也包含了rename、chown这些操作，同时DBFS中对错误返回值的处理可能没有按照Posix标准进行返回。总体而言，DBFS的正确性处于一个较高的水平，与一些用户态文件系统相比，其对POSIX接口的兼容性具有更大的优势。
