# DBFS项目介绍

DBFS是一个使用数据库作为后端实现的文件系统。



## 项目细节





### Feature

未来的工作可以分为几个阶段：

1. 进一步探索jammdb数据库本身的设计与实现，通过分析数据库中潜在的性能瓶颈，改善DBFS的设计和实现，使得文件系统的性能进一步提升。

2. 选择更强大，更完善的数据库来实现基于数据库的文件系统，并提供与现代文件系统相当的性能和可靠性
3. 在DBFS移植到Alien OS的基础上，进一步探索如何将DBFS完全移植到Linux内核中，而这里的核心是如何将数据库移植到内核中。
4. 探索如何将数据库的事务特性充分融合到操作系统当中

