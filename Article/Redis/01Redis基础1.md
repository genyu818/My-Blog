## Redis基础: 准备



## 1.	什么是Redis

Redis是一种NoSQL，什么是NoSQL呢？

我们传统型数据库例如MySql，Oracle等都是关系型数数据库。简单来说，传统型数据库，都是遵循严格的格式，每一行都是一条数据，每一列都是一个字段。Redis则是一种Key-Value数据库，可以存储键值对(刚开始学习时，可以粗略的理解为Map<String， Object>)。

再来是非常重要的一点，Redis是由C语言开发完成的数据库，同时Redis的数据存在内存当中，所以它的读写速度非常快，因此现在Redis被广泛用到缓存中。

总结一下Redis特点

1. 数据写在**内存**中，读取速度非常快
2. 可以存储非关系型数据，Redis提供了多种数据结构来支持多种业务场景



## 2.	Redis安装



## 3.	Redis数据结构

Redis支持5中基本类型，以及三种特殊类型。接下来我们会详细介绍Redis中的数据类型以及常用API

常用数据类型

1. String
2. List
3. hash
4. set
5. Zset

特殊数据类型

1. geospatial
2. HyperLogLog
3. BitMap

