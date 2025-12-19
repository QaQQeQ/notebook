wordpress：基础php开发的一款cms系统（内容发布系统）
步骤：
server1  192.168.100.10   安装：nginx+php  
server2 192.168.100.20   安装：mariadb

1、nginx安装完成
2、php、php-fpm安装完成
3、nginx与php集成完成
4、mysql数据库部署完成
5、数据库、用户创建、授权
     例如：创建一个数据库wordpress  创建一个用户user1

6、部署wordpress网页代码（放到网站根目录）
	wget https://cn.wordpress.org/wordpress-4.9.26-zh_CN.tar.gz
    tar -axf wordpress-4.9.26-zh_CN.tar.gz
    mv wordpress/* /usr/local/nginx/html/
    yum -y install php-mysql  #php连接mysql驱动

7、根据页面向导完成后续安装
   访问：浏览器访问nginx服务ip地址，根据提示完成安装向导。
   wp-config.php ：这个文件放在网站根目录




[[lnmp部署oa系统]]