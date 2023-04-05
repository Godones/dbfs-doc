# 讨论记录

## 2023.3.8

1. 叶子键值对：一个文件；文件的一部分；一个目录；没有内部结构；
2. 桶（非叶子键值对）：一个文件、文件的一部分；一个目录；有内部结构；
3. 桶的组织结构：基本属性、可选属性
4. 桶的组织结构：用“系统的解析函数”来解析基本属性；用“用户的解析函数”来解析可选属性；用户定义的解析函数可以委托系统来执行；（应用和系统的界线变得灵活了）
5. VFS可以一个缓存机构；对热点数据的缓存；

### 后续工作

1. 了解数据库的存储结构（一个键值对的数据结构也是树？）和检索方法；
2. 整理文档，形成设计方案的第二个版本。



## 2023.3.17

存在的问题：

1. 没有充分利用数据库已经存在的优势
   1. 如何体现出数据库的事务特性？
   2. 如何利用数据库查找速度快的优势？
2. 对于磁盘读写的优化还不够清晰明了
   1. 数据库内部在不存在对磁盘读写优化的情况下，如何改善？

### 后续的工作

1. 先解决当前出现的一些问题，并将已经有的设计实现，验证可行性