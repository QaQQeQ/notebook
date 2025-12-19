ansible ：批量系统配置、批量程序部署、批量运行命令的自动化工具

环境：
server1 192.168.100.10   控制端、安装ansible
server2 192.168.100.20   被控端、无需安装任何软件
server3 192.168.100.30   被控端、无需安装任何软件

# 安装ansible
1）换源（官方源安装慢或者无法访问）
需要两个源（base、epel（扩展源））
   21  cd /etc/yum.repos.d/
   23  mkdir bak
   25  mv * bak
   27  curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
   30  yum -y install wget
   31  wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo

2）安装
    yum install -y ansible
    ansible --version  #查看版本

#使用前的准备工作
1）配置主机清单（你要管理谁）
vim /etc/ansible/hosts
[webservers]        #组名
192.168.100.20    #组成员，可以写域名支持通配符

[dbservers]
192.168.100.30


2）ssh配置（通讯方式、认证）
推荐使用免密登录
  ssh-keygen   #生成公钥和私钥
  ssh-copy-id 192.168.10.50    #推送公钥到被控端
  ssh-copy-id 192.168.10.60

3）测试与被控端通信
ansible all -m ping   # 返回成功（绿色）

#常用的模块
命令执行方式：两种（ad-hoc命令行方式、playbook剧本）
1）ad-hoc命令行方式
语法：ansible [-i 自定义清单文件] 那些主机 -m 模块名称 -a "模块参数"
模块用法查看帮助信息：
ansible-doc  模块名称   -s （概要信息）
ansible-doc -l  #列出所有的模块
命令执行结果颜色表示的含义：
1、绿色：成功（被控端无改变）
2、黄色：成功（被控端发生了改变）
3、紫色：成功（有警告信息，一般表示模块有代替方案）
4、红色：报错



1.setup模块  收集被控端facts设备信息  （各种硬件配置信息）
例：ansible  aaa -m setup

2.ping模块  测试连通性
ansible  aaa -m ping


3.user模块  用户管理
参数：
create_home：是否创建家目录
shell：指定登录的shell    /sbin/nologin 不可登录
name :指定用户名
state：状态（具体的操作：absent（删除）、present（创建））
remove：删除家目录 （state=absent时才可用）
例：
ansible aaa -m user -a 'state=present name=nginx create_home=no shell=/sbin/nologin'
创建用户nginx 没有家目录，不能登录系统

ansible aaa -m user -a 'state=absent name=user02 remove=yes'
删除用户user02，连同家目录

4.group模块  组管理
state：状态（具体的操作：absent（删除）、present（创建））
name :指定组名
例：
ansible aaa -m group -a 'state=present name=yjs'  #创建yjs组
ansible aaa -m group -a 'state=absent name=yjs'   #删除yjs组

5.file模块  文件管理
state：touch创建或更新文件   directory创建或更新目录   absent（删除） file（更新）
owner：文件所有者
group：属组
mode：权限
path：目标路径  touch /tmp/1.txt
例：
ansible aaa -m file -a 'state=touch path=/tmp/1.txt      #创文件
ansible aaa -m file -a 'state=directory path=/tmp/aaa'  #创目录
ansible aaa -m file -a 'state=absent path=/tmp/1.txt'    #删除文件或目录
ansible aaa -m file -a 'state=touch path=/tmp/2.txt mode=644 owner=user01 group=yjs'
#创建文件，指定所有者、属组、权限
如果指定文件或目录已经存在，则为修改其属性



#copy模块  （主控端--》被控端）
src：原文件路径
dest：目标文件路径
mode：指定权限
owner：指定复制后的文件的属主
content：替代src，也就是将一段内容作为原，将这个内容直接写入到目标文件中
例：
ansible web -m copy -a 'src=/tmp/22222.txt dest=/root/'   #目标是目录，直接复制到指定目录        
ansible web -m copy -a 'src=/tmp/22222.txt dest=/root/4.txt' #目录是文件名，复制并改名
ansible web -m copy -a 'content=123 dest=/root/5.txt' #不指定源文件，而是具体内容

ansible web -m copy -a 'src=/tmp/1.txt dest=/aaa/'  #复制到aaa目录下，aaa目录不存在会创建
ansible web -m copy -a 'src=/tmp/1.txt dest=/aaa'   #复制到根目录下，改名为aaa
ansible web -m copy -a 'src=/root/aaa dest=/tmp/'  #复制aaa目录
注意：复制目录的时候不能是空目录

被控端--》被控端  需要使用选项： remote_src=yes



# firewalld 管理被控端的firewalld防火墙
state: enabled(表示add添加规则)、disabled（表示remove删除规则） absent、present（表示删除添加区域）
port：指定端口 端口/协议
service：指定服务名称
permanent：是否永久生效  （默认临时生效）
immediate：是否立即生效
例：
ansible web -m firewalld -a 'zone=public port=80/tcp state=enabled immediate=yes permanent=yes' #添加规则



# selinux 管理被动端的SELinux策略和状态
state: enforcing强制、permissive非强制但警告、disabled禁用
例：
 ansible web -m selinux -a 'state=disabled' #禁用selinux


# yum模块
参数：
name：指定软件包名称
state：present安装（默认）  absent（删除）    latest  （更新到最新的）
ansible web -m yum -a 'name=tree' 安装时可以省略 state=present
ansible web -m yum -a 'name=tree state=absent'
安装多个软件包时：name=软件包1，软件包2....


# service模块与systemd模块  ：服务管理模块
name：指定服务的名称
enabled：指定服务是否开机自启动 ：yes（开机自启）no（禁止开机自启）
state：指定服务最后的状态 started stopped reloaded restarted
ansible web -m service -a 'name=httpd state=started'  #启动服务
ansible web -m systemd -a 'name=httpd state=started'  #启动服务



# command模块:在被控端执行单个命令  （默认模块）  不具备幂等性
ansible web(组名)  -m command  -a  '要执行的命令'                            
注意：不识别变量及特殊符号：比如管道符
ansible web -m command -a 'tail -2 /etc/passwd'

# shell模块：在被控端执行单个命令或多个命令    不具备幂等性
与command模块类似，但是支持更复杂的命令及特殊符号比如管道符
执行多条命令：  命令1；命令2...


#script模块
ansible web -m script -a /tmp/1.sh  （脚本的绝对路径）


#template模块：拷贝模板文件到被控端 (一般用来拷贝配置文件)
src：源文件（一般是模板文件 *.j2）
dest:目标位置
例：
ansible web -m template -a 'src=/tmp/httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf'
copy模块无法识别文件内容中的变量，template可以。


#unarchive模块
src：源文件（压缩包）
dest：目标路径 （解压后）
ansible web -m unarchive -a 'src=/tmp/nginx-1.19.1.tar.gz dest=/usr/local/'

#get_url 模块 - 下载文件
常用参数：
url：下载地址（必填）
dest：保存路径（必填）
ansible all -m get_url -a "url=http://nginx.org/download/nginx-1.22.1.tar.gz dest=/root"

#ansible剧本  （类似shell脚本，把所有要执行的任务放到一个文件中）
使用语言：YAML
语法注意事项：
1.区分大小写
2.非常严格的层级关系（相同层级必须左对齐，只能使用空格缩进）
3.- hosts: web  （关键字短横线后和冒号后面需要有空格）

常用关键字：
hosts：指定主机或主机组（作用对象）
remote_user：使用哪个身份去执行任务
tasks：任务列表
           一个任务列表通常由多个name（名称）和具体执行的模块组成
vars：定义变量
          vars: 
            变量名: 变量的值
        引用变量： {{ 变量名 }}
notify：通知（当前任务发生改变时触发）
handlers：匹配对应的notify（具体的操作）（多用于服务管理）
例：
[root@localhost tmp]# cat 1.yaml 
---
- name: test playbook
  hosts: webservers
  vars:
    appname: httpd
  tasks:
    - name: 安装httpd软件
      yum: name={{ appname }} state=present
    - name: 启动httpd服务
      systemd: name={{ appname }} state=started
    - name: 分发配置文件
      template: src=/tmp/httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf
      notify:
        - restart web
    - name: 分发首页文件
      template: src=/tmp/index.html dest=/var/www/html/index.html
  handlers:
    - name: restart web
      systemd: name={{ appname}} state=restarted
注意：模版文件和首页文件要自己准备





[[docker基本管理]]