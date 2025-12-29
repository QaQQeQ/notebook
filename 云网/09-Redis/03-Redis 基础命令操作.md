
---

本章主要介绍 Redis 支持的常用数据类型及其核心操作命令。

#### **通用连接与管理命令**

- **关于数据库 `SELECT`**: Redis 默认提供 16 个逻辑数据库（0-15）。`SELECT n` 命令用于切换数据库，但这只是一个轻量级的**逻辑命名空间**，用于区分不同业务的数据，**并非用于安全或资源隔离**。生产环境中的多租户场景应使用独立的 Redis 实例。

| 命令 | 说明 |
| :--- | :--- |
| `redis-cli -h IP -p PORT -a PWD` | 远程连接 Redis 服务。 |
| `INFO [section]` | 查看服务器详细信息（如 `memory`, `persistence`）。 |
| `DBSIZE` | 查看当前数据库中 key 的数量。 |
| `SELECT n` | 切换到指定的数据库（n 从 0 到 15）。 |
| `TYPE key` | 查询指定 key 的数据类型。 |

---

### **Redis核心数据类型与命令**

#### **1. String (字符串) 类型**

-   **简介**: Redis 最基础的 Key-Value 类型，value 可以是字符串、整数或浮点数。
-   **应用场景**: 缓存用户信息、配置、计数器、分布式锁等。

| 命令 | 说明 |
| :--- | :--- |
| `SET key value [EX seconds]` | 设置键值，并可选地指定过期时间（秒）。 |
| `GET key` | 获取键的值。 |
| `DEL key` | 删除一个键。 |
| `INCR key` | 将键的值自增 1（键必须是整数类型）。 |
| `DECR key` | 将键的值自减 1（键必须是整数类型）。 |
| `APPEND key value` | 在原有值的末尾追加字符串。 |
| `MGET key1 key2 ...` | 批量获取多个键的值。 |

-   **`INCR` / `DECR` 典型应用**:
    -   **网站计数器**: 页面访问量、点赞数、文章阅读量。
    -   **库存控制**: 秒杀场景下的库存扣减。
    -   **唯一ID生成**: 生成全局唯一的订单号或流水号。

---

#### **2. Hash (哈希) 类型**

-   **简介**: 一个 key 对应多个 `field-value` 对的集合，非常适合存储对象。可以看作是 key-value 里面的 value 又是一个 key-value 集合。
-   **应用场景**: 存储用户信息（如姓名、年龄、城市）、商品属性、会话数据等。

| 命令 | 说明 |
| :--- | :--- |
| `HSET key field value` | 设置哈希表中一个字段的值。 |
| `HGET key field` | 获取哈希表中一个字段的值。 |
| `HGETALL key` | 获取哈希表中所有的字段和值。 |
| `HDEL key field` | 删除哈希表中的一个或多个字段。 |
| `HEXISTS key field` | 判断哈希表中指定字段是否存在。 |
| `HKEYS key` | 获取哈希表中所有的字段名 (field)。 |
| `HVALS key` | 获取哈希表中所有的值 (value)。 |

> **注意**: `HMSET` 和 `HMGET` 命令在 Redis 4.0 后已不推荐使用，`HSET` 和 `HGET` 现已支持多字段操作。

---

#### **3. List (列表) 类型**

-   **简介**: 一个双向链表，可以从两端推入（push）或弹出（pop）元素，元素可重复。
-   **应用场景**: 消息队列、任务队列、实时排行榜、最近浏览历史记录。

| 命令 | 说明 |
| :--- | :--- |
| `LPUSH key value1 [value2 ...]` | 从列表左侧（头部）推入一个或多个元素。 |
| `RPUSH key value1 [value2 ...]` | 从列表右侧（尾部）推入一个或多个元素。 |
| `LPOP key` | 从列表左侧弹出一个元素。 |
| `RPOP key` | 从列表右侧弹出一个元素。 |
| `LRANGE key start end` | 获取列表中指定范围的元素（`0 -1` 表示获取所有）。 |
| `LLEN key` | 获取列表的长度。 |
| `LREM key count value` | 从列表中删除指定值的元素。 |

-   **`LREM` 命令中 `count` 的含义**:
    -   `count > 0`: 从左到右，删除 `count` 个值为 `value` 的元素。
    -   `count < 0`: 从右到左，删除 `|count|` 个值为 `value` 的元素。
    -   `count = 0`: 删除所有值为 `value` 的元素。

---

#### **4. Set (无序集合) 类型**

-   **简介**: 一个无序且元素唯一的集合，类似于数学中的集合。
-   **应用场景**: 去重、标签系统、好友关系、黑名单、随机抽奖。

| 命令 | 说明 |
| :--- | :--- |
| `SADD key member1 [member2 ...]` | 向集合中添加一个或多个成员。 |
| `SREM key member` | 从集合中移除一个或多个成员。 |
| `SMEMBERS key` | 获取集合中的所有成员。 |
| `SISMEMBER key member` | 判断一个成员是否存在于集合中。 |
| `SCARD key` | 获取集合的成员数量。 |
| `SPOP key [count]` | 随机弹出一个或多个成员（会从集合中删除）。 |
| `SRANDMEMBER key [count]` | 随机获取一个或多个成员（不会删除）。 |
| `SUNION/SINTER/SDIFF` | 执行并集、交集、差集运算。 |

---

#### **5. Sorted Set (ZSet, 有序集合) 类型**

-   **简介**: 类似于 Set，但每个成员都关联一个 `score` (分数)，Redis 会根据分数对成员进行排序。成员唯一，但分数可以重复。
-   **应用场景**: 排行榜、实时热门榜单、带权重的任务队列、延时任务。

| 命令 | 说明 |
| :--- | :--- |
| `ZADD key score member [...]` | 添加或更新一个或多个成员及其分数。 |
| `ZRANGE key start end [WITHSCORES]` | 按排名范围获取成员（`WITHSCORES` 可显示分数）。 |
| `ZRANGEBYSCORE key min max [...]` | 按分数范围获取成员。 |
| `ZREM key member` | 删除一个或多个成员。 |
| `ZCARD key` | 获取有序集合的成员数量。 |
| `ZSCORE key member` | 获取指定成员的分数。 |
| `ZINCRBY key increment member` | 为指定成员的分数增加 `increment`。 |

---

#### **6. 其他高级数据类型 (简介)**

-   **Bitmap (位图)**:
    -   **本质**: 基于 String 类型实现，将字符串看作一个位的数组。
    -   **场景**: 大规模布尔状态统计，如用户签到、活跃用户统计。极省空间。
    -   **核心命令**: `SETBIT`, `GETBIT`, `BITCOUNT`, `BITOP`。

-   **HyperLogLog (基数统计)**:
    -   **本质**: 概率性算法，用于估算一个集合中不重复元素的数量（基数）。
    -   **场景**: 统计海量数据的 UV (独立访客数)、DAU/MAU。占用内存极小 (约 12KB)，但有微小误差。
    -   **核心命令**: `PFADD`, `PFCOUNT`, `PFMERGE`。

---

#### **Key 管理与安全配置命令**

| 命令 | 说明 |
| :--- | :--- |
| `KEYS pattern` | 查找匹配模式的 key (如 `KEYS *`，**生产环境慎用**)。 |
| `EXPIRE key seconds` | 为 key 设置过期时间。 |
| `TTL key` | 查看 key 的剩余过期时间 (-1 表示永不过期, -2 表示已删除)。 |
| `PERSIST key` | 移除 key 的过期时间，使其永久有效。 |
| `EXISTS key` | 判断一个或多个 key 是否存在。 |
| `FLUSHDB` | 清空当前数据库的所有 key (**慎用**)。 |
| `FLUSHALL` | 清空所有数据库的所有 key (**极度慎用**)。 |
| `BGSAVE` / `SAVE` | 执行 RDB 持久化。 |
| `BGREWRITEAOF` | 重写 AOF 文件。 |
| `rename-command CONFIG ""` | 禁用危险命令（如 `CONFIG`），提升安全性。 |