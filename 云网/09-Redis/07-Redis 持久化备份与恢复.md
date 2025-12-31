
---
## Redis 持久化介绍
Redis 是缓存数据库，默认情况下，服务器重启会导致数据丢失。为了解决这个问题，Redis 提供了两种持久化机制，可以将内存中的数据保存到磁盘，以便在重启后恢复。  

*   **RDB (Redis DataBase)**：快照持久化。在指定的时间间隔内，将内存中的数据生成一个二进制的快照文件（默认为 `dump.rdb`）。
*   **AOF (Append Only File)**：日志持久化。将 Redis 执行的每一条写命令追加到文件（默认为 `appendonly.aof`）末尾。重启时，通过重新执行这些命令来恢复数据。

在生产环境中，通常会同时开启 RDB 和 AOF，以兼顾数据安全性和恢复速度。  

### RDB 与 AOF 对比

| 特性     | RDB                   | AOF                   |
| ------ | --------------------- | --------------------- |
| 持久化方式  | 定期生成快照                | 持续记录写命令               |
| 数据恢复速度 | 快，直接加载二进制文件           | 慢，需要重放所有命令            |
| 数据丢失风险 | 较高，可能丢失最近一次快照后的数据     | 较低，取决于 `fsync` 策略     |
| 文件大小   | 小，经过压缩的二进制格式          | 大，记录了所有历史写命令          |
| 性能影响   | 低，`fork` 子进程执行，主进程影响小 | 略高，每次写操作都需记录日志，有I/O开销 |

---

## RDB 持久化

### RDB 配置参数
在 `redis.conf` 文件中进行配置。

*   <mark>save</mark>  
    设置自动触发快照的条件。满足任意一个条件即会执行 `BGSAVE`。
    ```
    # 900秒（15分钟）内至少有1个key被修改
    save 900 1
    # 300秒（5分钟）内至少有10个key被修改
    save 300 10
    # 60秒（1分钟）内至少有10000个key被修改
    save 60 10000
    ```

*   <mark>stop-writes-on-bgsave-error yes</mark>    
    当后台快照（bgsave）失败时，Redis 将禁止所有写操作，直到你解决问题为止。默认为 `yes`，能防止因持久化问题导致的数据不一致。

*   <mark>rdbcompression yes</mark>     
    进行镜像备份时，是否进行压缩，压缩需要消耗一定的 cpu 性能。

*   <mark>rdbchecksum yes</mark>     
    是否在 RDB 文件末尾追加 CRC64 校验和。默认为 `yes`，当 Redis 启动或加载 RDB 文件时，它会重新计算一次校验值，验证文件内容是否被篡改或损坏。      
    开启校验（yes）：确保数据文件的完整性与可靠性。如果校验失败，Redis 会拒绝加载该 RDB 文件，并在日志中报错。    
    关闭校验（no）：加载和保存 RDB 文件速度会稍微快一点（少一次 CRC64 计算），但丧失了检测文件损坏的能力。


*   <mark>dbfilename dump.rdb</mark>    
    RDB 快照文件的名称。

*   <mark>dir /var/lib/redis</mark>    
    存放快照的目录

### RDB 重要命令
*   `BGSAVE`  
    **异步执行快照**。**一次性快照任务**，执行后生成一份 RDB 快照文件（dump.rdb），不会持续自动执行。Redis 会 `fork` 一个子进程来创建 RDB 文件，主进程可以继续处理客户端请求，基本无阻塞。自动触发的 `save` 配置以及手动执行 `BGSAVE` 都是通过此方式。  
    `BGSAVE` 执行时 Redis 做的事情：
    1. 发现`save`规则满足
    2. `fork`子进程
    3. 子进程开始写 RDB 文件
    4. 不阻塞主进程
    5. 新 RDB 完成后替换旧的
    6. 主进程继续正常运行
*   `SAVE`  
    **同步执行快照**。**一次性快照任务**，它和 bgsave 不同点在于：save 是实时同步阻塞备份（强一致性），会阻塞整个 Redis，**不建议在生产环境中使用**。

---

## AOF 持久化

### AOF 配置参数
*   <mark>appendonly no</mark>  
    是否开启 AOF，默认为 `no` (关闭)。生产环境强烈建议设为 `yes`。

*   <mark>appendfilename "appendonly.aof"</mark>  
    AOF 日志文件的名称。

*   <mark>appenddirname "appendonlydir"</mark>  
    AOF 文件存放的目录名（Redis 7+ 新增）。

*   <mark>appendfsync everysec</mark>  
    AOF 文件同步到磁盘的策略，直接影响数据安全性和性能。

| 参数       | 机制                 | 性能  | 丢失数据量     |
| -------- | ------------------ | --- | --------- |
| always   | 每次写操作立即 fsync      | 最差  | 0         |
| everysec | 每秒 fsync 一次        | 中   | 最多丢 1 秒   |
| no       | 不主动 fsync（由系统自己决定） | 最好  | 不可控（可能很多） |


*   <mark>no-appendfsync-on-rewrite no</mark>  
    在进行 AOF 重写时，是否暂停 `fsync` 操作。  
    默认为 no，表示重写期间依然执行 `fsync`，数据最安全但 I/O 压力较大。  
    **生产环境建议：保持 no**

*   <mark>auto-aof-rewrite-percentage 100</mark>  
    自动 AOF 重写阈值（百分比）  
    自动 rewrite 的触发条件：当前AOF大小 >= 上次rewrite后AOF大小 × (1 + 100%)  
    即：上次重写后 100MB，达到 200MB 时触发 rewrite

*   <mark>auto-aof-rewrite-min-size 64mb</mark>  
    自动重写最小阈值  
    只有当 AOF 文件 >= 64MB 才会考虑自动 rewrite。  
    也就是两条件需同时满足：  
    **文件大小 >= auto-aof-rewrite-min-size**  
    **增长比例 >= auto-aof-rewrite-percentage**

*   <mark>aof-load-truncated yes</mark>  
    当 AOF 结尾损坏时，是否自动忽略损坏部分并继续启动  
    yes（默认）：跳过损坏尾部，继续加载  
    no：加载失败，Redis 启动报错  
    Redis 崩溃后 AOF 末尾常会损坏，所以生产一般设为 yes。

*   <mark>aof-use-rdb-preamble yes</mark>  
    AOF 重写后的文件是否使用 “RDB 作为前缀 + AOF 作为增量”  
    AOF 文件结构如下：\[RDB snapshot] + \[AOF 后续增量指令]  
    好处：加载速度比纯 AOF 更快（因为 RDB 更紧凑）、文件更小、更安全（崩溃更容易恢复）
*   <mark>aof-timestamp-enabled no</mark>  
	是否在 AOF 中记录命令执行时间戳  
	no（默认）：不记录  
	yes：AOF 文件每条指令带时间戳（Redis 6.2+）  
	使用场景：调试、回放、时间审计（很少用）。大部分公司都设为 no

### AOF 重写 (Rewrite)
#### 为什么需要重写？
随着时间推移，AOF 文件会记录大量冗余命令（如对同一个 key 的多次修改），导致文件体积膨胀，拖慢 Redis 重启速度。  
**AOF 重写** 并不是读取和修改旧的 AOF 文件，而是根据当前内存中的数据状态，生成一套能恢复当前数据集的、最精简的命令，并写入一个新的 AOF 文件中，最后替换掉旧文件。  
>⚠️如果面试中提到这个问题，可以按照下面来回答：  
>AOF 重写是 Redis 为了控制 AOF 文件大小而提供的“瘦身机制”。  
>它不会对旧 AOF 做编辑，而是根据当前内存中的数据重新生成一个最小化、无冗余的新 AOF 文件。  
>重写过程由子进程执行，不影响主线程服务。主线程新写入的命令会保存到重写缓冲区，最终和新 AOF 合并，再原子替换旧 AOF 文件。  
>重写可以显著减小 AOF 文件大小、加快 Redis 重启速度，是生产环境必备机制。


#### 重写流程

>BGREWRITEAOF：执行命令后，后台异步执行 AOF 重写（Append-Only File Rewrite）  

1.  **触发重写**：  
	当执行`BGREWRITEAOF`命令  
	主进程做两件事：  
	* 检查是否已经有重写在进行（避免重复）
	* fork 一个子进程来执行重写
2.  **子进程写入**：  
	子进程根据当前内存数据，将最简命令写入一个新的临时 AOF 文件（temp-rewriteaof.aof.temp）。此过程不影响主进程。
3.  **主进程缓冲**（重写期间产生的新写操作怎么办？）：  
	主进程继续处理写命令。但子进程读的是 重写开始那一刻的快照，不会包含正在发生的新写入。  
	在子进程重写期间，主进程接收到的新写命令会同时写入**旧的 AOF 文件**和**AOF 重写缓冲区**(rewrite buffer)。
4.  **合并与替换**（子进程重写完成后）：  
	子进程退出，把新的 AOF 文件写完。主进程负责最后一步： 把 rewrite buffer 中的增量命令追加到新 AOF 中，并且用新 AOF 原子替换旧的 AOF（rename，保证不会中途损坏）。整个切换过程几乎瞬间完成。

#### 重写触发方式
*   **手动触发**：执行 `BGREWRITEAOF` 命令。
*   **自动触发（推荐）**：当 AOF 文件大小同时满足 `auto-aof-rewrite-min-size` 和 `auto-aof-rewrite-percentage` 两个配置条件时，Redis 会自动执行 `BGREWRITEAOF`。

---

## Redis 备份与恢复

### 备份流程
1.  **开启持久化方案**
	```
	save 900 1
	save 300 10
	save 60 10000
	appendonly yes
	dir /redis/install/data
	
	[redis@henry ~]$ ls /redis/install/data
	appendonlydir  dump.rdb
	```

2.  **查看持久化配置**
	```
	CONFIG GET save
	CONFIG GET appendonly
	CONFIG GET appenddirname
	CONFIG GET appendfilename
	CONFIG GET dir
	```
3.  **检查持久化状态**  
    备份前，通过 `INFO persistence` 命令检查持久化状态，确保 `rdb_last_bgsave_status` 和 `aof_last_bgrewrite_status` 均为 `ok`。

4.  **生成最新快照**  
    为确保备份数据的完整性，手动触发一次持久化。推荐使用 `BGSAVE` 进行异步备份。

5.  **复制持久化文件**  
    找到持久化文件目录（通过 `CONFIG GET dir` 和 `CONFIG GET appenddirname` 等查看），并将 RDB 文件和 AOF 目录完整复制到安全的备份位置。建议使用带时间戳的命名方式保存历史版本。
    ```bash
    # 查看目录位置
    CONFIG GET dir  # 查 RDB
    CONFIG GET appenddirname # 查 AOF
    
    # 查看文件名
    CONFIG GET dbfilename # 查 RDB
    CONFIG GET appendfilename # 查 AOF
    
    # 备份 RDB
    cp /path/to/redis/data/dump.rdb /backup/redis/`date +%F-%H%M%S`-dump.rdb
  
    # 备份 AOF，建议复制整个目录
    cp -r /path/to/redis/data/appendonlydir /backup/redis/aof-`date +%F-%H%M%S`
    ```
    **注意**：备份文件时要注意文件权限问题。

6.  **校验备份文件**
    使用 Redis 提供的工具检查备份文件的完整性。
    ```bash
    # 校验 RDB 文件
    redis-check-rdb /backup/redis/xxxx-dump.rdb
  
    # 校验并尝试修复 AOF 文件
    redis-check-aof --fix /backup/redis/aof-xxxx/appendonly.aof.*.incr.aof
    ```

7.  **异地容灾**
    将备份文件上传到云存储或另一台物理服务器，以防本地存储损坏。

### 恢复流程
1.  **恢复优先级**  
    当 Redis 启动时，如果数据目录下同时存在 `dump.rdb` 和 `appendonly.aof`，**Redis 会优先加载 AOF 文件**，因为 AOF 含有更详细的命令日志，数据最完整、丢失最少。

2.  **恢复步骤**   
    a. 关闭 Redis 服务。   
    b. 清空 Redis 数据目录。  
    c. 将备份的 RDB 文件和/或 AOF 目录复制回 Redis 的数据目录。确保文件名和目录名与 `redis.conf` 中的配置一致。  
    d. 确保文件和目录的属主和权限正确（通常为 `redis:redis`）。  
    e. 启动 Redis 服务，它会自动加载持久化文件进行恢复。  
    f. 连接客户端，验证数据是否恢复成功。  

3.  **强制使用 RDB 恢复**  
    如果希望使用 RDB 文件进行恢复（例如，AOF 文件损坏），可以先临时在 `redis.conf` 中设置 `appendonly no`，然后启动 Redis。恢复完成后，再按正确步骤重新开启 AOF。

### 中途开启 AOF 的注意事项
如果 Redis 实例已在仅使用 RDB 的模式下运行了一段时间，此时直接在配置文件中设置 `appendonly yes` 并重启，会导致数据丢失！因为 Redis 会优先加载一个新创建的、空的 AOF 文件。

**正确步骤如下：**  
1.  在 `redis.conf` 中设置 `appendonly yes`，或者通过 `CONFIG SET appendonly yes` 动态开启。**此时不要重启 Redis**。
2.  立即执行 `BGREWRITEAOF` 命令。该命令会读取当前内存中的数据（这些数据是从 RDB 加载的），并生成一个包含所有数据的完整 AOF 文件。
3.  等待重写完成后，AOF 持久化机制便已平滑启用。此后重启 Redis 即可安全地从 AOF 文件恢复数据。

**最佳实践**：在初始部署 Redis 时，就同时开启 RDB 和 AOF。

### 总结
理论上只要备份好RDB / AOF 文件，Redis 就能恢复数据。
但生产环境远不止复制文件这么简单，还必须考虑：

*   恢复时 Redis 必须停机。
*   AOF 必须备份整个目录（Redis 7+）。
*   RDB/AOF 有加载优先级，不能混用。
*   `manifest` 文件不能缺失（必须备份 `appendonlydir` 整个目录，且必须包含 manifest 清单文件，否则 Redis 无法恢复 AOF）。
*   文件权限、版本兼容要正常（跨大版本可能无法恢复）。
*   需检查 AOF 尾部是否有损坏。
*   恢复后必须重启 Redis 才能加载数据。

只要这些细节没处理好，恢复就不稳定甚至失败。