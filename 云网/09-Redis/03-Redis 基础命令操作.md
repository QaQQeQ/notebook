
---

本章主要介绍 Redis 支持的常用数据类型及其核心操作命令。

# 通用连接与管理命令

- **关于数据库 `SELECT`**: Redis 默认提供 16 个逻辑数据库（0-15）。`SELECT n` 命令用于切换数据库，但这只是一个轻量级的**逻辑命名空间**，用于区分不同业务的数据，**并非用于安全或资源隔离**。生产环境中的多租户场景应使用独立的 Redis 实例。

| 命令                               | 说明                                    |
| :------------------------------- | :------------------------------------ |
| `redis-cli -h IP -p PORT -a PWD` | 远程连接 Redis 服务。                        |
| `INFO [section]`                 | 查看服务器详细信息（如 `memory`, `persistence`）。 |
| `DBSIZE`                         | 查看当前数据库中 key 的数量。                     |
| `SELECT n`                       | 切换到指定的数据库（n 从 0 到 15）。                |
| `TYPE key`                       | 查询指定 key 的数据类型。                       |

---

# Redis核心数据类型与命令

## 1. String (字符串) 类型

-   **简介**: Redis 最基础的 Key-Value 类型，value 可以是字符串、整数或浮点数。
-   **应用场景**: 缓存用户信息、配置、计数器、分布式锁等。

| 命令                           | 说明                  |
| :--------------------------- | :------------------ |
| `SET key value [EX seconds]` | 设置键值，并可选指定过期时间（秒）。  |
| `GET key`                    | 获取键的值。              |
| `DEL key`                    | 删除一个键。              |
| `INCR key`                   | 将键的值自增 1（键必须是整数类型）。 |
| `DECR key`                   | 将键的值自减 1（键必须是整数类型）。 |
| `APPEND key value`           | 在原有值的末尾追加字符串。       |
| `MGET key1 key2 ...`         | 批量获取多个键的值。          |

-   **`INCR` / `DECR` 典型应用**:
    -   **网站计数器**: 页面访问量、点赞数、文章阅读量。
    -   **库存控制**: 秒杀场景下的库存扣减。
    -   **唯一ID生成**: 生成全局唯一的订单号或流水号。

---

## 2. Hash (哈希) 类型

-   **简介**: 一个 key 对应多个 `field-value` 对的集合，非常适合存储对象。可以看作是 key-value 里面的 value 又是一个 key-value 集合。
-   **应用场景**: 存储用户信息（如姓名、年龄、城市）、商品属性、会话数据等。

| 命令                                           | 说明                    |
| :------------------------------------------- | :-------------------- |
| `HSET key field value`                       | 设置哈希表中一个字段的值。         |
| `HGET key field`                             | 获取哈希表中一个字段的值。         |
| `HGETALL key`                                | 获取哈希表中所有的字段和值。        |
| `HDEL key field`                             | 删除哈希表中的一个或多个字段。       |
| `HEXISTS key field`                          | 判断哈希表中指定字段是否存在。       |
| `HKEYS key`                                  | 获取哈希表中所有的字段名 (field)。 |
| `HVALS key`                                  | 获取哈希表中所有的值 (value)。   |
| `HMSET key field1 value1 field2 value2（将废弃）` | 批量设置                  |
| `HMGET key field1 field2 （将废弃）`              | 批量获取                  |

> **注意**: `HMSET` 和 `HMGET` 命令在 Redis 4.0 后已不推荐使用，`HSET` 和 `HGET` 现已支持多字段操作。

**示例**:
```redis
# 设置键值
hset user1 name henry age 22 city WH

# 获取键值
hget user1 name

# 获取所有键值
hgetall user1

# 删除city字段
hdel user1 city

# 判断某字段是否存在
HEXISTS user1 city
```

---

## 3. List (列表) 类型

-   **简介**: 一个双向链表，可以从两端推入（push）或弹出（pop）元素，元素可重复。
-   **应用场景**: 消息队列、任务队列、实时排行榜、最近浏览历史记录。

| 命令                              | 说明                           |
| :------------------------------ | :--------------------------- |
| `LPUSH key value1 [value2 ...]` | 从列表左侧（头部）推入一个或多个元素。          |
| `RPUSH key value1 [value2 ...]` | 从列表右侧（尾部）推入一个或多个元素。          |
| `LPOP key`                      | 从列表左侧弹出一个元素。                 |
| `RPOP key`                      | 从列表右侧弹出一个元素。                 |
| `LRANGE key start end`          | 获取列表中指定范围的元素（`0 -1` 表示获取所有）。 |
| `LLEN key`                      | 获取列表的长度。                     |
| `LREM key count value`          | 从列表中删除指定值的元素。                |

-   **`LREM` 命令中 `count` 的含义**:
    -   `count > 0`: 从左到右，删除 `count` 个值为 `value` 的元素。
    -   `count < 0`: 从右到左，删除 `|count|` 个值为 `value` 的元素。
    -   `count = 0`: 删除所有值为 `value` 的元素。

**示例**:
```redis
# 左侧入栈设置键值
LPUSH mylist "a" "b" "c"

# 列表现在: c b a
# 从右弹出第一个元素
RPOP mylist
输出: "a"

# 获取区间所有元素（从头到尾）
LRANGE mylist 0 -1
输出: ["c","b"]

# 取第 0、1、2 个元素
LRANGE list 0 2

# 从第二个取到最后
LRANGE list 1 -1

# 取倒数第 3 到最后
LRANGE list -3 -1
```

---

## 4. Set (无序集合) 类型

-   **简介**: 一个无序且元素唯一的集合，类似于数学中的集合。
-   **应用场景**: 去重、标签系统、好友关系、黑名单、随机抽奖。

| 命令                               | 说明                    |
| :------------------------------- | :-------------------- |
| `SADD key member1 [member2 ...]` | 向集合中添加一个或多个成员。        |
| `SREM key member`                | 从集合中移除一个或多个成员。        |
| `SMEMBERS key`                   | 获取集合中的所有成员。           |
| `SISMEMBER key member`           | 判断一个成员是否存在于集合中。       |
| `SCARD key`                      | 获取集合的成员数量。            |
| `SPOP key`                       | 随机弹出一个或多个成员（会从集合中删除）。 |
| `SRANDMEMBER key [count]`        | 随机获取一个或多个成员（不会删除）。    |
| `SUNION/SINTER/SDIFF`            | 执行并集、交集、差集运算。         |

**示例**: 
```redis
# 添加集合
SADD fruits "apple" "banana" "orange"

# 删除集合元素
SREM fruits "banana"

# 判断元素是否存在
SISMEMBER fruits "banana"
# 输出: 0

# 获取现在集合元素
SMEMBERS fruits
# 输出: apple, orange

# 随机获取2个数元素
SRANDMEMBER fruits 2

# 并/交/查
SUNION 并集天然去重
SINTER/SDIFF
```

---

## 5. Sorted Set (ZSet, 有序集合) 类型

-   **简介**: 类似于 Set，但每个成员都关联一个 `score` (分数)，Redis 会根据分数对成员进行排序。成员唯一，但分数可以重复。
-   **应用场景**: 排行榜、实时热门榜单、带权重的任务队列、延时任务。

| 命令                                         | 说明                             |
| :----------------------------------------- | :----------------------------- |
| `ZADD key score1 member1 [score2 member2]` | 添加或更新一个或多个成员及其分数。              |
| `ZRANGE key start end [WITHSCORES]`        | 按排名范围获取成员（`WITHSCORES` 可显示分数）。 |
| `ZRANGEBYSCORE key min max [WITHSCORES]`   | 按分数范围获取成员。                     |
| `ZREM key member`                          | 删除一个或多个成员。                     |
| `ZCARD key`                                | 获取有序集合的成员数量。                   |
| `ZSCORE key member`                        | 获取指定成员的分数。                     |
| `ZINCRBY key increment member`             | 为指定成员的分数增加 `increment`。        |

**示例**: 
```redis
# 设置集合
ZADD scores 100 "Alice" 200 "Bob" 150 "Tom"

# 输出全部集合内容
ZRANGE scores 0 -1 WITHSCORES
# 输出: Alice 100, Tom 150, Bob 200

# 输出指定范围值的用户
ZRANGEBYSCORE scores 120 200
# 输出: Tom, Bob

# 为Alice用户增加 50 （也可以是 -50 减少）
ZINCRBY scores 50 "Alice"
ZSCORE scores "Alice"
# 输出: 150

# 删除成员 Bob
ZREM scores Bob
```

---

## 6. Bitmap (位图)类型

-   **简介**: Bitmap 并非一个独立的数据类型，而是**基于 String 类型**的一种扩展。它将一个**String**看作是一个由二进制位（0 或 1）组成的数组，并提供了直接操作这些位的方法即**按位运算**。这种结构在处理大规模**boolean**状态时极为高效且节省空间。
-   **应用场景**:
    *   **用户签到统计**: 记录用户每天的签到状态，一个 key 即可管理所有用户一天的签到数据。
    *   **活跃用户统计**: 统计日活跃用户（DAU）或月活跃用户（MAU）。
    *   **在线状态**: 记录大量用户的在线或离线状态。
    *   **功能开关/灰度发布**: 标记用户是否具有访问某个新功能的权限。

| 命令                                        | 说明                                                      |
| :---------------------------------------- | :------------------------------------------------------ |
| `SETBIT key offset value`                 | 设置指定偏移量（offset）上的位的值为 `value`（0 或 1）。                   |
| `GETBIT key offset`                       | 获取指定偏移量（offset）上的位的值。                                   |
| `BITCOUNT key [start end]`                | 统计指定范围内被设置为 1 的位的数量。                                    |
| `BITOP operation destkey key1 [key2 ...]` | 对多个 Bitmap 执行位运算（如 AND, OR, XOR, NOT），并将结果存入 `destkey`。 |

**示例**: 
```redis
# 用户 1001 和 1002 签到
SETBIT sign:20251118 1001 1  
SETBIT sign:20251118 1002 1

# 查询用户是否签到
GETBIT sign:20251118 1001

# 当日签到总人数
BITCOUNT sign:20251118

# 用户活跃统计
SETBIT dau:20251118 1001 1
SETBIT dau:20251119 1001 1
# 统计某月活跃用户
BITOP OR dau:nov dau:20251118 dau:20251119  
BITCOUNT dau:nov

# 用户 12345 开启了测试功能
SETBIT feature:beta 12345 1

# 查询用户是否有该功能
GETBIT feature:beta 12345
```

---

## 7. HyperLogLog (基数统计)

-   **简介**: HyperLogLog 是一种**概率算法**，用来估算大规模集合的基数（不同元素的数量），在不保存所有元素的情况下，用很少的内存得到一个近似值。最初的算法叫 LogLog。核心思想：用哈希函数把元素映射成二进制串,然后统计二进制串前导零的最大长度，也就是看哈希值开头有多少个连续 0，概率上，前导零越多，集合元素越大，最后利用对数（log）关系来估算集合大小，因为使用了对数，所以叫 LogLog。后来对 LogLog 算法做了增强（Hyper），所以完整名字就是 HyperLogLog。
-   **应用场景**:
    *   **网站独立访客数 (UV)**: 统计每天、每周或每月的独立访客数量。
    *   **搜索引擎关键词统计**: 统计每日被搜索的独立关键词数量。
    *   **大规模数据去重**: 在海量日志或事件流中估算不重复的条目数。

| 命令 | 说明 |
| :--- | :--- |
| `PFADD key element [element ...]` | 将一个或多个元素添加到指定的 HyperLogLog 中。 |
| `PFCOUNT key [key ...]` | 返回一个或多个 HyperLogLog 的基数估算值。 |
| `PFMERGE destkey sourcekey [sourcekey ...]` | 将多个 HyperLogLog 合并到一个新的 HyperLogLog 中，新 HLL 的基数是所有源 HLL 的并集。 |

**示例**: 
```redis
# 添加访问的用户名
PFADD dau:20251118 user_1001 user_1002
PFADD dau:20251119 user_1001 user_1003

# 当日独立用户数
PFCOUNT dau:20251118
PFMERGE dau:nov dau:20251118 dau:20251119

# 合并后月独立用户数
PFCOUNT dau:nov

每天的用户 ID 放入当天 HLL
PFCOUNT 得到日活、月活人数
内存占用极低，适合高并发大用户量
```

---

## Key 命令操作

| 命令                         | 说明                                      |
| :------------------------- | :-------------------------------------- |
| `KEYS pattern`             | 查找匹配模式的 key (如 **`KEYS *`**查询当前库所以key)。 |
| `EXPIRE key seconds`       | 为 key 设置过期时间。                           |
| `TTL key`                  | 查看 key 的剩余过期时间 (-1 表示永不过期, -2 表示已删除)。   |
| `PERSIST key`              | 移除 key 的过期时间。                           |
| `EXISTS key`               | 判断 key 是否存在。                            |
| `FLUSHDB`                  | 清空当前库的所有 key。                           |
| `FLUSHALL`                 | 清空所有库的所有 key。                           |

**示例**:
```redis
# 添加数据
set salary 5000

# 针对键设置过期时间
EXPIRE salary 50

# 也可以在设置值时候带上过期时间
set salary 5000 EX 50

# 查看键剩余时间
ttl salary

# 移除过期时间
PERSIST salary

# 清空当前库
flushdb

# 清空所有库
flushall

默认 0-15 共计 16 个逻辑数据库
```

---

## 持久化及安全配置管理命令

| 命令                         | 说明          |
| -------------------------- | ----------- |
| `BGSAVE`                   | 后台保存 RDB 快照 |
| `SAVE`                     | 前台保存 RDB 快照 |
| `BGREWRITEAOF`             | 重写 AOF 文件   |
| `LASTSAVE`                 | 上次保存 RDB 时间 |
| `requirepass PASSWORD`     | 设置访问密码      |
| `bind 127.0.0.1`           | 绑定指定 IP     |
| `rename-command CONFIG ""` | 禁用危险命令      |
