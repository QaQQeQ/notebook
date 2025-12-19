docker：


### docker的安装部署
1、环境准备（静态ip、防火墙、网络、selinux、卸载旧版）
2、安装（安装源官方较慢，建议使用阿里源）
### step 1: 安装必要的一些系统工具
yum install -y yum-utils

### step 2: 添加软件源信息
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

### step 3: 安装Docker
 yum -y install docker-ce 

### step 4: 开启Docker服务
systemctl start docker

### docker info  #查看docker信息

----------------
扩展：安装指定版
### step1: 列出历史版本
yum list docker-ce.x86_64 --showduplicates | sort -r
### step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
#### sudo yum -y install docker-ce-[VERSION]

核心：镜像-----容器-----------仓库
#镜像的基本管理命令
注意：官方仓库已无法使用，需要配置镜像加速器
```
cat > /etc/docker/daemon.json <<EOF
{
 "registry-mirrors": [
                   "https://docker.1ms.run"
 ]
}
EOF
```
```
systemctl daemon-reload
systemctl restart docker
```


docker images  #查看
docker pull nginx:1.22  #从仓库拉取镜像（冒号后面是版本，如果不知道版本会下载最新版本）
docker save -o nginx.tar nginx:1.22 #镜像导出  镜像》文件
docker load -i nginx.tar  #镜像导入   文件》镜像
docker rmi 0f8498f13f3a （nginx:1.22）#删除镜像
docker tag nginx:1.22 mynginx:v1 #更改镜像标签（换名）
docker inspect nginx:1.22  #查看镜像详情

#容器管理命令（镜像的运行状态）
docker run -it -d --name web1 -p 80:80 nginx:1.22 #基于nginx镜像运行容器
docker run  [选项]  [--name 容器别名] [-p 本机端口:容器端口] 镜像名
常用选项：
-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上
 -i 让容器的标准输入保持打开
-d 后台运行
-p 端口映射  （需要访问容器里的服务）
-v 卷挂载        （容器中的数据需要持久化保存）
--name 指定容器名称
-e 向容器中传递参数

注意：一个容器如果要持续运行，必须有一个前台进程做支撑。

# 
docker ps    #查看运行状态的容器
docker ps -a   #查看所有的容器包括停止的容器
docker stop  容器名       #关闭容器
docker start  容器名      #启动容器
docker restart  容器名   #重启容器
docker rm  容器名         #删除容器
 -f #强制删除 包括运行中的容器
docker ps -aq | xargs docker rm -f   #删除所有的容器


docker exec -it 容器id bashshell    #进入到指定容器
例：docker exec -it web1 /bin/bash

docker commit 容器名 镜像名  （创建新镜像的一种方式）
例：docker commit nginx01 mynginx:1.2

docker inspect  容器名  # 查看容器详情
docker logs 容器名称  #查看容器启动时的日志、输出信息（用于排错）

-e 传递参数
docker run -it -d --name db2 -e MYSQL_ROOT_PASSWORD='123456'  mysql:5.7

-p 端口映射
-p：宿主机端口：容器端口

1）指定目录挂载： -v 宿主机路径：容器路径
例：docker run -d --name nginx03 -v /tmp/www:/usr/share/nginx/html/ -p 80:80 nginx
只需向宿主机/tmp/www目录放置网页文件，即可访问

2）具名挂载：-v 卷名：容器路径
例：docker run -d --name nginx03 -v www:/usr/share/nginx/html/ -p 80:80 nginx
默认的宿主机挂载目录：/var/lib/docker/volumes/

#自定义镜像
原始镜像-----》容器（进行修改）-----》提交成新镜像
Dockerfile  ---》容器（中间的临时容器）----》提交成新镜像

dockerfile关键字：
FROM             导入：镜像的基础镜像
LABEL             设置：镜像的标签
USER              设置：后续指令和容器的运行用户和组
ENV                设置：环境变量
WORKDIR       设置：后续指令中任何指令的当前目录
ADD                执行：文件复制，向镜像传输文件数据
COPY              执行：文件复制，向镜像传输文件数据
SHELL             设置：镜像的默认Shell程序
RUN                执行：Shell命令，会向镜像追加程序数据
EXPOSE           设置：镜像的可暴露端口
VOLUME          设置：镜像的挂载点   #类似之前的匿名挂载
CMD                设置：容器启动之后的初始执行命令
ENTRYPOINT   设置：容器启动之后的初始执行命令

构建命令：
docker build --rm    -t my_centos:7 -f ./Dockerfile . 
--rm #删除临时容器，默认选项可以不写
-t：指定镜像标签
-f：指定Dockerfile的文件名，果然不指定，文件名只能是Dockerfile
注意后面的.（点）

练习课件案例，熟悉关键字
构建nginx镜像
要求：
1、基于centos基础镜像，构建nginx镜像
2、nginx采用源码安装方式
3、更换nginx默认首页内容
参考示例：
```
FROM centos:7
USER 0:0
SHELL ["/bin/sh","-c"]
RUN yum install -y gcc pcre-devel zlib-devel make;\
    useradd -M -s /sbin/nologin nginx
WORKDIR /root
ADD http://nginx.org/download/nginx-1.19.7.tar.gz ./
RUN tar -axf nginx-1.19.7.tar.gz
WORKDIR /root/nginx-1.19.7
RUN ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx;\
    make && make install
COPY index.html /usr/local/nginx/html
EXPOSE 80
CMD ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]   #服务需要前台运行
```

或者改成：
ENTRYPOINT ["/usr/local/ngin/sbin/nginx"]
CMD ["-g","daemon off;"]
cmd的内容会追加到entrypoint后面


作业：
构建镜像：nginx与php集成
要求：以centos：7为基础镜像，nginx用源码安装，php、php-fpm可以
直接用yum安装，设置默认首页为index.php
```
<?php
phpinfo()
?>
```


```
FROM centos:7
USER 0:0
SHELL ["/bin/sh","-c"]
RUN yum install -y gcc pcre-devel zlib-devel make php php-fpm
RUN useradd -M -s /sbin/nologin nginx
WORKDIR /root
ADD http://nginx.org/download/nginx-1.19.7.tar.gz ./
RUN tar -axf nginx-1.19.7.tar.gz
WORKDIR /root/nginx-1.19.7
RUN ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx;\
    make && make install
COPY index.html /usr/local/nginx/html
EXPOSE 80
CMD ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]   #服务需要前台运行
```







[[k8s]]