# redis 数据类型：
[官方数据类型说明](http://www.redis.cn/topics/data-types.html "官方数据类型说明")

### 字符串
	字符串是所有编程语言中最常见的和最常用的数据类型，而且也是 redis 最基本的数据类型之一，而且 redis 中所有的 key 的类型都是字符串。
	Redis字符串是二进制并且是安全的，这意味着一个Redis字符串能包含任意类型的数据，例如： 一张JPEG格式的图片或者一个序列化的Ruby对象。
	一个字符串类型的值最多能存储512M字节的内容。

- 利用INCR命令簇（INCR, DECR, INCRBY）来把字符串当作原子计数器使用。

- 利用INCR命令簇（INCR, DECR, INCRBY）来把字符串当作原子计数器使用。

- 将字符串作为GETRANGE 和 SETRANGE的随机访问向量。

- 在小空间里编码大量数据，或者使用 GETBIT 和 SETBIT创建一个Redis支持的Bloom过滤器。

1. 添加一个key
	172.20.45.131:6379> set key_name value
	172.20.45.131:6379> MSET key1 value1 key2 value2（批量设置多个 key）

2. 获取一个key的值
	172.20.45.131:6379> get key_name
	172.20.45.131:6379> mget key1 key2...(批量获取多个 key)

3. 获取key的数据类型
	172.20.45.131:6379> type key_name

4. 设置key的自动过期时间
	172.20.45.131:6379> set key_name value ex #（单位是秒，超过指定的时间，key会自动销毁）

5. 删除一个key
	172.20.45.131:6379> del key_name （可以用空格隔开多个，同时删除多个key）

6. 追加数据：
	172.20.45.131:6379> append key_name str：把指定的str追加到原来的value后面

7. 数值递增
	172.20.45.131:6379> set num 10
	172.20.45.131:6379> incr num ：每执行一次，就会使这个值加1

8. 数值递减
	172.20.45.131:6379> set num 10
	172.20.45.131:6379> decr num ：每执行一次，就会使这个值减1

9. 返回指定key的值的长度
	172.20.45.131:6379> strlen key_name

### 键命令
查找键值,支持正则表达式

1. 查看所有的键
127.0.0.1:6378> keys *

2. 正则表达式匹配
127.0.0.1:6379> keys *name*

3. 判断键值是否存在: 存在返回1,不存在返回0
127.0.0.1:6379> EXISTS username
(integer) 1

4. 设置键值的过期时间
127.0.0.1:6379> EXPIRE username 3

5. 查看键值的有效时间: -1 表示永久有效
127.0.0.1:6379> TTL username2
(integer) -1
127.0.0.1:6379> TTL username4
(integer) 35994


### 列表
	列表是一个双向可读写的管道，其头部是左侧，尾部是右侧，一个列表最多可以包含 2^32-1 个元素即4294967295 个元素。

1. 生成列表并插入数据
	172.20.45.131:6379> lpush list_name str1 str2 str3.....
		生成的数据从左到右为：str3 str2 str1
	172.20.45.131:6379> rpush list_name str1 str2 str3.....
		生成的数据从左到右为：str1 str2 str3

2. 向列表追加数据
	172.20.45.131:6379> lpush list_name str：插入数据到最左侧
	172.20.45.131:6379> rpush list_name str：追加数据在最右侧

3. 获取列表长度
	172.20.45.131:6379> llen list_name

4. 移除列表数据：
	172.20.45.131:6379> lpop list_name：移除列表最左侧的数据
	172.20.45.131:6379> rpop list_name：移除列表最右侧的数据
	
5. 在指定的值后面插入数据
127.0.0.1:6379> LINSERT key_name before 现有元素 要插入的元素    
(integer) 4

6. 获取列表数据

127.0.0.1:6379> lrange key_name start stop

- start:表示起始位置的下标,0为第一个元素
- stop:表示结束位置的下标,-1为最后一个元素
  
7. 设置指定位置额元素的值
127.0.0.1:6379> lset key_name 元素的位置 要设置的元素的值

8. 删除指定位置的元素
127.0.0.1:6379> LREM key_name count  要移除的元素

- count: 表示要移除的元素的个数
- count 大于 0 : 表示从列表的左侧开始,移除指定个数的指定元素
- count 小于 0 : 表示从列表的右侧开始,移除指定个数的指定元素
- count 等于 0 : 表示移除列表中,指定的所有元素(注意不是所有元素,而是指定的元素都会被移除)


### 集合
	Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

1. 生成集合key：
	172.20.45.131:6379> sadd set_name value...

2. 追加数值：不能追加已存在的数值
	172.20.45.131:6379> sadd set_name value

3. 查看集合的所有数据
	172.20.45.131:6379>  smembers set_name

4. 获取集合的差集：
	172.20.45.131:6379> sdiff set1 set2：属于set1但是不属于set2的数据

5. 获取集合的交集
	172.20.45.131:6379> sinter set1 set2：既属于set1又属于set2的值

6. 获取集合的并集
	172.20.45.131:6379> sunion set1 set2 ：属于set1或属于set2中的数据

7. 删除指定的元素
srem key value



### 有序集合
	Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员，不同的是每个元素都会关联一个 double(双精度浮点型)类型的分数，redis 正是通过分数来为集合中的成员进行从小到大的排序，有序集合的成员是唯一的,但分数(score)却可以重复，集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)， 集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储 40 多亿个成员)。

1. 生成有序集合
	172.20.45.131:6379> zadd zset_name n value：其中n的值为value的分数

2. 显示指定集合内所有key和分数情况：
	172.20.45.131:6379>  ZADD paihangbang 10 key1 20 key2 30 key3
	(integer) 3
	172.20.45.131:6379> ZREVRANGE paihangbang 0 -1 withscores 
	1) "key3"
	2) "30"
	3) "key2"
	4) "20"
	5) "key1"
	6) "10"

3. 批量添加多个数值：
	172.20.45.131:6379> ZADD zset2 1 v1 2 v2 4 v3 5 v5

4. 获取集合的长度
	172.20.45.131:6379> zcard zset_name

5. 基于索引返回数值：索引从0开始
	172.20.45.131:6379> zrange zset_name n n：查看指定索引范围内的数据

6. 返回某个值的索引
	172.20.45.131:6379> zrank zset_name value

7. 返回score值在min和max之间的成员
zrangebyscore key min ma

8. 返回成员member的score值
zscore key membe

9. 删除指定的元素
zrem key member1 member2 ..

10. 删除权重在指定范围的元素
zremrangebyscore key min max

### 哈希（hash）
	hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象,Redis 中每个 hash 可以存储 23^2 - 1 键值对（40 多亿）。

1. 生成hash
	127.0.0.1:6379> HSET hset1 name tom age 18

2. 获取hash key字段值
	127.0.0.1:6379> HGET hset1 name
	"tom"
	127.0.0.1:6379> HGET hset1 age
	"18"

3. 删除一个 hash key 的字段：
	127.0.0.1:6379> HDEL hset1 age

4. 获取所有 hash 表中的字段：
	127.0.0.1:6379> HSET hset1 name tom age 19
	127.0.0.1:6379> HKEYS hset1
	1) "name"
	2) "age"
	
5.  查看一个hash有多少个属性
127.0.0.1:6379> HLEN user1
(integer) 3