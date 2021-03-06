# Redis

> redis 可以存储key与5种不同类型的值（value）之间的映射。可以将存储在内存的键值对数据持久化到硬盘。复制特性扩展读性能，客户端分片来扩展写性能。

## 数据结构

| 类型     | 结构存储值                                                   | 结构读写能力                                                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `String` | 字符串、整数、浮点数                                         | 对整个字符串或者字符串的其中一部分执行操作；对整个字符串和浮点数自增或者自减。 |
| `List`   | 链表，链表的每个节点包含一个字符串                           | 链表两端推入或者弹出；根据偏移量对链表进行修剪；读取单个或者多个元素；根据值查找或者移除元素。 |
| `Hash`   | 包含键值对的无序散列表                                       | 添加、获取、移除单个键值对；获取所有键值对。                 |
| `Set`    | 包含字符串的无序收集器，被包含的每个字符串都是独一无二的     | 添加、获取、移除单个元素；检查一个元素是否存在于集合中；计算交集、并集、差集；从集合里面随机获取元素。 |
| `Zset`   | 字符串成员与浮点数分值之间的有序映射，排列顺序由分值大小决定 | 添加、获取、移除单个元素；根据分值范围或者成员来获取元素。   |



## 附

[Redis In Action](https://www.manning.com/books/redis-in-action)

[Data types](https://redis.io/topics/data-types)

