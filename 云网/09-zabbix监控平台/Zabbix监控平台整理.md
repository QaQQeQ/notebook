
***
## 01 - Zabbix Server 核心服务器部署
### 一、 Zabbix 功能简介
Zabbix 是一个开源的企业级监控解决方案，用于监控网络、服务器、应用程序和服务的可用性与性能。其核心功能包括：
*   **数据采集**：支持通过 Agent、SNMP、JMX、IPMI 等多种方式采集监控指标。
*   **问题检测**：强大的触发器引擎，可灵活定义问题的阈值和条件。
*   **告警通知**：支持通过邮件、短信、钉钉、企业微信等多种媒介发送告警通知。
*   **可视化**：提供图形、仪表盘、网络拓扑图等丰富的可视化功能。
*   **自动化**：支持自动发现、自动注册和自动化任务，简化运维工作。

### 二、 部署 Zabbix Server 6.4 核心服务器
#### 1. 环境说明
*   **操作系统**: Rocky Linux 9 
>注：MySQL数据库、web前端、zabbix服务器均在不同节点，配置的ip需根据实际情况替换

#### 2. 安装部署: MySQL数据库
首先，安装数据库服务器，创建 Zabbix 数据库、用户并授权。
```bash
dnf install -y mysql-server mysql
systemctl enable mysqld
systemctl restart mysqld
mysqladmin -uroot password 'a123456!'
mysql -uroot -p'a123456!' <<EOF
create database if not exists zabbix character set utf8mb4 collate utf8mb4_bin;
create user if not exists root@'%' identified by 'a123456!';
grant all privileges on *.* to root@'%';
create user if not exists zabbix@localhost identified by 'a123456!';
grant all privileges on zabbix.* to zabbix@localhost;
create user if not exists zabbix@'%' identified by 'a123456!';
grant all privileges on zabbix.* to zabbix@'%';
set global log_bin_trust_function_creators = 1; ## 在<zabbix用户数据库>导入数据之前,允许:<非特权用户>创建<存储函数>
EOF
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload

```

#### 3. 安装部署: Nginx Web前端
**使用国内源**   
```
vi /etc/yum.repos.d/ustc_epel_9.repo  ## 这是一个国内的 EPEL YUM 镜像源

[epel]
name=Extra Packages for Enterprise Linux $releasever - $basearch
baseurl=https://mirrors.ustc.edu.cn/epel/$releasever/Everything/$basearch
failovermethod=priority
enabled=1
gpgcheck=0
excludepkgs=zabbix*

```
安装 Zabbix 官方仓库，然后安装 Nginx 和 Zabbix 前端相关包。
```bash
# 安装 Zabbix 仓库
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/9/x86_64/zabbix-release-6.0-4.el9.noarch.rpm
yum clean all
yum makecache

# 安装 Zabbix 前端和 Nginx
dnf -y install zabbix-web-mysql zabbix-nginx-conf 
```
修改 Nginx 配置文件 `/etc/nginx/conf.d/zabbix.conf`，例如监听端口或 server_name。
```bash
......
listen 80;
server_name zabbix.test.cn;
root /usr/share/zabbix;
index index.php;
......
```
配置nginx web前端的php配置文件，设置正确的时区
```
vi /etc/php-fpm.d/zabbix.conf
[zabbix]
user = nginx
group = nginx
listen = /run/php-fpm/zabbix.sock
listen.acl_users = apache,nginx
listen.allowed_clients = 127.0.0.1
... ...
php_value[date.timezone] = Asia/Shanghai

```
授予 nginx 用户针对`/etc/zabbix/web`目录的写入权限，然后启动nginx服务进程、php-fpm服务进程
```bash
setfacl -m u:nginx:rwx /etc/zabbix/web/

systemctl enable nginx php-fpm
systemctl restart nginx php-fpm
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --reload
## 禁用：SElinux
if ! [[ $(getenforce) == "Disabled" ]]; then
  sed -r -i 's/^\s*SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
  reboot
fi

```

#### 4. 安装部署: Zabbix Server 核心服务器
**同上使用国内源，以及安装zabbix库**  
安装 Zabbix Server 软件包，并配置其数据库连接。
```bash
# 安装
dnf install -y zabbix-server-mysql zabbix-sql-scripts zabbix-selinux-policy zabbix-agent


# 针对MySQL数据库，导入初始架构和数据
dnf install -y mysql
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
      mysql --default-character-set=utf8mb4 -uzabbix -p'a123456!' -h 192.168.168.16 zabbix


# 导入数据库架构后，针对数据库禁用 log_bin_trust_function_creators 选项
mysql -uroot -p'a123456!' -h 192.168.168.16 <<EOF
set global log_bin_trust_function_creators = 0; ## 在<zabbix 用户数据库>导入数据之后，禁止：<非特权用户>创建<存储函数>
use zabbix;
show tables;
EOF


# 配置数据库连接参数
vi /etc/zabbix/zabbix_server.conf
... ...
DBHost=192.168.168.16
DBName=zabbix
DBUser=zabbix
DBPassword=a123456!

DBPort=3306
... ...
```
查看连接配置(示例)  
`cat /etc/zabbix/zabbix_server.conf | grep -vE "^\s*(#|$)"`  
![](assets/Zabbix监控平台整理/连接配置示例.png)   

启动各个服务进程
```bash
systemctl enable zabbix-server zabbix-agent
systemctl restart zabbix-server zabbix-agent
firewall-cmd --permanent --zone=public --add-port=10050/tcp  ## 防火墙放行：<Zabbix Agent 客户端代理>的<侦听端口>
firewall-cmd --permanent --zone=public --add-port=10051/tcp  ## 防火墙放行：<Zabbix Server 核心服务>的<侦听端口>
firewall-cmd --reload
## 禁用：SElinux
if ! [[ $(getenforce) == "Disabled" ]]; then
  sed -r -i 's/^\s*SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
  reboot
fi

```

#### 5. 执行: Web安装向导
访问 `http://<your_server_ip>/`或访问`http://zabbix.test.cn`  
>访问域名需要在主机上配置hosts或dns解析

按照 Web 页面的引导完成安装：
1.  **Welcome**: 欢迎页面，点击 Next。
2.  **Check of pre-requisites**: 检查 PHP 环境依赖，确保全部为 OK。
3.  **Configure DB connection**: 填写之前创建的数据库信息（用户 zabbix，密码 password）。
4.  **Zabbix server details**: 填写 Zabbix Server 的主机和端口（默认端口即可）。
5.  **Summary**: 确认配置信息。
6.  **Install**: 完成安装，下载配置文件并登录（默认用户 `Admin`，密码 `zabbix`）。  

遇到的错误：    
![](assets/Zabbix监控平台整理/错误.png)    
解决方式：
```
## 在<Web 安装向导>执行完成之后，需编辑：<Nginx Web 前端>的</etc/zabbix/web/zabbix.conf.php>
## 明确指定：$ZBX_SERVER = '192.168.168.18';  ## 这是: Zabbix Server 的 IP 地址
## $ZBX_SERVER_PORT = '10051';  ## 这是: Zabbix Server 的侦听端口
## 从而访问：分离式部署的<Zabbix Server 核心服务节点>
## 否则：将会默认访问<localhost>
##
vi /etc/zabbix/web/zabbix.conf.php
......
$ZBX_SERVER = '192.168.168.18';
$ZBX_SERVER_PORT = '10051';
$ZBX_SERVER_NAME = 'zabbix_server';
......

# 最后刷新页面即可
```

#### 6. 查看日志: zabbix-server.log 日志
通过查看日志文件可以排查启动或运行中的问题。
```bash
tail -f /var/log/zabbix/zabbix_server.log
```

## 02 - Zabbix Agent 客户端代理部署
### 一、 部署: Zabbix Agent 客户端代理
#### 1. 环境说明
在需要被监控的客户端主机上执行以下操作。

#### 2. 安装配置: Zabbix Agent 客户端代理
安装 Zabbix Agent 并配置其连接到 Zabbix Server。
```bash
# 在客户端主机安装 Zabbix 仓库
rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/9/x86_64/zabbix-release-6.4-1.el9.noarch.rpm
dnf clean all

# 安装 Agent
dnf install zabbix-agent -y

# 编辑配置文件 /etc/zabbix/zabbix_agentd.conf
# Server: Zabbix Server 的 IP 地址 (用于被动模式)
Server=192.168.1.100
# ServerActive: Zabbix Server 的 IP 地址 (用于主动模式)
ServerActive=192.168.1.100
# Hostname: 客户端的唯一标识，必须与 Web 界面中配置的主机名一致
Hostname=My-Linux-Host

# 启动并设置开机自启
systemctl restart zabbix-agent
systemctl enable zabbix-agent
```

#### 3. 查看日志: Zabbix Agent日志
检查 Agent 是否正常启动并与 Server 通信。
```bash
tail -f /var/log/zabbix/zabbix_agentd.log
```

### 二、 了解: 被动模式、主动模式
#### 1. 被动模式 (Passive Mode)
*   **工作方式**: Zabbix Server 定期向 Agent 的 10050 端口发起连接，请求监控项数据，Agent 收到请求后返回数据。
*   **特点**: 由 Server 主导，配置简单。适用于 Server 能直接访问 Agent 的网络环境。

#### 2. 主动模式 (Active Mode)
*   **工作方式**: Zabbix Agent 首先向 Server 的 10051 端口请求需要监控的指标列表，然后 Agent 主动、定期地将采集到的数据发送给 Server。
*   **特点**: 由 Agent 主导，适用于 Agent 在防火墙或 NAT 之后，Server 无法直接访问的场景。性能开销比被动模式略低。

## 03 - Zabbix 整体架构
### 一、 了解 Zabbix 程序组件
1.  **zabbix_server**: 核心守护进程，负责数据采集、触发器计算、告警发送等。
2.  **zabbix_agentd**: 部署在被监控对象上的守护进程，负责采集本地系统指标。
3.  **zabbix_proxy**: 可选组件，代表 Server 采集数据，分担 Server 压力，简化分布式监控部署。
4.  **zabbix_get**: 命令行工具，用于从 Server 或 Proxy 手动向 Agent 请求数据，常用于测试和排错。

### 二、 了解 Zabbix 通讯机制
1.  **Zabbix的两种<监视模式>**: 即主动模式和被动模式，决定了数据流的发起方。
2.  **Zabbix组件间<通讯机制>**:
    *   **Server -> Agent (被动)**: TCP 端口 10050
    *   **Agent -> Server (主动)**: TCP 端口 10051
    *   **Server <-> Proxy**: Proxy 主动连接 Server 的 10051 端口
    *   **Frontend (Web UI) -> Server**: 通过 Zabbix API 进行通信
    *   **Server/Proxy -> 数据库**: 存储配置信息和历史数据

## 04 - Zabbix 监控对象管理
### 一、 实验环境
本节内容均在 Zabbix Web 前端界面进行操作。

### 二、 主机群组 (Host Groups)
1.  **功能**: 用于对主机进行逻辑分组，如 "Linux Servers", "Web Servers", "Network Devices"。
2.  **要素**: 仅需一个唯一的群组名称。
3.  **创建**: 在 `Configuration` -> `Host groups` 菜单中创建。主机群组是权限管理的基本单位。

### 三、 主机 (Hosts)
1.  **功能**: 代表一个被监控的物理或虚拟设备。
2.  **要素**:
    *   **Host name**: 必须唯一，且与 Agent 配置文件中的 `Hostname` 一致。
    *   **Interfaces**: 定义如何连接到设备（Agent, SNMP, JMX, IPMI），需指定 IP 或 DNS。
    *   **Templates**: 链接的监控模板，决定了该主机监控哪些内容。
    *   **Host groups**: 主机所属的群组。
3.  **创建**: 在 `Configuration` -> `Hosts` 菜单中创建。

### 四、 Discovery自动发现与监控
#### 1. 创建: <Discovery发现规则>
*   **步骤说明**: 在 `Configuration` -> `Discovery` 中创建规则，定义 IP 范围和检查方式。
*   **解析: 定义栏**:
    *   **IP range**: 要扫描的 IP 地址范围，如 `192.168.1.1-254`。
    *   **Checks**: 检查条件，如 Zabbix Agent 是否响应、SNMP 服务是否可用、特定端口是否开放等。

#### 2. 创建: <Discovery发现动作>
*   **步骤说明**: 在 `Configuration` -> `Actions` -> `Discovery actions` 中创建动作，响应发现事件。
*   **解析: 定义栏**:
    *   **Conditions**: 触发动作的条件，如发现的服务类型、主机 IP、收到的值等。
    *   **Operations**: 执行的操作，如添加主机、将主机加入群组、链接模板、发送通知等。

### 五、 Autoregistration自动注册与监控
Agent 主动向 Server 注册，常用于云环境或自动扩缩容场景。

#### 1. 配置: <Zabbix Agent客户端>开启\<Autoregistration>
确保 Agent 配置文件中的 `ServerActive` 指向正确的 Zabbix Server，且 `Hostname` 已设置。

#### 2. 确保: \<Zabbix Server服务端>侦听\<10051/tcp>
防火墙需要放行 Zabbix Server 的 10051 端口。

#### 3. 配置: <Zabbix Server服务端>执行<Autoregistration动作>
*   **步骤说明**: 在 `Configuration` -> `Actions` -> `Autoregistration actions` 中创建动作。
*   **解析: 定义栏**:
    *   **Conditions**: 触发条件，可以基于 `Hostname`、`Host metadata` 等。例如，`Host metadata contains "Linux"`。
    *   **Operations**: 执行的操作，如添加主机、链接 "Linux by Zabbix agent" 模板等。

## 05 - Zabbix Items 监控项
### 一、 实验环境
在已添加的主机上进行监控项的配置。

### 二、 了解: Items 监控项
#### 1. 了解: <items监控项>
*   **<items监控项>**: 一个 Item 就是一个从主机上采集的具体数据指标，例如 "CPU 使用率" 或 "可用内存"。
*   **<Key监控键>**: 每个 Item 的唯一标识符，定义了要采集什么数据。例如 `system.cpu.load[percpu,avg1]`。

#### 2. 了解: \<Zabbix Agent>的\<Key内置监控键>
*   **列出**: Zabbix Agent 预置了大量开箱即用的 Key，覆盖了大部分系统指标。
*   **调用**: 在创建 Item 时，直接选择或填写这些 Key 即可。例如 `vm.memory.size[available]` 用于获取可用内存。

### 三、 应用Items监控项
在主机的 `Items` 标签页，可以创建具体的监控项。
1.  **举例1: CPU监控项**: Key: `system.cpu.util[,user]` (用户态 CPU 使用率)
2.  **举例2: 内存监控项**: Key: `vm.memory.size[pavailable]` (可用内存百分比)
3.  **举例3: 磁盘监控项**: Key: `vfs.fs.size[/,pfree]` (/分区剩余空间百分比)
4.  **举例4: 网络监控项**: Key: `net.if.in[eth0,bytes]` (eth0 网卡入向流量)
5.  **举例5: 进程监控项**: Key: `proc.num[httpd]` (httpd 进程数量)
6.  **举例6: 内核监控项**: Key: `kernel.maxfiles` (系统最大打开文件数)

### 四、 自定义Items监控项
当内置 Key 无法满足需求时，可以自定义监控。

#### 1. 创建: <Zabbix Agent客户端代理>的<自定义Key>
*   **方法1: 命令法 (简单)**: 在 Agent 配置文件 `zabbix_agentd.conf` 中使用 `UserParameter` 指令。
    `UserParameter=my.custom.check,echo "hello"`
*   **方法2: 脚本法 (强大)**: `UserParameter` 可以执行一个脚本并返回其输出。
    `UserParameter=nginx.connections, /usr/local/bin/check_nginx.sh connections`

#### 2. 创建: <Zabbix自定义Item监控项>
*   **方法: 新建**: 在 Zabbix Web 界面的主机 `Items` 中，创建一个新的 Item。
*   **解析: 定义栏**:
    *   **Name**: 自定义一个易于理解的名称。
    *   **Type**: 选择 `Zabbix agent`。
    *   **Key**: 填写在 Agent 端定义的 `UserParameter` Key，如 `nginx.connections`。
    *   **Update interval**: 数据采集频率。

## 06 - Zabbix Visualization 可视化
### 一、 实验环境
基于已创建的 Items 进行可视化配置。

### 二、 了解Visualization (可视化)
1.  **Graphs(图形)**: 将一个或多个 Item 的历史数据以折线图等形式展示。
2.  **Maps(网络拓扑图)**: 创建网络拓扑或业务逻辑图，并关联主机或触发器状态。
3.  **Screens/Dashboards(仪表盘)**: 将多个图形、数据、地图等元素聚合在一个页面中，形成综合视图。

### 三、 创建Visualization Graph(图形)
1.  **前提**: 必须先有采集数据的 Items。
2.  **方法**: 在 `Configuration` -> `Hosts` -> (选择主机) -> `Graphs` 中创建。
3.  **演示**: 创建一个图形，将 "CPU user time" 和 "CPU system time" 两个 Item 添加到同一个图中进行对比。

### 四、 创建Visualization Map(网络拓扑图)
在 `Monitoring` -> `Maps` 中创建，通过拖拽图标、连线、设置标签来构建拓扑，并将图标与主机、主机组或触发器关联。

### 五、 创建Visualization Screen/Dashboard(仪表盘)
在 `Monitoring` -> `Dashboards` 中，可以添加各种小部件（widget），如图形、问题列表、时钟、纯文本数据等，自由组合成个性化监控面板。

## 07 - Zabbix Triggers 触发器
### 一、 实验环境
基于已创建的 Items 创建触发器。

### 二、 了解 Triggers 触发器
1.  **了解: 什么是<触发器>?**: 一个逻辑表达式，用于评估 Item 采集到的数据是否达到“问题”状态。当表达式为真时，触发器状态变为 `Problem`。
2.  **了解: <触发器>计算<时机>**: 每当 Zabbix Server 收到一个新的 Item 值时，就会重新计算与该 Item 相关的触发器表达式。

### 三、 创建Trigger 触发器
1.  **前提**: 必须先有用于判断的 Item。
2.  **方法**: 在 `Configuration` -> `Hosts` -> (选择主机) -> `Triggers` 中创建。
3.  **演示创建**:
    *   **名称**: `High CPU load on {HOST.NAME}`
    *   **表达式**: `last(/My-Linux-Host/system.cpu.load[percpu,avg1])>5`
    *   **含义**: 当主机 `My-Linux-Host` 的1分钟平均负载超过5时，触发器进入 `Problem` 状态。
4.  **压力测试**: 在被监控主机上运行高负载程序（如 `stress` 命令），观察 Zabbix 界面是否产生问题事件。

## 08 - Zabbix Templates 监控模板
### 一、 实验环境
在 Zabbix Web 前端进行模板相关操作。

### 二、 了解: Templates监控模板
1.  **了解: 什么是Template模板?**: 模板是一套可复用的监控实体集合，可以包含 Items, Triggers, Graphs, Discovery rules 等。
2.  **了解: Templates out of the box(开箱即用的模板)**: Zabbix 自带了大量针对常见操作系统和应用的官方模板，如 "Linux by Zabbix agent", "MySQL by Zabbix agent"。

### 三、 应用/取消应用: Templates监控模板
在主机的 `Templates` 标签页，可以链接或取消链接模板。链接后，模板中定义的所有实体都会应用到该主机上。

### 四、 创建/删除: Templates监控模板
在 `Configuration` -> `Templates` 中可以创建自定义模板。通常的做法是先创建一个空模板，然后向其中添加 Items, Triggers 等。

### 五、 更新: Template模板的<批量更新>
当修改了模板中的某个实体（如修改了 Item 的采集频率），所有链接了该模板的主机都会自动继承这个变更，实现批量管理。

### 六、 导出/导入: Templates监控模板
模板可以导出为 XML 或 JSON 文件，便于在不同的 Zabbix 系统之间分享和迁移。

## 09 - Zabbix Notifications Upon Events 事件通知
### 一、 实验环境
配置当触发器产生问题时，发送告警通知。

### 二、 了解 Notifications Upon Events事件通知
当触发器状态从 `OK` 变为 `Problem` 时，会产生一个事件（Event）。Zabbix 的告警系统基于这些事件来执行动作（Action），如发送通知。

### 三、 创建Notifications Upon Events事件通知
#### 1. 前提: 准备好Triggers 触发器
必须有能够产生问题事件的触发器。

#### 2. 创建: <Media报警媒介>
在 `Administration` -> `Media types` 中配置报警方式，如 Email, Webhook (用于钉钉、微信)。以 Email 为例，需要配置 SMTP 服务器信息。

#### 3. 配置: <报警媒介>关联的<用户>
在 `Administration` -> `Users` 中，为接收告警的用户（如 `Admin`）配置 `Media`，指定其接收告警的邮箱地址。

#### 4. 创建: <Action报警动作>
在 `Configuration` -> `Actions` -> `Trigger actions` 中创建动作：
*   **Conditions**: 定义触发动作的条件，如 `Trigger severity >= High`。
*   **Operations**: 定义要执行的操作，主要是发送消息。在操作中选择要通知的用户/用户组，以及使用的媒介类型。

## 10 - Zabbix Proxy 分布式代理
### 一、 了解: Zabbix Proxy分布式代理
Zabbix Proxy 是一个轻量级的数据收集器，可以代替 Zabbix Server 采集性能和可用性数据。所有收集到的数据会在本地进行缓冲，然后统一发送给 Zabbix Server。
**使用场景**:
*   监控远程分支机构。
*   减轻 Zabbix Server 的负载。
*   简化分布式环境的维护。

### 二、 实验环境
需要一台额外的服务器来扮演 Proxy 角色。

### 三、 部署: Zabbix Proxy分布式代理
#### 1. 安装部署: MySQL数据库
Proxy 也需要一个数据库（可以是 SQLite, MySQL, PostgreSQL）来存储配置和缓冲数据。步骤与 Server 端类似，但数据库规模可以小得多。

#### 2. 安装部署: Zabbix Proxy分布式代理
```bash
# 在 Proxy 服务器上安装仓库
rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/9/x86_64/zabbix-release-6.4-1.el9.noarch.rpm
dnf clean all

# 安装 Proxy 软件包
dnf install zabbix-proxy-mysql -y

# 编辑配置文件 /etc/zabbix/zabbix_proxy.conf
# Server: Zabbix Server 的 IP 地址
Server=192.168.1.100
# Hostname: Proxy 在 Zabbix Server 中注册的名称
Hostname=My-Zabbix-Proxy
# DBName, DBUser, DBPassword: 配置 Proxy 自己的数据库连接

# 启动并设置开机自启
systemctl restart zabbix-proxy
systemctl enable zabbix-proxy
```

#### 3. 安装部署: Zabbix Agent客户端代理
被 Proxy 监控的 Agent 安装方式与普通 Agent 相同。

### 四、 在\<Zabbix Server>上, 创建: Zabbix Proxy代理
在 `Administration` -> `Proxies` 中，创建一个新的 Proxy。
*   **Proxy name**: 必须与 Proxy 配置文件中的 `Hostname` 完全一致。
*   **Proxy mode**: Active (Proxy 主动连接 Server) 或 Passive (Server 连接 Proxy)。推荐使用 Active。

### 五、 在<被监控端>上, 配置: Zabbix Agent使用Zabbix Proxy
#### 1. 配置: Zabbix Agent使用Zabbix Proxy
修改被 Proxy 监控的 Agent 的配置文件 `zabbix_agentd.conf`：
`Server` 和 `ServerActive` 参数应指向 **Proxy 的 IP 地址**，而不是 Server 的。

#### 2. 配置: Zabbix Server通过Zabbix Proxy来采集
在 Zabbix Web 界面创建或编辑主机时，在 `Monitored by proxy` 字段选择之前创建的 Proxy。这样，Zabbix Server 就会委托该 Proxy 来监控这台主机。