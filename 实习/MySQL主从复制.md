#mysql主从复制  （mysql8.0以下、mariadb）
1、环境准备：
master  192.168.100.10  安装mysql服务
slave  192.168.100.11    安装mysql服务

2、修改配置文件：/etc/my.cnf   主从都要修改
...在[mysqld]里面添加
server_id=2        #主从的编号不能一样
log_bin=mybinlog
....

3、重启服务配置主从复制
主节点创建同步用的用户：
mysql> create user 'repl'@'%' identified by 'Aa123456#';
mysql> grant replication slave, replication client on *.* to 'repl'@'%';
mysql -urepl -p123456 -h 192.168.100.10  #可以在从节点测试登录

mysql> show master status;  #查看最新的日志名称和位置


在从节点配置主节点信息：

```
> [!NOTE]
> change master to
>               master_host='192.168.100.10',       #主节点地址
>               master_user='repl',                         #之前创建的用户
>               master_password='Aa123456#',
>               master_log_file='binlog.000001',   #show master status; 查看的结果
>               master_log_pos=648;
> 
> start slave;
> show slave status \G;
> .....
> 			Slave_IO_Running: Yes
>             Slave_SQL_Running: Yes   #确保这两个线程状态是yes
> ..........
```

测试：主节点写入数据，，从节点可以自动同步。





[[nginx服务]]