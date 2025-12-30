
---

## 1. Redis 主从复制 (Replication)

主从复制是 Redis 高可用的基础，主要用于**读写分离**和**数据备份**，解决的是**性能**和**读扩展**问题。

### 核心原理

1.  **单向同步**：数据同步是单向的，只能从主节点 (Master) 到从节点 (Slave/Replica)。
2.  **读写分离**：
    *   **写操作**：只在 Master 节点上进行。
    *   **读操作**：可以在 Master 和所有 Slave 节点上进行，通过增加 Slave 节点可以线性提升读性能。
3.  **数据同步过程**：
    *   **全量同步 (Full Synchronization)**：
        *   当一个 Slave 首次连接 Master 时触发。
        *   Master 执行 `BGSAVE` 创建 RDB 快照文件。
        *   Master 将 RDB 文件发送给 Slave，Slave 清空自身数据后加载 RDB 文件。
        *   Master 将 RDB 生成期间的增量写命令发送给 Slave。
    *   **增量同步 (Partial Synchronization)**：
        *   当 Slave 因网络等原因短暂断线后重连时触发。
        *   Slave 会携带 `replid` (复制ID) 和 `offset` (偏移量) 请求同步。
        *   Master 只需将 Slave 断线期间的增量写命令发送过去即可，避免了全量同步的开销。

### 优点

*   **读写分离**：有效分担 Master 的读压力，提高整体吞吐量。
*   **高可用性基础**：为后续的哨兵和集群模式提供数据备份基础。
*   **读扩展性**：可以方便地增加 Slave 节点来扩展读能力。

### 致命缺陷

*   **Master 单点故障 (SPOF)**：
    *   一旦 Master 宕机，整个集群将**无法执行任何写操作**，业务中断。
    *   故障恢复完全依赖**人工操作**，过程繁琐且耗时：
        1.  手动将一个 Slave 提升为新的 Master (`SLAVEOF NO ONE`)。
        2.  修改所有客户端应用的连接地址，指向新 Master。
        3.  让其他 Slave 节点跟随新的 Master。
*   **故障恢复混乱**：
    *   缺乏自动故障检测和切换机制。
    *   人工操作容易出错，可能导致**数据不一致**或**脑裂**（出现多个 Master）。

---

## 2. Redis 哨兵 (Sentinel)

Sentinel 是一个独立的进程，用于监控 Redis 主从集群，解决了主从复制模式下的**自动故障恢复**问题，实现了真正的高可用性。

### 核心目标

*   **监控 (Monitoring)**：持续监控 Master 和 Slave 节点是否正常工作。
*   **通知 (Notification)**：当被监控的节点出现问题时，通过 API 通知管理员或其他应用程序。
*   **自动故障转移 (Automatic Failover)**：当 Master 宕机时，能自动从 Slave 节点中选举出一个新的 Master，并让其他 Slave 跟随新主，保证服务不中断。

### 监控与故障判断机制

1.  **心跳检测 (`PING`)**：
    *   每个 Sentinel 每秒向所有 Redis 节点（Master, Slave）和其他 Sentinel 发送 `PING` 命令。
    *   若在 `down-after-milliseconds` 配置的时间内未收到响应，该 Sentinel 会将目标节点标记为**主观下线 (SDOWN - Subjectively Down)**。

2.  **两段式故障确认 (SDOWN -> ODOWN)**：
    *   **主观下线 (SDOWN)**：单个 Sentinel 认为节点不可用。这只是一个“疑似”状态，不会立即触发故障转移。
    *   **客观下线 (ODOWN - Objectively Down)**：当足够数量的 Sentinel (由 `quorum` 参数配置) 都认为 Master 已 SDOWN 时，Master 才被最终确认为**客观下线**。**只有 ODOWN 才会触发故障转移**。

### Leader 选举与故障转移 (Failover) 流程

当 Master 被判定为 ODOWN 后，Sentinel 集群会执行以下步骤：

1.  **选举 Sentinel Leader**：
    *   所有 Sentinel 节点会发起投票，选举一个 Leader 来主导故障转移。
    *   选举采用类似 Raft 算法的机制（随机延迟 + 先到先得），保证最终只有一个 Leader 胜出。
2.  **选举新 Master**：
    *   Leader Sentinel 从所有 Slave 节点中，根据**延迟低、数据新、优先级高**等规则，选举出一个最合适的 Slave 作为新的 Master。
3.  **执行切换**：
    *   Leader 对选出的新 Master 执行 `SLAVEOF NO ONE` 命令，使其成为主节点。
    *   Leader 让其他 Slave 节点执行 `SLAVEOF new_master_ip new_master_port`，跟随新的 Master。
    *   更新内部配置，并将新 Master 的地址通知给客户端。

### 核心配置参数解析 (`sentinel.conf`)

| 参数 | 说明 |
| :--- | :--- |
| `sentinel monitor <master-name> <ip> <port> <quorum>` | 监控名为 `master-name` 的主节点，`quorum` 是判断 ODOWN 所需的最少 Sentinel 票数。 |
| `sentinel down-after-milliseconds <master-name> <ms>` | 判定节点 SDOWN 的超时时间（毫秒）。 |
| `sentinel auth-pass <master-name> <password>` | 如果 Redis 集群设置了密码，需要配置此项。 |
| `sentinel parallel-syncs <master-name> <num>` | 在故障转移后，一次性可以有多少个 Slave 同时对新 Master 进行同步。 |
| `sentinel failover-timeout <master-name> <ms>` | 故障转移的超时时间。 |

---

## 3. 部署实践：1主2从3哨兵

本节将详细介绍如何搭建一个包含1个主节点、2个从节点和3个哨兵进程的高可用 Redis 集群。

### 3.1 环境规划

| 节点角色                           | IP 地址 (示例)           | DNS (示例)        |
| :----------------------------- | :------------------- | :-------------- |
| **主 Master** (同时运行 Sentinel 1) | `192.168.226.134/24` | `192.168.226.2` |
| **从 Slave1** (同时运行 Sentinel 2) | `192.168.226.135/24` | `192.168.226.2` |
| **从 Slave2** (同时运行 Sentinel 3) | `192.168.226.136/24` | `192.168.226.2` |

### 3.2 在所有节点上安装 Redis

在三台服务器（Master, Slave1, Slave2）上都执行以下脚本来快速安装 Redis。

```bash
# 安装 wget 并下载安装脚本
yum install -y wget
wget -P /root/ https://gitee.com/tianhairui/sh/raw/master/Application/redis-fast-preconfig.sh

# 赋予执行权限并运行
chmod +x /root/redis-fast-preconfig.sh
./redis-fast-preconfig.sh
```
>[redis-fast-preconfig.sh](脚本/redis-fast-preconfig.sh.md)

### 3.3 配置 Redis 节点

**1. 修改 Master 节点配置文件 (`192.168.226.134`)**

```bash
# 清理并备份原始配置文件
mv /redis/install/conf/redis.conf /redis/install/conf/redis.conf.bak 
grep -Ev '^\s*$|^\s*#' /redis/install/conf/redis.conf.bak > /redis/install/conf/redis.conf

```

```ini
# 使用 vim 或其他编辑器修改 /redis/install/conf/redis.conf，确保以下参数正确
# (注意：IP地址应替换为当前节点的实际IP)
bind 192.168.226.134 -::1
protected-mode no
timeout 300
daemonize yes
pidfile /redis/install/conf/redis_6379.pid
logfile "/redis/install/log/redis.log"
dir /redis/install/data
requirepass redhat
masterauth redhat
```

**2. 修改 Slave 1 节点配置文件 (`192.168.226.135`)**

> **注意**: 从 Redis 5.0 开始，`SLAVEOF` 命令被 `REPLICAOF` 替代，功能相同。

```bash
# 清理并备份原始配置文件
mv /redis/install/conf/redis.conf /redis/install/conf/redis.conf.bak 
grep -Ev '^\s*$|^\s*#' /redis/install/conf/redis.conf.bak > /redis/install/conf/redis.conf

```

```ini
# 修改 /redis/install/conf/redis.conf，确保以下参数正确
bind 192.168.226.135 -::1
timeout 300
daemonize yes
pidfile /redis/install/conf/redis_6379.pid
logfile "/redis/install/log/redis.log"
dir /redis/install/data
requirepass redhat
masterauth redhat
replicaof 192.168.226.134 6379
```

**3. 修改 Slave 2 节点配置文件 (`192.168.226.136`)**

配置与 Slave 1 类似，只需修改 `bind` 的 IP 地址。

```ini
bind 192.168.226.136 -::1
timeout 300
daemonize yes
pidfile /redis/install/conf/redis_6379.pid
logfile "/redis/install/log/redis.log"
dir /redis/install/data
requirepass redhat
masterauth redhat
replicaof 192.168.226.134 6379
```

### 3.4 启动并验证主从复制

1.  **在三个节点上分别启动 Redis 服务**
    切换到 `redis` 用户执行：
    ```bash
    # Master 节点
    [redis@master ~]$ redis-server /redis/install/conf/redis.conf
    # Slave 1 节点
    [redis@slave01 ~]$ redis-server /redis/install/conf/redis.conf
    # Slave 2 节点
    [redis@slave02 ~]$ redis-server /redis/install/conf/redis.conf
    ```

2.  **验证主从状态**
    登录 Master 节点，执行 `info replication` 查看从节点连接状态。
    ```bash
    [redis@master ~]$ redis-cli -h 192.168.226.134 -p 6379 -a redhat
    192.168.226.134:6379> role
    192.168.226.134:6379> info replication
    ```
    应能看到 `role:master` 以及 `connected_slaves:2` 和两个 slave 的信息。

3.  **测试读写分离**
    *   **主库写**：`192.168.226.134:6379> set mykey "hello"` (成功)
    *   **从库读**：`192.168.226.135:6379> get mykey` (应返回 "hello")
    *   **从库写**：`192.168.226.135:6379> set anotherkey "world"` (失败，返回 `READONLY` 错误)

### 3.5 配置并启动 Sentinel

在三个节点上分别配置 `sentinel.conf`，内容**完全一致**。

**1. 准备 Sentinel 配置文件**
在每个节点上执行：
```bash
# 清理并备份原始配置文件
grep -Ev '^\s*$|^\s*#' /redis/install/redis-8.2.0/sentinel.conf > /redis/install/conf/sentinel.conf


```

**2. 核心配置内容 (`sentinel.conf`)**
```ini
# 修改 /redis/install/conf/sentinel.conf

# 绑定当前节点的 IP，每个节点此项不同
# Master 节点: bind 192.168.226.134
# Slave 1 节点: bind 192.168.226.135
# Slave 2 节点: bind 192.168.226.136

port 26379
daemonize yes
pidfile /redis/install/conf/redis-sentinel.pid
logfile "/redis/install/log/sentinel.log"
dir /redis/install/data

# 以下配置在所有 Sentinel 节点上必须完全相同
sentinel monitor mymaster 192.168.226.134 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel auth-pass mymaster redhat
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```
*   `sentinel monitor mymaster 192.168.226.134 6379 2`：监控名为 `mymaster` 的主节点，至少需要 `2` 个 Sentinel 同意才能判定其客观下线。

**3. 在所有节点启动 Sentinel 进程**
```bash
redis-sentinel /redis/install/conf/sentinel.conf
```
启动后，可以通过 `ps -ef | grep redis-sentinel` 或登录 Sentinel 端口 `26379` 查看信息。

### 3.6 高可用集群验证

1.  **模拟 Master 宕机**
    在 Master 节点 (`192.168.226.134`) 上关闭 Redis 服务。
    ```bash
    redis-cli -h 192.168.226.134 -a redhat shutdown
    ```

2.  **观察 Sentinel 日志**
    在任一 Sentinel 节点的日志文件 (`/redis/install/log/sentinel.log`) 中，可以看到故障转移的全过程：
    *   `+sdown master mymaster ...` (主观下线)
    *   `+vote-for-leader ...` (开始选举 Leader)
    *   `+odown master mymaster ...` (客观下线)
    *   `+switch-master mymaster <old_ip> <new_ip>` (切换主节点)
    *   `+slave slave ... @ mymaster <new_ip>` (其他从节点跟随新主)

3.  **验证新主**
    故障转移完成后，可以登录任一 Sentinel (`redis-cli -p 26379`)，执行 `sentinel get-master-addr-by-name mymaster` 来获取当前主节点的地址，会发现地址已经自动切换为某个原从节点的 IP。同时，登录新的主节点可以进行写操作。