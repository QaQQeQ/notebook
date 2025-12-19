注意：数据库与php的版本，yum源版本过低，需要更换安装源
步骤：
server1  192.168.100.10   安装：nginx+php  
server2 192.168.100.20   安装：mysql8.0

1、nginx安装完成
2、php、php-fpm安装完成（7.0以上版本）
源码编译安装（步骤见手册）


3、nginx与php集成完成


4、mysql数据库部署完成（8.0以上版本）
    rm -rf /var/lib/mysql  #如果之前有旧版本，需要删除数据目录

5、用户创建、授权
     例如：  创建一个用户user1@%

6、部署ZenTaoPMS网页代码（放到网站根目录）
       见手册





[[nginx扩展]]