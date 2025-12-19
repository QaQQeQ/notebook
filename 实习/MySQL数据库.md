1、mariadb安装（是mysql下面的一个开源分支）
前提：yum源，网络正常
yum -y install mariadb mariadb-server  #安装
systemctl start mariadb
初始化数据库
mysql_secure_installation     回车---Y---两遍密码--后面默认回车

登录：mysql -uroot  -p密码



2、mysql安装（只能使用官方源）www.mysql.com
1、卸载mariadb
```bash
yum -y remove mariadb*
```

2、下载安装yum源
```bash
wget https://dev.mysql.com/get/mysql84-community-release-el7-2.noarch.rpm
```

```bash
rpm -ivh mysql84-community-release-el7-2.noarch.rpm #会生成2个repo文件
```

3、安装mysql服务端客户端
yum -y install mysql-server

4、启动服务
systemctl start mysqld
如果之前安装过mariadb  需要删除旧数据 rm -rf /var/lib/mysql

5、查看临时密码
```bash
cat /var/log/mysqld.log | grep password
```

6、修改密码（多种方式）
使用初始化工具重置密码
mysql_secure_installation
输入初始密码----y---2遍密码--y---2遍密码（满足复杂性要求）--默认回车

7、使用新密码登录
mysql -uroot -pAa123456!
mysql> select version();   #查看数据库的版本

复习任务：
1、创建数据库hbkj
2、在hbkj库中创建一个student表（id,name,age,num）
3、表里面插入数据
4、创建一个普通用户zhangsan
5、zhangsan对hbkj有所有权限




[[MySQL主从复制]]

































