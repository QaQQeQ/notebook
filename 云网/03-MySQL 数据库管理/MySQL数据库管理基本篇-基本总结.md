


---

#### **第一部分：MySQL 基础入门**

这部分主要介绍数据库的基本概念、MySQL的安装部署以及核心组件。

**1. 数据库基础概念**
*   **数据库系统 (DBS)**: 由数据库(DB)、数据库管理系统(DBMS)、数据库管理员(DBA)和应用程序组成的系统。
*   **逻辑构成**: 客户端 -> 连接管理 -> SQL接口 -> 解析器 -> 优化器 -> 缓存 -> **存储引擎** -> 物理文件。
*   **SQL语言分类**:
    *   **DQL**: 数据查询语言，如 `select`。
    *   **DML**: 数据操纵语言，如 `insert`, `update`, `delete`。
    *   **DDL**: 数据定义语言，如 `create`, `alter`, `drop`。
    *   **DCL**: 数据控制语言，如 `grant`, `revoke`。

**2. MySQL 安装与部署**
*   **三种主要安装方式**:
    1.  **YUM/APT 安装 (推荐)**:
        *   通过系统的包管理器安装，如 `yum install mysql-server`。
        *   优点：安装、升级、卸载简单，依赖自动处理，与系统集成度高。
        *   步骤：配置YUM源 -> 安装软件包 -> 启动服务 (`systemctl start mysqld`) -> 安全初始化 (获取临时密码、修改密码策略、设置新密码)。
    2.  **二进制包安装**:
        *   下载官方编译好的二进制包，解压后进行配置。
        *   优点：比YUM灵活，比源码编译简单。
    3.  **源码编译安装 (Make)**:
        *   下载源码，使用 `cmake` 和 `make` 工具进行编译安装。
        *   优点：最高度的定制化，可以优化编译参数。
        *   缺点：过程极其复杂，耗时长，易出错，需要手动解决依赖。

*   **进程管理**:
    *   使用 `systemctl` 命令管理 `mysqld` 服务：`start`, `stop`, `restart`, `status`, `enable`。

**3. 存储引擎**
*   **定义**: 位于MySQL底层，负责数据的实际存储、读取、索引和事务管理。表级别的概念，一个库中可以有不同引擎的表。
*   **两大核心引擎**:
    *   **InnoDB (默认)**:
        *   **特性**: 支持**事务 (ACID)**、**行级锁**、**外键约束**、崩溃安全恢复。
        *   **适用**: 绝大多数场景，特别是需要高并发、数据一致性和可靠性的应用。
    *   **MyISAM (已不常用)**:
        *   **特性**: **表级锁**、不支持事务和外键、访问速度快（特别是读操作）。
        *   **适用**: 读密集、非事务性的简单应用，或需要全文索引的旧场景。

---

#### **第二部分：数据库核心对象管理**

这部分深入学习表结构、数据类型和约束，是数据库设计的基础。

 **4. 库、表、记录管理 (DDL 和 DML)**

*  **库管理 (DDL)**
	*   **创建数据库**: `create database 数据库名;`
	*   **查看所有数据库**: `show databases;`
	*   **选中/进入数据库**: `use 数据库名;`
	*   **删除数据库**: `drop database 数据库名;`

*  **表管理 (DDL)**
	*   **创建表**:
	    *   `create table 表名 (列名1 数据类型 [约束], ...);`
	    *   **示例**: 
	        ```sql
	        create table student (
	            id int auto_increment primary key,
	            name varchar(50) not null,
	            age int unsigned,
	            birthday date
	        );
	        ```

	*   **查看表结构**:
	    *   `desc 表名;` 或 `show create table 表名;`
	    *   **示例**:
	        ```sql
	        -- 查看基本结构
	        desc students;
	      
	        -- 查看完整的创建语句
	        show create table students;
	        ```

	*   **修改表 (`alter table`)**:
	    *   **添加新列 (`add column`)**:
	        *   **示例**: 为 `students` 表添加一个 `email` 列。
	            ```sql
	            alter table students add column email varchar(100) unique;
	            ```
	
	    *   **修改列的数据类型或约束 (`modify column`)**:
	        *   **示例**: 将 `email` 列的长度修改为 150。
	            ```sql
	            alter table students modify column email varchar(150);
	            ```
	
	    *   **修改列名和数据类型 (`change column`)**:
	        *   **示例**: 将 `email` 列重命名为 `student_email`，并修改其类型。
	            ```sql
	            alter table students change column email student_email varchar(200);
	            ```
	
	    *   **删除列 (`drop column`)**:
	        *   **示例**: 删除 `student_email` 列。
	            ```sql
	            alter table students drop column student_email;
	            ```
	          
	    *   **修改表名 (`rename to`)**:
	        *   **示例**: 将 `students` 表重命名为 `student_info`。
	            ```sql
	            alter table students rename to student_info;
	            ```

	*   **克隆表**:
	    *   **示例**:
	        ```sql
	        -- 方法一：仅复制表结构
	        create table student_backup_struct like student_info;
	      
	        -- 方法二：复制表结构和所有数据
	        create table student_backup_data as select * from student_info;
	        ```
	
	*   **删除表**: `drop table 表名;`

* **记录管理 (DML)**

	*   **插入记录**:
	    *   `insert into 表名 (列...) values (值...);`
	    *   **示例**:
	        ```sql
	        -- 插入完整记录,必须严格字段个数及顺序，依次插入字段值
	        insert into student values (1,'张三',18,'2005-09-01')
										(2,'李四',20,'2005-10-01');
	
	        -- 不必严格遵循字段个数及顺序，乱序插入字段值
	        insert into student (name,age) values ('wangwu',19)
													('qianqi',20);
	      
	        -- 不必严格遵循字段个数及顺序，乱序插入字段值
	        insert into student set name='王五',age=22;
	        ```

	*   **更新记录**:
	    *   `update 表名 set 列=新值 where 条件;`
	    *   **示例**:
	        ```sql
	        -- 更新 id 为 1 的学生年龄
	        update student_info set age = 19 where id = 1;
	        ```

	*   **删除记录**:
	    *   `delete from 表名 where 条件;`
	    *   **示例**:
	        ```sql
	        -- 删除 id 为 4 的学生记录
	        delete from student_info where id = 4;
	        ```
      
	*   **清空表记录**:
	    *   `truncate 表名;`

**5. 数据类型、约束与表达式**
*   **数据类型**:
    *   **数值型**: `int`, `decimal`, `float`/`double`, `bit`。
    *   **字符串型**: `char`, `varchar`, `text`。
    *   **日期时间型**: `date`, `time`, `datetime`, `timestamp`, `year`。
*   **完整性约束**:
    *   **实体完整性**: `primary key`, `unique`。保证行的唯一性。
    *   **域完整性**: `not null`, `default`, `check` (MySQL 8.0+ 支持)。保证列值的有效性。
    *   **引用完整性**: `foreign key`。保证多个表之间的引用关系正确。
*   **表达式与运算符**:
    *   算术运算符 (`+`, `-`, `*`, `/`)，比较运算符 (`=`, `>`, `<`, `!=`)，逻辑运算符 (`and`, `or`, `not`)。
    *   特殊运算符: `between...and...`, `in(...)`, `like` (通配符`%`, `_`) , `is [not] null`。

---

#### **第三部分：数据查询与高级操作**

这是DBA和开发人员日常工作的核心，涵盖了从查询到运维的各个方面。

**6. SELECT 查询 (DQL)**
*   **单表查询**:
    *   基本语法: `select [distinct] 列 from 表`
    *   `where`: 条件过滤
    *   `group by`: 分组
    *   `having`: 对分组后的结果进行过滤
    *   `order by`: 排序 (`asc`升序, `desc`降序)
    *   `limit`: 限制返回的行数
    *   **聚合函数**: `count()`, `sum()`, `avg()`, `max()`, `min()`
*   **多表查询 (JOIN)**--**连接条件用on**:
    *   **内连接 (inner join)**: 返回两表中能匹配上的记录。
    *   **左连接 (left join)**: 返回左表所有记录，以及右表中能匹配上的记录。
    *   **右连接 (right join)**: 返回右表所有记录，以及左表中能匹配上的记录。

**7. 用户与权限管理 (DCL)**
*   **账户管理**:
    *   创建用户: `create user '用户名'@'主机' identified by '密码';` (主机可以是 `localhost`, `IP地址`, `%`通配符)
    *   删除用户: `drop user '用户名'@'主机';`
    *   修改密码: `alter user '用户名'@'主机' identified by '新密码';`
*   **权限管理**:
    *   授权: `grant 权限 on 库.表 to '用户名'@'主机';`
    *   撤销权限: `revoke 权限 on 库.表 from '用户名'@'主机';`
    *   查看权限: `show grants for '用户名'@'主机';`
    *   刷新权限: `flush privileges;` (修改授权表后通常需要执行)。

**8. 索引管理**
*   **定义**: 一种特殊的数据结构（B+树），能极大提高数据检索速度，但会降低写操作的性能。
*   **工作原理**: 类似于书的目录，通过索引直接定位到数据所在物理位置，避免全表扫描。
*   **分类**:
    *   **聚集索引 (Clustered Index)**: 索引的顺序就是数据的物理存储顺序。一个表只能有一个，通常是主键。
    *   **非聚集索引 (Secondary Index)**: 索引和数据分开存储，索引存储了指向数据行的指针。
*   **管理**: `create index`, `drop index`, `show index from 表名`。
*   **设计原则**: 在 `where`, `join`, `order by` 中频繁使用的列上创建索引；选择区分度高的列；避免在索引列上进行函数运算。

**9. 日志管理**
*   **错误日志 (Error Log)**: 记录启动、运行、关闭过程中的严重错误信息。
*   **通用查询日志 (General Query Log)**: 记录所有到服务器的连接和执行的语句。（影响性能，仅调试时开启）
*   **慢查询日志 (Slow Query Log)**: 记录执行时间超过 `long_query_time` 阈值的SQL语句，用于性能优化。
*   **二进制日志 (Binary Log / binlog)**: **极其重要**。记录所有修改了数据的SQL语句(DDL, DML)，用于**数据恢复**和**主从复制**。

**10. 备份与还原**
* 逻辑备份 (Logical Backup): `mysqldump` 工具
	逻辑备份的核心是导出一个包含完整 `create` 和 `insert` 等语句的SQL脚本文件。它是最常用、最灵活的备份方式。

	*   **优点**:
	    *   备份文件是可读的SQL文本，易于理解和编辑。
	    *   与存储引擎无关，跨版本、跨平台，兼容性好。
	*   **缺点**:
	    *   恢复时需要逐条执行SQL，对于大型数据库，恢复速度较慢。

	**`mysqldump` 核心用法与数据一致性选项**

	在备份过程中，为保证数据的一致性快照，`mysqldump` 提供了两种关键机制：

	*   **机制一：`--lock-all-tables` (温备份)**
	    *   **原理**: 在备份开始时，对所有需要备份的表施加一个全局读锁，阻塞所有写操作。
	    *   **适用场景**: 包含 **MyISAM** 或混合存储引擎的数据库。这是保证混合引擎数据一致性的**唯一方法**。
	    *   **缺点**: 备份期间会阻塞业务写入，属于“温备份”。
	
	*   **机制二：`--single-transaction` (热备份 - 推荐)**
	    *   **原理**: 利用InnoDB存储引擎的**事务**和**多版本并发控制(MVCC)**特性。在备份开始时启动一个事务，从而获取一个一致性的数据视图。
	    *   **适用场景**: 数据库中的所有表**均为InnoDB**存储引擎。
	    *   **优点**: 备份期间**不锁表**，不影响业务的正常读写，实现真正的“热备份”。

	**常用备份命令示例:**
	
	1.  **全库备份 (混合引擎)**
	    *   使用全局锁来保证所有库（包括MyISAM表）的一致性。
	    ```bash
	    mysqldump -u root -p --lock-all-tables --all-databases > full_backup.sql
	    ```
	
	2.  **多库备份 (仅InnoDB引擎)**
	    *   使用事务模式进行热备份，对业务无影响。
	    ```bash
	    mysqldump -u root -p --single-transaction --databases db1 db2 > multi_db_backup.sql
	    ```
	
	3.  **多表备份 (仅InnoDB引擎)**
	    *   备份指定数据库中的特定几个表。
	    ```bash
	    mysqldump -u root -p --single-transaction db1 table1 table2 > multi_table_backup.sql
	    ```

		**逻辑恢复 (Restore)**
		
		恢复操作非常简单，就是将导出的SQL脚本文件通过`mysql`客户端重新导入执行即可。
		
		```bash
		# 登录到mysql后执行
		source /path/to/your/backup.sql;
		
		# 或者在shell命令行直接执行
		mysql -u root -p < /path/to/your/backup.sql
		```

* 物理备份 (Physical Backup): 冷备份

	物理备份是直接复制数据库的物理数据文件。冷备份是其中最简单直接的一种。
	
	*   **优点**:
	    *   备份和恢复速度极快，因为它只是文件拷贝。
	*   **缺点**:
	    *   操作期间**必须关闭数据库服务**，会导致业务中断。
	    *   依赖于文件系统和MySQL版本，可移植性较差。

	**冷备份与恢复步骤:**
	
	1.  **执行备份 (Backup)**
	    ```bash
	    # 1. 停止MySQL服务，确保没有数据写入
	    systemctl stop mysqld
	  
	    # 2. 将整个MySQL数据目录打包
	    #    (数据目录通常是 /var/lib/mysql)
	    tar -czf /backup/mysql-data.tar.gz /var/lib/mysql
	  
	    # 3. 重新启动MySQL服务
	    systemctl start mysqld
	    ```
	
	2.  **执行恢复 (Restore)**
	    ```bash
	    # 1. 停止MySQL服务
	    systemctl stop mysqld
	  
	    # 2. 清空或重命名当前损坏的数据目录（非常重要！）
	    rm -rf /var/lib/mysql/*
	  
	    # 3. 将备份文件解压到原数据目录
	    tar -xzf /backup/mysql-data.tar.gz -C /
	  
	    # 4. 确保文件权限正确（用户和组通常是 mysql）
	    chown -R mysql:mysql /var/lib/mysql
	  
	    # 5. 重新启动MySQL服务
	    systemctl start mysqld
	    ```

* **补充：表数据的文本导入导出**

	这是一种针对单个表、轻量级的数据迁移方式，不属于完整的备份方案。
	
	*   **数据导出: `select ... into outfile`**
	    *   将查询结果直接保存为服务器上的一个文件。
	    *   **注意**: MySQL的`secure_file_priv`安全参数会限制导出目录，需要提前配置。
	    ```sql
	    select * from my_db.my_table 
	    into outfile '/var/lib/mysql-files/my_table.txt';
	    ```
	
	*   **数据导入: `load data infile`**
	    *   从服务器上的一个文件快速加载数据到表中，性能远高于`insert`语句。
	    ```sql
	    load data infile '/var/lib/mysql-files/my_table.txt' 
	    into table my_db.my_table;
	    ```

**11. 主从复制 (Replication)**
*   **定义**: 将主库 (Master) 的数据变更实时同步到一个或多个从库 (Slave) 的过程。
*   **工作原理**:
    1.  Master将数据变更写入自己的 **Binary Log**。
    2.  Slave启动一个 **IO 线程**，连接到 Master，请求并拉取 Binlog，写入自己的 **Relay Log** (中继日志)。
    3.  Slave启动一个 **SQL 线程**，读取 Relay Log 中的事件，并在本地重放执行，实现数据同步。
*   **核心用途**: 读写分离、高可用、数据备份。
*   **拓扑结构**: 一主一从、一主多从、双主复制等。
*   **关键配置**: `server_id` (每个节点必须唯一), `log-bin` (主库必须开启)。