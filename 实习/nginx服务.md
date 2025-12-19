nginx：轻量级、高并发、高性能的静态web应用
高级功能：代理服务、负载均衡器

#nginx源码安装  （编译安装）
前提环境：静态ip、网络、yum源换国内、firewalld和selinux关掉
不能运行有其他的web应用：netstat -tunlp  #检查有没有80端口
1、下载（www.nginx.org）
yum -y install wget
wget http://nginx.org/download/nginx-1.22.1.tar.gz

2、解包
tar -axf nginx-1.22.1.tar.gz

3、安装依赖包
yum install -y gcc pcre-devel zlib-devel

4、配置（检查安装环境、指定安装位置、安装的功能等）
cd nginx-1.22.1
./configure
如果无法判断是否成功，可以执行 echo  $?  (返回0成功)
5、编译
make
6、编译安装
make install
7、优化命令路径
vim /etc/profile
最后面添加：export PATH=$PATH:/usr/local/nginx/sbin
刷新生效：source /etc/profile

8、启动服务
nginx  #启动
nginx -s stop  #停止服务
nginx -s reload  #重载服务

9、测试访问
浏览器访问ip


#nginx与php集成（nginx可以相应php页面）
1、nginx完成安装
2、安装php与php-fpm
 yum -y install php php-fpm
php-fpm  #接口，是一个服务
systemctl start php-fpm  #启动服务
netstat -tunlp   #php-fpm监听端口是本机的9000

3、修改nginx配置文件，支持php页面
/usr/local/nginx/conf/nginx.conf
.........
        location / {
            root   html;
            index  index.php index.html index.htm;  #把index.php加在前面
        }
.......    # 去掉下面7行的注释（#）
        location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;  #修改这个一行的路径 /script--->$document_root
            include        fastcgi_params;
        }

nginx -t  #检查配置文件语法
nginx -s reload  #重载服务

4、编写测试页面
cd /usr/local/nginx/html
vim 1.php
<?php
phpinfo();
?>

5、测试访问
192.168.100.10/1.php  #出现php信息页面




[[wordpress博客平台部署]]















