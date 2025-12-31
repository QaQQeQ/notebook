
---


本章旨在通过构建一个完整的 LAMP (Linux + Apache + MariaDB + PHP) 环境，并将其与 Redis 集成，来演示 Redis 在真实 Web 应用中的使用场景，特别是作为数据库缓存的典型应用。

## **1. 快速构建 LAMP 环境**

为了简化 Apache、MySQL、PHP 的安装与配置，我们使用 **XAMPP** 集成包。XAMPP 是一个免费、易于安装的 Apache 发行版，内置了 MariaDB (MySQL 的一个分支)、PHP 和 Perl。    
XAMPP 官网 https://www.apachefriends.org/index.html

### **1.1 下载并安装 XAMPP**

1.  **下载安装程序**
    ```bash
    wget https://sourceforge.net/projects/xampp/files/XAMPP%20Linux/8.2.12/xampp-linux-x64-8.2.12-0-installer.run
    ```

2.  **添加执行权限并运行**  
    根据提示，一路按回车或输入 `y` 即可完成默认安装。
    ```
    [root@node1 ~]# chmod +x xampp-linux-x64-8.2.12-0-installer.run
    [root@node1 ~]# ./xampp-linux-x64-8.2.12-0-installer.run
    ```
    安装过程中的交互提示如下：
    ```text
    ----------------------------------------------------------------------------
    Welcome to the XAMPP Setup Wizard.
    ----------------------------------------------------------------------------
    Select the components you want to install...
    XAMPP Core Files : Y (Cannot be edited)
    XAMPP Developer Files [Y/n] :y
  
    Is the selection above correct? [Y/n]: y
    ----------------------------------------------------------------------------
    Installation Directory
    XAMPP will be installed to /opt/lampp
    Press [Enter] to continue:
    ----------------------------------------------------------------------------
    Setup is now ready to begin installing XAMPP on your computer.
    Do you want to continue? [Y/n]: y
    ----------------------------------------------------------------------------
    Please wait while Setup installs XAMPP on your computer.
    Installing
    0% ______________ 50% ______________ 100%
    ###############
    ----------------------------------------------------------------------------
    Setup has finished installing XAMPP on your computer.
    ```

### **1.2 修改 Apache 配置**

默认配置下，XAMPP 的 Web 服务只允许本地访问。我们需要修改配置以允许所有 IP 访问。

```bash
# 编辑 Apache 的 XAMPP 配置文件
[root@node1 ~]# vim /opt/lampp/etc/extra/httpd-xampp.conf
# 将 Require loacl 改为 Require all granted
```

### **1.3 安装依赖并启动服务**

1.  **安装 `libnsl` 依赖库**
    ```bash
    [root@node1 ~]# yum install -y libnsl
    ```

2.  **重启 XAMPP 服务**  
    XAMPP 的管理脚本位于 `/opt/lampp/lampp`。
    ```bash
    [root@node1 ~]# /opt/lampp/lampp restart
    Restarting XAMPP for Linux 8.2.12-0...
    XAMPP: Stopping Apache...not running.
    XAMPP: Stopping MySQL...not running.
    XAMPP: Stopping ProFTPD...not running.
    XAMPP: Starting Apache...ok.
    XAMPP: Starting MySQL...ok.
    XAMPP: Starting ProFTPD...ok.
    ```
    *   **启动**：`/opt/lampp/lampp start`
    *   **停止**：`/opt/lampp/lampp stop`

### **1.4 验证环境**

1.  **访问 Web 页面**  
    在浏览器中访问 `http://<your-ip-address>/dashboard/` (例如: `http://192.168.226.100/dashboard/`)，如果能看到 XAMPP 的欢迎页面，则表示 Apache 服务正常。

2.  **登录数据库**  
    XAMPP 内置的 MariaDB 默认 root 用户没有密码。数据库的默认配置文件为 `/opt/lampp/etc/my.cnf`
    ```bash
    [root@node1 ~]# /opt/lampp/bin/mysql -uroot -p
    Enter password:  # 直接回车
  
    Welcome to the MariaDB monitor...
    MariaDB [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | phpmyadmin         |
    | test               |
    +--------------------+
    ```

---

## **2. PHP 与 Redis 集成**

为了让 PHP 代码能够操作 Redis，需要安装 `phpredis` 扩展。  
离线下载地址：二选一  
https://github.com/phpredis/phpredis  
https://pecl.php.net/package/redis
### **2.1 下载并编译 `phpredis` 扩展**

1.  **下载源码包并解压**
    ```bash
    [root@node1 ~]# wget https://gitee.com/tianhairui/sh/raw/master/Software/phpredis-6.2.0.tar.gz
    [root@node1 ~]# tar -zxvf phpredis-6.2.0.tar.gz
    ```

2.  **安装编译工具 `autoconf`**  
    `phpize` 工具依赖 `autoconf`。
    ```bash
    [root@node1 ~]# yum install -y autoconf
    ```

3.  **编译和安装扩展**  
    进入源码目录，使用 XAMPP 自带的 `phpize` 和 `php-config` 工具进行编译。
    ```bash
    [root@node1 ~]# cd phpredis-6.2.0/
  
    # 1. 生成 configure 文件
    [root@node1 phpredis-6.2.0]# /opt/lampp/bin/phpize
  
    # 2. 配置编译选项，指定 PHP 配置路径
    [root@node1 phpredis-6.2.0]# ./configure --with-php-config=/opt/lampp/bin/php-config
  
    # 3. 编译并安装
    [root@node1 phpredis-6.2.0]# make && make install
    ```
    安装成功后，会看到类似如下的输出，提示扩展已安装到指定目录：  
    `Installing shared extensions: /opt/lampp/lib/php/extensions/no-debug-non-zts-20220829/`

### **2.2 配置 `php.ini`**

1.  **编辑 `php.ini` 文件**
    ```bash
    [root@node1 phpredis-6.2.0]# vim /opt/lampp/etc/php.ini
    ```

2.  **添加 Redis 扩展配置**  
    在文件末尾添加以下两行：
	```
    extension_dir='/opt/lampp/lib/php/extensions/no-debug-non-zts-20220829/'
    extension=redis.so
	```

3.  **重启 XAMPP**
    ```bash
    [root@node1 phpredis-6.2.0]# /opt/lampp/lampp restart
    ```

### **2.3 验证集成**

刷新 XAMPP 欢迎页 (`http://<your-ip-address>/dashboard/`)，点击顶部的 "PHPInfo" 链接。在打开的页面中搜索 "redis"，如果能找到 Redis 相关的配置信息，说明扩展已成功加载。

---

## **3. 综合测试：使用 Redis 作为 MySQL 缓存**

这个测试模拟一个常见的 Web 应用场景：优先从 Redis 读取数据，如果 Redis 中没有，则从 MySQL 查询，并将结果存入 Redis 以便下次快速访问。  
>注意：测试的redis环境为前面的分布式环境

### **3.1 准备 MySQL 测试数据**

1.  **登录 MySQL**
    ```bash
    /opt/lampp/bin/mysql -uroot -p
    ```

2.  **创建数据库、表并插入数据**
    ```sql
    -- 创建数据库
    create database gzc;
    use gzc;
  
    -- 创建表
    create table abc(id int not null auto_increment primary key,name varchar(20));
  
    -- 插入初始数据
    insert into abc(name) values('tom'),('henry'),('natasha');
    commit;
  
    -- 创建一个用于 PHP 连接的只读用户
    create user access01@'%' identified by '123456';
    grant select on gzc.* to access01@'%';
    flush privileges;
    ```

### **3.2 部署 PHP 测试脚本**

1.  **下载脚本**
    ```bash
    [root@node1 ~]# wget -O /opt/lampp/htdocs/custom.php https://gitee.com/tianhairui/sh/raw/master/Application/RedisCluster2MySQL.php
    ```

2.  **修改脚本配置**
    > **重要提示**：必须编辑该文件，修改其中的 Redis 集群节点信息和 MySQL 数据库连接信息（IP、用户名、密码），以匹配您的实际环境。
	    
    ```bash
    [root@node1 ~]# vim /opt/lampp/htdocs/custom.php
    ```
    [RedisCluster2MySQL.php](脚本/RedisCluster2MySQL.php.md)

### **3.3 测试缓存逻辑**

1.  **首次访问**
    在浏览器中访问 `http://<your-ip-address>/custom.php`。页面会显示 "数据来源于: MySQL"，并列出从数据库中查询到的三条记录。

2.  **验证 Redis 缓存**
    此时登录 Redis 集群，查询对应的 key，会发现数据已被缓存。
    ```bash
    [redis@node1 ~]$ redis-cli -c -h 192.168.226.100 -p 7001 -a redhat
    192.168.226.100:7001> get 1
    -> Redirected to slot [15495] located at 192.168.226.102:7005
    "tom"
    192.168.226.102:7005> get 2
    -> Redirected to slot [5649] located at 192.168.226.100:7001
    "henry"
    ```

3.  **再次访问**
    刷新 `custom.php` 页面，此时会显示 "数据来源于: Redis"，表示数据已从缓存中快速读取。

### **3.4 缓存一致性问题演示**

1.  **在 MySQL 中更新数据**
    在 MySQL 中直接插入新数据，模拟后台业务变更。
    ```sql
    insert into abc(name) values('sissi'),('jerry');
    commit;
    ```

2.  **观察 Web 页面**
    再次刷新 `custom.php` 页面，会发现新插入的 `sissi` 和 `jerry` **并不会显示**。这是因为 PHP 程序仍然从 Redis 中读取到了旧的缓存数据。

3.  **缓存同步问题说明**
    *   **问题**：当后端数据库（MySQL）的数据发生变化（更新、删除）时，Redis 中的缓存不会自动更新，导致数据不一致。
    *   **简单解决方案**：手动清空 Redis 缓存。
        *   `flushall`: 清空所有 Redis 数据库的 key。
        *   `flushdb`: 清空当前数据库的 key。
    *   **生产级解决方案**：在应用层面实现缓存更新逻辑（例如，更新 MySQL 后主动删除或更新 Redis 中的对应缓存），或者使用更复杂的机制如数据库触发器、消息队列等（不推荐在数据库层面直接操作）。