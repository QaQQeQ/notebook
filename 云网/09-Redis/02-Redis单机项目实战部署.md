
---

本章内容为在 **RockyLinux 9.6** 平台上，通过源码编译方式部署 **Redis 8.2.0** 的详细步骤。

# **1. 环境准备**

在安装 Redis 之前，需要对操作系统进行基础配置。

## **1.1 关闭防火墙及 SELinux**

为避免端口和权限问题干扰 Redis 服务，建议关闭防火墙和 SELinux。

- **临时关闭防火墙**:
  ```bash
  systemctl stop firewalld
  ```
- **永久禁用防火墙**:
  ```bash
  systemctl disable firewalld
  ```
- **临时关闭 SELinux**:
  ```bash
  setenforce 0
  ```
- **永久禁用 SELinux**:
  修改配置文件 `/etc/selinux/config`，将 `SELINUX=enforcing` 改为 `SELINUX=disabled`。
  ```bash
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
  ```

## **1.2 安装基础软件包**

安装编译 Redis 所需的依赖工具，如 GCC 编译器、make 等。

```bash
dnf install -y gcc make tcl vim bash-completion net-tools wget zip unzip bzip2 gcc-c++ tar
```

## **1.3 创建专用用户及目录**

为了安全起见，使用非 root 用户运行 Redis。

1.  **创建用户组和用户**:
    ```bash
    groupadd -g 820 redis
    useradd -u 8200 -g 820 redis
    echo '123456' | passwd --stdin redis
    ```
2.  **创建 Redis 工作目录** (日志、配置、数据):
    ```bash
    mkdir -p /redis/install/{log,conf,data}
    ```
3.  **授权目录给 redis 用户**:
    ```bash
    chown -R redis.redis /redis/install
    ```

## **1.4 设置系统资源限制**

为 `redis` 用户提高最大打开文件数（`nofile`）和最大进程数（`nproc`）的限制，以满足高并发需求。

- **编辑 `/etc/security/limits.conf` 文件并添加以下内容**:
```bash
cat <<"EOF" >> /etc/security/limits.conf
redis        hard    nofile          10240
redis        soft    nofile          10240
redis        hard    nproc           8192
redis        soft    nproc           8192
EOF
```
---

# **2. 源码编译与安装**

## **2.1 安装 jemalloc (高性能内存分配器)**

Redis 默认使用 jemalloc 作为内存分配器，如果系统里没有 jemalloc，编译 Redis 会提示缺失依赖，或者使用系统默认的 glibc malloc，导致性能下降、内存碎片严重

1.  **下载源码包**:
    ```bash
    # 从 GitHub 下载
    wget -P /root/ https://github.com/jemalloc/jemalloc/releases/download/5.3.0/jemalloc-5.3.0.tar.bz2
    # 或从 Gitee 镜像下载
    # wget -P /root/ https://gitee.com/tianhairui/sh/raw/master/Software/jemalloc-5.3.0.tar.bz2
    ```
2.  **解压并编译安装**:
    ```bash
    tar -jxvf /root/jemalloc-5.3.0.tar.bz2
    cd jemalloc-5.3.0
    ./configure
    make && make install
    ```

## **2.2 编译安装 Redis**

1.  **下载 Redis 源码包**:
    ```bash
    # 从 GitHub 下载
    wget -P /root/ https://github.com/redis/redis/archive/refs/tags/8.2.0.tar.gz
    # 或从 Gitee 镜像下载
    # wget -P /root/ https://gitee.com/tianhairui/sh/raw/master/Software/8.2.0.tar.gz
    ```
2.  **源码编译安装 Redis**:
    ```bash
    # 将源码包移动到工作目录并授权
    mv /root/8.2.0.tar.gz /redis/install/
    chmod 777 -R /redis/install/*

    # 切换到 redis 用户
    su - redis

    # 解压
    cd /redis/install
    tar -zxvf 8.2.0.tar.gz
    cd /redis/install/redis-8.2.0/deps

    # 编译依赖
    make fast_float jemalloc linenoise lua hiredis

    # 编译 Redis
    cd /redis/install/redis-8.2.0
    make

    # 安装到指定目录
    cd /redis/install/redis-8.2.0/src
    make PREFIX=/redis/install install
    
    # 复制默认配置文件
    cp ../redis.conf /redis/install/conf/

    # 添加 Redis 命令到环境变量
    echo 'export PATH=$PATH:$HOME/.local/bin:/redis/install/bin:$HOME/bin' >> ~/.bash_profile
    source ~/.bash_profile
    ```

---

# **3. Redis 配置与管理**

## **3.1 修改配置文件 `redis.conf`**

切换到 `redis` 用户，编辑 `/redis/install/conf/redis.conf` 文件，修改以下关键配置：

- **`bind 192.168.10.134 -::1`**: 监听指定的 IP 地址（请替换为你的服务器 IP）。
- **`daemonize yes`**: 以守护进程（后台）模式运行。
- **`logfile "/redis/install/log/redis.log"`**: 指定日志文件路径。
- **`requirepass redhat`**: 设置连接密码为 `redhat`（请替换为强密码）。

## **3.2 启动与验证服务**

1.  **启动 Redis 服务**:
    ```bash
    # 确保是 redis 用户
    su - redis
    redis-server /redis/install/conf/redis.conf
    ```
2.  **检查端口是否监听**:
    ```bash
    ss -lntp | grep 6379
    ```
3.  **登录并测试**:
    ```bash
    # -h 指定主机，-a 指定密码
    redis-cli -h 192.168.10.134 -a redhat

    # 进入后测试
    192.168.10.134:6379> ping
    PONG
    192.168.10.134:6379> set mykey "hello"
    OK
    192.168.10.134:6379> get mykey
    "hello"
    ```

## **3.3 关闭服务**

- **方法一：通过 redis-cli 关闭** (推荐)
  ```bash
  redis-cli -h 192.168.208.134 -a redhat shutdown
  ```
- **方法二：登录后关闭**
  ```bash
  redis-cli -h 192.168.208.134 -a redhat
  192.168.208.134:6379> shutdown
  ```

## **3.4 密码管理**
- **登陆后**
- **查看当前密码**:
  ```bash
  config get requirepass
  ```
- **临时修改密码** (服务重启后失效):
  ```bash
  CONFIG SET requirepass "new_password"
  ```
- **永久修改密码**:
  直接修改 `redis.conf` 文件中的 `requirepass` 配置项，然后重启 Redis 服务。