文件服务：共享、传输文件的服务
常用的文件服务：ftp(下载居多，传输)、samba（共享，windows客户端）、NFS（linux服务器直接的数据共享）

#samba服务安装配置
1、环境准备：静态ip、网络正常、主机名、防火墙、yum换源
2、安装samba
yum -y install samba

3、启动服务（两个服务）
systemctl enable smb nmb   #设置为开机自启
systemctl start smb nmb

4、修改配置 ：/etc/samba/smb.conf
```
[global]
        workgroup = SAMBA
        security = user                     #启用用户验证
        passdb backend = tdbsam  #认证数据库方式

        printing = cups
        printcap name = cups
        load printers = yes
        cups options = raw

[homes]                                              #每个用户的家目录，可以整个删掉
        comment = Home Directories
        valid users = %S, %D%w%S
        browseable = No
        read only = No
        inherit acls = Yes

[公告]                                                #访问时显示的名称
        comment = 描述信息
        browseable = Yes                   #访问共享服务的时候目录是否显示
        read only = No                        #是否只读
        path = /share                          #共享目录真实的路径
```

创建共享目录与path指定的一致
mkdir /share
touch /share/1.txt


5、创建samba用户（基于系统用户）
useradd zhangsan -M -s /sbin/nologin   #创建系统用户，不能登录系统
smbpasswd -a zhangsan  #将系统用户添加为samba用户

6、windows测试访问
打开“运行” \\192.168.100.10

#权限管理
最终权限：samba设置的权限和linux目录本地权限共同决定（交集）
不同用户的权限管理
valid users = zhangsan  #指定用户访问

net use * /delete   #清除之前登录信息


作业：
公司目录（财务部，人事部）
财务部：zhangsan
人事部：lisi
经理：wangwu
要求：1.只能访问自己部门的共享目录，并且有写的权限
       2.经理可以看所有文件，但不能修改






[[MySQL数据库]]