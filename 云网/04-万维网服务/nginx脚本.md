### 基础版
```sh
#!/bin/bash

rpm -e httpd --nodeps
which wget || yum install -y wget

wget -c http://nginx.org/download/nginx-1.23.2.tar.gz
tar -axf nginx-1.23.2.tar.gz

cd nginx-1.23.2

id nginx || useradd -r -s /sbin/nologin nginx
yum install -y gcc pcre-devel zlib-devel openssl-devel
make clean

./configure --prefix=/usr/local/nginx --user=nginx --group=nginx \
--with-http_ssl_module \
--with-http_mp4_module \
--with-http_flv_module \
--with-http_dav_module

make -j $(nproc) &> /dev/null
make install &> /dev/null

cd
cat > /etc/profile.d/nginx.sh <<EOF
export PATH="/usr/local/nginx/sbin:\$PATH"
EOF
source /etc/profile.d/nginx.sh
nginx -V

firewall-cmd --permanent --zone=public --add-port=80/tcp --add-port=443/tcp
firewall-cmd --reload

if ! grep -i "nginx" /etc/rc.d/rc.local; then
    echo "/usr/local/nginx/sbin/nginx" >> /etc/rc.d/rc.local    ## 设置: 开机自启动
chmod +x /etc/rc.d/rc.local
fi

nginx
netstat -tunlp

```

### 完整版
```sh
#!/bin/bash

rpm -e httpd --nodeps
which wget || yum install -y wget

wget -c http://nginx.org/download/nginx-1.23.2.tar.gz
tar -axf nginx-1.23.2.tar.gz

cd nginx-1.23.2

id nginx || useradd -r -s /sbin/nologin nginx
yum install -y gcc pcre-devel zlib-devel openssl-devel
make clean

./configure --prefix=/usr/local/nginx --user=nginx --group=nginx \
--with-http_ssl_module \
--with-http_mp4_module \
--with-http_flv_module \
--with-http_dav_module

make -j $(nproc) &> /dev/null
make install &> /dev/null

cd

openssl req -new -x509 -keyout /usr/local/nginx/conf/cert.key -out /usr/local/nginx/conf/cert.pem -nodes -batch
chown nginx:nginx /usr/local/nginx/conf/cert.key
chown nginx:nginx /usr/local/nginx/conf/cert.pem
chmod 400 /usr/local/nginx/conf/cert.key
test -f /usr/local/nginx/conf/nginx.conf.bak || \
	cp /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.bak

cat > /usr/local/nginx/conf/nginx.conf <<EOF
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    # HTTP server
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    # HTTPS server
    server {
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate      cert.pem;
        ssl_certificate_key  cert.key;
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
        ssl_ciphers          HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
EOF


cd
cat > /etc/profile.d/nginx.sh <<EOF
export PATH="/usr/local/nginx/sbin:\$PATH"
EOF
source /etc/profile.d/nginx.sh
nginx -V

firewall-cmd --permanent --zone=public --add-port=80/tcp --add-port=443/tcp
firewall-cmd --reload

if ! grep -i "nginx" /etc/rc.d/rc.local; then
    echo "/usr/local/nginx/sbin/nginx" >> /etc/rc.d/rc.local    
	chmod +x /etc/rc.d/rc.local
fi

nginx
netstat -tunlp

```