DNS:域名解析    域名------>ip

安装配置：
server1    192.168.100.10  服务端
server2    192.168.100.20  客户端

可以关闭防火墙和selinux

## 在服务端操作
1、安装软件包（bind bind-utils）

2、修改配置
/etc/named.conf    # 主配置文件
/etc/named.rfc1912.zones   #区域控制文件 （为哪个域名做解析）可以自定义
/var/named/zone.xx.com    #数据文件  （具体的解析记录）

2.1 修改主配置文件named.conf
3个any
......      
  listen-on port 53 { any; };
  listen-on-v6 port 53 { any; };
......
  allow-query     { any; };

2.2 修改区域控制文件named.rfc1912.zones
按照正向解析区域复制5行，进行修改，如下：
zone "hbkj.com" IN {                  #为哪个域名做解析
        type master;
        file "zone.hbkj.com";          #指定数据文件的名字
        allow-update { none; };
};

2.3 修改区域数据文件zone.hbkj.com  （与之前指定的文件名必须一致）
  cd /var/named/
  cp -a named.localhost zone.hbkj.com
  vim zone.hbkj.com
参考内容如下：
```
$TTL 1D
@       IN SOA  @ hbkj.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
          NS     dns.hbkj.com.    #指定dns服务地址
dns     A       192.168.100.10  #必须先解析dns服务器自己的地址
www   A       1.1.1.1             # 主机记录    主机名-----ip        
mail    A       2.2.2.2
```

重启服务：
systemctl restart named

如果重启报错，进行检查：
named-checkconf /etc/named.conf    #检查配置文件
named-checkzone hbkj.com /var/named/zone.hbkj.com    #检查数据文件
                                  域名              数据文件名

## 客户端测试：
1、客户端修改网卡配置文件，dns指向自己的dns服务器
2、安装测试工具：yum -y install bind-utils
3、测试解析：
   nslookup www.hbkj.com
  解析结果与数据文件中写的ip地址一致







[[Samba文件服务]]