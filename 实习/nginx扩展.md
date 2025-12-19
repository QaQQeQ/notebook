nginx高级功能：反向代理与负载均衡

#反向代理配置
环境：
server1  192.168.100.10    安装nginx（代理服务器）
server2  192.168.100.20    安装httpd（真实网站服务器）
1、基本环境准备
2、安装nginx
3、修改配置文件（先回复到默认配置文件）
.......
        location / {                               #访问网站的任何路径
           # root   html;                        #nginx上面不需要网页文件
           # index  index.html index.htm;
           proxy_pass http://192.168.100.20;      #把请求转移给真实服务器
        }
.................
重启服务器


4、准备后端服务器做测试
yum -y install httpd
systemctl start httpd
echo "this is 100.20" > /var/www/html/index.html

测试：浏览器访问nginx的ip，页面出现是httpd的页面



#负载均衡
环境：
server1  192.168.100.10    安装nginx（代理服务器）
server2  192.168.100.20    安装httpd（真实网站服务器1）
server3  192.168.100.30    安装httpd（真实网站服务器2）

1、基本环境准备
2、安装nginx
3、修改配置文件（先回复到默认配置文件）
```
.......
upstream myservers {                             #定义一个服务器池
        server 192.168.100.20 weight=3;
        server 192.168.100.30;
}
  #添加以上upstream配置信息，注意添加位置  
    server {
        listen       80;
        server_name  localhost;

        location / {                               #访问网站的任何路径
           # root   html;                        #nginx上面不需要网页文件
           # index  index.html index.htm;
           proxy_pass http://myservers;      #把请求转移给之前定义的服务器池，名称一定要一直
        }
.................
```
重启服务器


4、准备后端服务器做测试
两台服务器安装httpd，页面不同为了测试效果
测试：浏览器访问nginx的ip，页面出现是httpd的页面，刷新会自动跳转到另一个httpd页面。






[[shell基础]]