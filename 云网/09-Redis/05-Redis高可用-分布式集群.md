
---

## 1. 从 Sentinel 到 Cluster：为什么需要分布式集群？

Sentinel 架构成功解决了主从模式的**自动故障恢复**问题，但仍存在两大瓶瓶颈：

1.  **单 Master 瓶颈**：所有写操作都集中在单个 Master 节点，当数据量巨大（如百 GB 级）或并发请求极高时，单机的内存、CPU 和网络将成为性能瓶颈。
2.  **高可用依赖**：虽然 Sentinel 是分布式系统，但如果所有 Sentinel 节点同时宕机，主从集群将失去自动故障转移能力。

**Redis Cluster** 是 Redis 官方提供的分布式解决方案，它不仅具备高可用性，还通过数据分片实现了**水平扩展**，专为大规模、高并发场景设计。

| 对比维度 | Sentinel 主从架构 | Redis Cluster 分布式集群 |
| :--- | :--- | :--- |
| **核心目标** | 高可用、自动故障恢复 | 高可用 + **水平扩展** (容量和性能) |
| **数据存储** | 所有数据存储在单个 Master | 数据被**分片 (sharded)** 到多个 Master |
| **架构模型** | 中心化（一个 Master） | **去中心化**（所有节点对等） |
| **故障影响** | Master 宕机，整个集群写服务暂停 | 单个 Master 宕机，仅影响其负责的数据分片 |
| **扩展性** | 只能垂直扩展（提升单机性能） | 可**水平扩展**（增加节点即可增加容量和吞吐量） |

**总结：**
*   **Sentinel**：适用于数据量可控、对高可用要求高的场景。
*   **Cluster**：适用于海量数据、高并发、需要线性扩展能力的场景，如大型社交、电商、游戏平台。

---

## 2. Redis Cluster 核心原理

### 2.1 数据分区规则：哈希槽 (Hash Slot)

Redis Cluster 的核心是**虚拟哈希槽**分区机制。

*   **16384 个哈希槽**：集群将整个键空间划分为 16384 个逻辑槽 (slot)。
*   **CRC16 算法**：每个 key 通过 `CRC16(key) % 16384` 的公式计算出它属于哪个槽。
*   **槽位分配**：集群启动时，这 16384 个槽会被均匀地分配给所有的 Master 节点。例如，3 个 Master 的集群可能如下分配：
    *   Master 1: 槽 0 - 5460
    *   Master 2: 槽 5461 - 10922
    *   Master 3: 槽 10923 - 16383
*   **元数据共享**：每个节点都保存着完整的“槽-节点”映射表，知道哪个槽由哪个节点负责。

### 2.2 客户端路由与重定向

*   **智能客户端**：客户端（如 `redis-cli -c`）会缓存槽位映射表。当执行命令时，它会先在本地计算 key 的槽位，然后直接向负责该槽的节点发送请求。
*   **MOVED 重定向**：如果集群发生节点变更或数据迁移，客户端缓存的映射表可能过时。当它向错误的节点发送请求时，该节点会回复一个 `MOVED <slot> <ip>:<port>` 错误，客户端收到后会更新本地缓存并重定向到正确的节点。

---

## 3. Redis Cluster 分布式集群搭建 (3主3从)

本节将搭建一个由 3 台物理机、6 个 Redis 实例组成的 3 主 3 从集群。

### 3.1 环境规划

采用伪集群方式，在 3 台物理机上各运行 2 个 Redis 实例。**主从节点交叉部署以提高可用性。**

| 节点（实例端口）           | 节点/Master实例  | 节点/Slave实例   | ip              |
| :----------------- | :----------- | :----------- | --------------- |
| `node1`（7001/7002） | `node1` 7001 | `node2` 7004 | 192.168.226.100 |
| `node2`（7003/7004） | `node2` 7003 | `node3` 7006 | 192.168.226.101 |
| `node3`（7005/7006） | `node3` 7005 | `node1` 7002 | 192.168.226.102 |


### 3.2 部署并启动 6 个 Redis 实例

在**每个物理节点**上，重复以下步骤，注意修改对应的端口号和 IP 地址。

1.  **使用脚本安装 Redis**：
    ```bash
    yum install -y wget
    # 拉取脚本部署 redis（注意修改脚本中的端口号为 7001 和 7002）
    wget https://gitee.com/tianhairui/sh/raw/master/Application/redis-fast-preconfig-clustering.sh
    sh redis-fast-preconfig-clustering.sh
    ```
    >该脚本[redis-fast-preconfig-clustering.sh](脚本/redis-fast-preconfig-clustering.sh.md)
    >会创建多个实例目录（如 `/redis/install/7001`, `/redis/install/7002`）。

2.  **修改配置文件**：
    对每个实例（7001 到 7006）的配置文件进行修改。
    ```
    # 过滤掉空行及注释行 
    mv /redis/install/7001/conf/redis7001.conf /redis/install/7001/conf/redis7001.conf.bak 
    grep -Ev '^(#|$)' /redis/install/7001/conf/redis7001.conf.bak > /redis/install/7001/conf/redis7001.conf 
    
    mv /redis/install/7002/conf/redis7002.conf /redis/install/7002/conf/redis7002.conf.bak 
    grep -Ev '^(#|$)' /redis/install/7002/conf/redis7002.conf.bak > /redis/install/7002/conf/redis7002.conf
    ```
    >[7001实例配置文件](分布式集群配置文件/7001实例配置文件.md)、[7002实例配置文件](分布式集群配置文件/7002实例配置文件.md)   
    >[7003实例配置文件](分布式集群配置文件/7003实例配置文件.md)、[7004实例配置文件](分布式集群配置文件/7004实例配置文件.md)   
    >[7005实例配置文件](分布式集群配置文件/7005实例配置文件.md)、[7006实例配置文件](分布式集群配置文件/7006实例配置文件.md)   

3.  **启动所有实例**：
    在每个节点上，为对应的实例启动服务。
    ```bash
    # 在 node1 上
    redis-server /redis/install/7001/conf/redis7001.conf
    redis-server /redis/install/7002/conf/redis7002.conf
  
    # 在 node2 上
    redis-server /redis/install/7003/conf/redis7003.conf
    redis-server /redis/install/7004/conf/redis7004.conf

    # 在 node3 上
    redis-server /redis/install/7005/conf/redis7005.conf
    redis-server /redis/install/7006/conf/redis7006.conf
    ```

### 3.3 创建集群

在**任意一个节点**（如 `node1`）上执行以下命令来初始化集群。

1.  **安装 Ruby 环境**（`redis-cli --cluster` 依赖 Ruby）
    ```bash
    # root 用户安装 ruby
    yum install -y ruby
    ```

2.  **执行集群创建命令**
    **注意**：`redis-cli` 会自动选择前 N 个节点作为 Master。为了实现交叉复制，需要手动指定 Master 和 Slave 的配对，或者按照工具的默认行为来创建。文档中的命令是让工具自动分配。
    ```
    su - redis
    redis-cli -a redhat --cluster create 192.168.226.100:7001 192.168.226.100:7002 192.168.226.101:7003 192.168.226.101:7004 192.168.226.102:7005 192.168.226.102:7006 --cluster-replicas 1

	# 输出记录
	[redis@node1 ~]$ redis-cli -a redhat --cluster create 192.168.226.100:7001 192.168.226.100:7002 192.168.226.101:7003 192.168.226.101:7004 192.168.226.102:7005 192.168.226.102:7006 --cluster-replicas 1
	Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe. 
	>>> Performing hash slots allocation on 6 nodes... 
	Master[0] -> Slots 0 - 5460 
	Master[1] -> Slots 5461 - 10922 
	Master[2] -> Slots 10923 - 16383 
	Adding replica 192.168.226.101:7004 to 192.168.226.100:7001 
	Adding replica 192.168.226.102:7006 to 192.168.226.101:7003 
	Adding replica 192.168.226.100:7002 to 192.168.226.102:7005 
	M: acb1160a6bcc120885b56c99775b0b317789e1dd 192.168.226.100:7001
	   slots:[0-5460] (5461 slots) master 
	S: 2a14387a4a2172f82ab791e70b474b228807753b 192.168.226.100:7002 
	   replicates 8cca51fd4810549374dff70c6f5640e00c32f2f8
	M: e6ce30725eebbb8674425128965b14f36503ebce 192.168.226.101:7003 
	   slots:[5461-10922] (5462 slots) master 
	S: c02249cab4d9cd29c6f7b4d25f4f871532c92744 192.168.226.101:7004 
	   replicates acb1160a6bcc120885b56c99775b0b317789e1dd 
	M: 8cca51fd4810549374dff70c6f5640e00c32f2f8 192.168.226.102:7005 
	   slots:[10923-16383] (5461 slots) master 
	S: 03e2dbdbeea08f33bc09c4b756fa911b9402b709 192.168.226.102:7006 
	   replicates e6ce30725eebbb8674425128965b14f36503ebce 
	Can I set the above configuration? (type 'yes' to accept): yes 
	>>> Nodes configuration updated 
	>>> Assign a different config epoch to each node 
	>>> Sending CLUSTER MEET messages to join the cluster 
	Waiting for the cluster to join
	
	>>> Performing Cluster Check (using node 192.168.226.100:7001) 
	M: acb1160a6bcc120885b56c99775b0b317789e1dd 192.168.226.100:7001 
	   slots:[0-5460] (5461 slots) master 
	   1 additional replica(s) 
	S: 2a14387a4a2172f82ab791e70b474b228807753b 192.168.226.100:7002 
	   slots: (0 slots) slave 
	   replicates 8cca51fd4810549374dff70c6f5640e00c32f2f8 
	M: 8cca51fd4810549374dff70c6f5640e00c32f2f8 192.168.226.102:7005 
	   slots:[10923-16383] (5461 slots) master 
	   1 additional replica(s) 
	S: c02249cab4d9cd29c6f7b4d25f4f871532c92744 192.168.226.101:7004 
	   slots: (0 slots) slave 
	   replicates acb1160a6bcc120885b56c99775b0b317789e1dd 
	M: e6ce30725eebbb8674425128965b14f36503ebce 192.168.226.101:7003 
	   slots:[5461-10922] (5462 slots) master 
	   1 additional replica(s) 
	S: 03e2dbdbeea08f33bc09c4b756fa911b9402b709 192.168.226.102:7006 
	   slots: (0 slots) slave 
	   replicates e6ce30725eebbb8674425128965b14f36503ebce 
	[OK] All nodes agree about slots configuration. 
	>>> Check for open slots... 
	>>> Check slots coverage... 
	[OK] All 16384 slots covered.
	```

### 3.4 登录与测试

1.  **登录集群**
    使用 `-c` 参数以集群模式连接。
    ```bash
    redis-cli -c -h 192.168.226.100 -p 7001 -a redhat
    ```

2.  **测试数据存储与重定向**
    ```redis
    192.168.226.100:7001> set aaa 111
    -> Redirected to slot [10439] located at 192.168.226.101:7003
    OK
    192.168.226.101:7003> get aaa
    "111"
    ```
    可以看到，客户端自动重定向到负责该 key 槽位的节点。

3.  **查看集群状态**
    ```redis
    # 查看集群整体信息
    CLUSTER INFO
    # 查看所有节点状态和槽位分配
    CLUSTER NODES
    ```

---

## 4. 集群在线扩容与缩容（拓展）

Redis Cluster 支持在不停止服务的情况下动态添加或移除节点。

### 4.1 扩容：添加新节点

假设要添加一个新主节点 `7007` 和一个新从节点 `7008`。

1.  **准备并启动新实例**：按照 **3.2** 的步骤，在 `node1` 和 `node3` 上分别创建并启动 `7007` 和 `7008` 实例。

2.  **将新节点加入集群**：
    登录任一现有节点，使用 `CLUSTER MEET` 命令。
    ```redis
    192.168.226.100:7001> CLUSTER MEET 192.168.226.100 7007
    192.168.226.100:7001> CLUSTER MEET 192.168.226.102 7008
    ```
    此时新节点以 Master 身份加入，但没有分配任何槽位。

3.  **分配主从关系**：
    登录到 `7008` 节点，将其设置为 `7007` 的从节点。
    ```redis
    # 首先获取 7007 的 Node ID
    # redis-cli -p 7001 -a redhat CLUSTER NODES | grep 7007
    # 假设 7007 的 ID 为 1373988e...
  
    redis-cli -c -h 192.168.226.102 -p 7008 -a redhat
    192.168.226.102:7008> CLUSTER REPLICATE 1373988e8da47d1325e9e9126d907dda0e8d248e
    192.168.226.102:7008> CLUSTER NODES
    ```

4.  **为新 Master 重新分片 (Reshard)**：
    使用 `reshard` 命令从现有 Master 中迁移一部分槽位给新 Master `7007`。
    ```bash
    redis-cli -a redhat --cluster reshard 192.168.226.100:7001
    ```
    根据交互提示操作：
    *   要移动多少槽位？（例如 `4096`）
    *   接收槽位的节点 ID 是？（输入 `7007` 的 Node ID）
    *   从哪些源节点移动？（输入 `all` 表示从所有 Master 均匀迁移）
    *   输入 `yes` 确认执行。
    *   最后登录节点 `CLUSTER NODES` 查看

### 4.2 缩容：移除节点

> **高危操作提醒**：移除节点前必须先将其负责的槽位迁移出去，否则会导致数据丢失！  
> **<mark>顺序：先reshard -> 再删除从节点 -> 再删除主节点</mark>**    
> **不要直接 kill 7007/7008**，否则可能导致 slot 丢失或集群不完整   
> 如果集群开启密码/ACL，记得在命令中加 `-a redhat` 或 `--user ... --pass ...`   

1.  **迁移槽位**：
    使用 `reshard` 命令，将要移除的 Master（如 `7007`）的所有槽位迁移到其他 Master 节点。
    *   要移动多少槽位？（输入 `7007` 拥有的所有槽位数量）
    *   接收槽位的节点 ID 是？（输入一个或多个其他 Master 的 ID）
    *   源节点 ID 是？（只输入 `7007` 的 Node ID）

2.  **移除从节点**：
    从节点没有槽位，可以直接移除。
    ```bash
    # 假设 7008 的 ID 为 b88d5d...
    redis-cli -a redhat --cluster del-node 192.168.226.100:7001 b88d5d5cfdeedff0d03ca0c74dee2c766d74f55e
    ```

3.  **移除主节点**：
    在确保该节点槽位已全部迁移走后，执行移除命令。
    ```bash
    redis-cli -a redhat --cluster del-node 192.168.226.100:7001 1373988e8da47d1325e9e9126d907dda0e8d248e
    ```
    操作完成后，可以关闭被移除节点的 Redis 进程。