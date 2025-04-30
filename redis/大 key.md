## 什么是大 key

大 key 并不是指 key 的值很大，而是指 key 对应的 value 很大

一般而言，下面两种情况为大 key：

1. [String](数据类型#string) 类型的值大于 10 KB
2. [Hash](数据类型#Hash)、[List](数据类型#List)、[Set](数据类型#Set)、[ZSet](数据类型#ZSet) 类型元素的个数超过 5000 个

## 如何找到大 key

### 通过 redis-cli --bigkeys 命令查找大 key

```shell
redis-cli -h 127.0.0.1 -p 6379 -a "password" --bigkeys
```

注意事项：

1. 最好选择在从节点上执行该命令，因为在主节点上执行时，会阻塞主节点
2. 如果没有从节点，可以选择在 Redis 实例业务压力大低峰阶段进行扫描查询，以免影响到实例的正常运行；或者使用 -i 参数控制扫描间隔，避免长时间扫描降低 Redis 实例的性能

不足之处：

1. 这个方法只能返回每种类型中最大的那个 bigkey，无法得到大小排在前 N 位的 bigkey
2. 对于集合类型来说，这个方法只统计集合元素个数的多少，而不是实际占用的内存量。但是，一个集合的元素个数多，并不一定占用的内存就多

### 使用 SCAN 命令查找大 key

使用 SCAN 命令对数据库扫描，然后用 TYPE 命令获取返回的每一个 key 的类型

对于 [String](数据类型#string) 类型，可以使用 STRLEN 命令获取字符串的长度，也就是所占用的内存空间字节数

对于集合类型，有两种方法可以获取占用的内存大小：

1. 如果能够预先从业务层知道集合元素的平均大小，那么可以使用下列命令获取集合元素的个数，然后乘以集合元素的平均大小：
	1. [List](数据类型#List)：LLEN
	2. [Hash](数据类型#Hash)：HLEN
	3. [Set](数据类型#Set)：SCARD
	4. [ZSet](数据类型#ZSet)：ZCARD
2. 如果不能提前知道写入集合的元素大小，可以使用 MEMORYUSAGE 命令，查询一个键值对占用的内存空间

### 使用 RdbTools 工具查找大 key

例如，将大于 10 kb 的 key 输出到一个表格文件

```shell
rdb dump.rdb -c memory --bytes 10240 -f redis.csv
```

## 如何删除大 key

### 分批次删除

1. 对于 Hash，使用 HSCAN 扫描法
2. 对于 Set，使用 SRANDMEMBER 每次随机取数据删除
3. 对于 ZSet，可以使用 ZREMRANGEBYRANK 命令直接删除
4. 对于 List，使用 POP

### 异步删除

用 UNLINK 命令代替 DEL 删除，Redis 会将这个 key 放入一个异步线程中进行删除，不会阻塞主线程