## 要求：使用ansible部署
1. MySQL双主复制(2台)
2. nginx和zabbix前端
3. DNS轮询两台数据库服务器
4. 2和3公用1台服务器
5. zabbix Server
6. zabbix agent 客户端-自动注册

## 前提环境准备
#### 主机分配

| hostname |          域名           |       ip       | password |        role        | ansible role |
| :------: | :-------------------: | :------------: | :------: | :----------------: | :----------: |
| Rocky9-1 |    mysql.test.com     | 192.168.10.150 | a123456! |       mysql        |              |
| Rocky9-2 |    mysql.test.com     | 192.168.10.160 | a123456! |       mysql        |              |
| Rocky9-3 |    dns01.test.com     | 192.168.10.170 | a123456! | nginx、zabbix前端,DNS |     主控端      |
| Rocky9-4 | zabbixServer.test.com | 192.168.10.180 | a123456! |   zabbix server    |              |
| Rocky9-5 | zabbixAgent.test.com  | 192.168.10.190 | a123456! |    zabbix agent    |              |
#### selinux默认已关闭
#### ansible环境准备
```bash
# 针对主控端
dnf install -y https://mirrors.aliyun.com/epel/epel-release-latest-9.noarch.rpm
sed -i 's|^#baseurl=https://download.example/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
sed -i 's/enabled=1/enabled=0/g' /etc/yum.repos.d/epel-cisco-openh264.repo
dnf clean all
dnf makecache


dnf install -y ansible-core ansible
ansible --version

cat > /etc/profile.d/ansible.sh <<EOF
export ANSIBLE_HOST_KEY_CHECKING=False
EOF
source /etc/profile.d/ansible.sh



# 针对被控端（仍在主控端操作）
cat > linuxHosts.list <<EOF
[linuxHosts]
Rocky9-1 ansible_ssh_host=192.168.10.150 ansible_ssh_user=root ansible_ssh_pass='a123456!'
Rocky9-2 ansible_ssh_host=192.168.10.160 ansible_ssh_user=root ansible_ssh_pass='a123456!'
Rocky9-3 ansible_ssh_host=192.168.10.170 ansible_ssh_user=root ansible_ssh_pass='a123456!'
Rocky9-4 ansible_ssh_host=192.168.10.180 ansible_ssh_user=root ansible_ssh_pass='a123456!'
Rocky9-5 ansible_ssh_host=192.168.10.190 ansible_ssh_user=root ansible_ssh_pass='a123456!'
EOF
ansible all -i linuxHosts.list -m raw -a 'dnf install -y python3 python3-dnf python3-libselinux'

```
#### 分发公钥
剧本文件为1.yaml
```yaml
---
- hosts: localhost
  connection: local
  tasks:
  - name: 安装
    shell: dnf install -y python3-pexpect
  - name: 创建登录用户的SSH密钥对
    no_log: true
    ignore_errors: yes
    expect:
      command: ssh-keygen -t rsa -b 4096
      timeout: 10
      responses:
        '.*Overwirte.*': "n\n"
        '.*Enter file in which to save the key.*': "\n"
        '.*Enter passphrase.*':  "\n"
        '.*Enter same passphrase.*':  "\n"
- hosts: linuxHosts
  tasks:
  - name: 确保。。。
    file: 
      path: "~/.ssh"
      mode: "0700"
      state: directory
  - name: 确保。。。
    file:
      path: "~/.ssh/authorized_keys"
      mode: "0600"
      state: touch
- hosts: localhost
  connection: local
  tasks:
  - name: 传递公钥
    no_log: true
    ignore_errors: yes
    expect:
      command: ssh-copy-id {{ hostvars[item].ansible_ssh_user }}@{{ hostvars[item].ansible_ssh_host }}
      timeout: 10
      responses:
        'password:': "{{ hostvars[item].ansible_ssh_pass }}\n"
    loop: "{{ groups['linuxHosts'] }}"


```
ansible-playbook -i linuxHosts.list 1.yaml
## 一、MySQL双主复制
### 目录结构

```
[root@Rocky9-3 tmp]# tree
.
├── install_two_mysql
│   ├── hosts.list
│   ├── install_two_mysql.yaml
│   └── templates
│       └── master_and_slave.cnf.j2
```
### 编辑文件

#### vi /tmp/install_two_mysql/hosts.list
```
server01 ansible_ssh_host=192.168.10.150 ansible_ssh_user=root ansible_ssh_pass=a123456! server_id=1 auto_increment_offset=1

server02 ansible_ssh_host=192.168.10.160 ansible_ssh_user=root ansible_ssh_pass=a123456! server_id=2 auto_increment_offset=2

[groupA]
server01
[groupB]
server02

```

#### vi /tmp/install_two_mysql/templates/master_and_slave.cnf.j2
```j2
[mysqld]
######################
### 设置：主机 ID
######################
server_id = {{ server_id }}

######################
### 配置 Master 角色功能
######################

## 开启 binlog 日志
log_bin = /var/lib/mysql/binlog
log_bin_index = /var/lib/mysql/binlog.index
binlog_format = MIXED

## 设置 binlog 日志数据的落盘策略
sync_binlog=1

## 防止主键冲突
auto_increment_offset={{ auto_increment_offset }}
auto_increment_increment=2

## OFF 禁止：将 Relay log 中继日志的重放操作写入 Binlog 日志，从而避免造成“无用的主从复制”
log-slave-updates = OFF

## 确保：可写入
read_only=0
super_read_only=0

######################
### 配置 Slave 角色功能
######################

## 设置 relaylog 中继日志文件（默认已设置）
## 设置 relaylog 中继日志的落盘策略
sync_relay_log = 1

######################
### 其他可能的通用配置
######################

## 设置 redo 日志数据的落盘策略
innodb_flush_log_at_trx_commit=1

## 采用可兼容的密码策略
default_authentication_plugin=mysql_native_password

## 禁止：反向域名解析
skip-name-resolve

```
#### vi /tmp/install_two_mysql/install_two_mysql.yaml
```yaml
---
- hosts: all
  tags: 安装及初始化MySQL(或重置密码)
  vars_prompt:
    - name: mysql_root_password
      prompt: "请设置或更新 MySQL root 密码"
      private: true
#      confirm: yes

  tasks:
  - name: 检查 MySQL 是否已初始化
    stat:
      path: /var/lib/mysql/mysql.sock
    register: mysql_sock

  - name: 安装 MySQL
    dnf:
      name:
        - mysql
        - mysql-server
        - python3-PyMySQL
      state: present
    when: not mysql_sock.stat.exists

  - name: 开机自启 MySQL 服务
    systemd:
      name: mysqld
      state: started
      enabled: true

  - name: 开放防火墙端口
    shell: |
      firewall-cmd --permanent --zone=public --add-port=3306/tcp
      firewall-cmd --reload
    ignore_errors: true

  - name: 等待 MySQL socket 可用
    wait_for:
      path: /var/lib/mysql/mysql.sock
      state: present
      timeout: 30

# -----------------------------
#  尝试用 socket 登录并修改密码
# -----------------------------
  - name: 尝试正常方式修改 root 密码
    mysql_user:
      login_unix_socket: /var/lib/mysql/mysql.sock
      login_user: root
      name: root
      host: localhost
      password: "{{ mysql_root_password }}"
      update_password: always
      state: present
    ignore_errors: true
    register: reset_try

  - name: 设置root远程登录权限
    mysql_user:
      login_host: localhost
      login_port: 3306
      login_user: root
      login_password: "{{ mysql_root_password }}"
      name: root
      host: '%'
      password: "{{ mysql_root_password }}"
      priv: '*.*:all,GRANT'
      update_password: always
      state: present
    when: reset_try is succeeded

# ------------------------------------------------------------
#   若无法正常登录（说明旧密码未知）则强制重置密码流程
# ------------------------------------------------------------
  - name: 强制重置 MySQL root 密码（跳过验证）
    block:
      - name: 停止 MySQL 服务
        service:
          name: mysqld
          state: stopped

      - name: 以 skip-grant-tables 模式启动 MySQL
        command: nohup mysqld_safe --skip-grant-tables &
        async: 10
        poll: 0

      - name: 等待 MySQL 启动
        wait_for:
          port: 3306
          delay: 5
          timeout: 30

      - name: 强制修改 root 密码（免验证）
        shell: >
          mysql -uroot --socket=/var/lib/mysql/mysql.sock
          -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '{{ mysql_root_password }}';
              FLUSH PRIVILEGES;"

      - name: 杀掉临时 mysqld_safe
        command: pkill -f 'mysqld_safe'
        ignore_errors: true

      - name: 正常启动 MySQL 服务
        service:
          name: mysqld
          state: started

      - name: 设置root远程登录权限
        mysql_user:
          login_host: localhost
          login_port: 3306
          login_user: root
          login_password: "{{ mysql_root_password }}"
          name: root
          host: '%'
          password: "{{ mysql_root_password }}"
          priv: '*.*:all,GRANT'
          update_password: always
          state: present
    when: reset_try is failed

  - name: 提示操作结果
    debug:
      msg: >
        {{ '检测到已安装 MySQL，已更新 root 密码（或强制重置）' if mysql_sock.stat.exists
        else '首次安装并初始化 MySQL，已设置 root 密码' }}

# ------------------------------------------------------------------------------------------
- hosts: all
  tags: 双主配置
  vars_prompt:
    - name: mysql_root_password
      prompt: "请输入 MySQL root 密码（用于双主登录）"
      private: yes
    - name: mysql_copy_user
      prompt: "请设置用于双主复制的用户名"
      private: no
      default: repl
    - name: mysql_copy_password
      prompt: "请设置用于双主复制的用户密码"
      private: yes

  tasks:
  - name: 导入模板文件
    template:
      src: "master_and_slave.cnf.j2"
      dest: /etc/my.cnf.d/master_and_slave.cnf

  - name: 创建用于双主复制的用户并授权
    mysql_user:
      login_host: localhost
      login_port: 3306
      login_user: root
      login_password: "{{ mysql_root_password }}"
      name: "{{ mysql_copy_user }}"
      host: '%'
      password: "{{ mysql_copy_password }}"
      priv: '*.*:REPLICATION SLAVE'
      update_password: on_create
      state: present
  
  - name: restart_mysqld
    service:
      name: mysqld
      state: restarted

# ------------------------------------------------------------
  - name: B节点
    block:
      - name: 重置A库
        mysql_query:
          login_user: root
          login_password: "{{ mysql_root_password }}"
          login_host: "{{ hostvars[groups['groupA'][0]].ansible_ssh_host }}"
          query: "reset master;"
      - name: 导出全库数据
        shell: |
               mysqldump -uroot -p"{{ mysql_root_password }}" -h "{{ hostvars[groups['groupA'][0]].ansible_ssh_host }}" \
                     --add-locks \
                     --all-databases \
                     --source-data=1 \
                     --flush-privileges > ~/mysql_master_full_backup.sql
        args:
          creates: ~/mysql_master_full_backup.sql
      - name: 停止复制
        mysql_replication:
          mode: stopreplica
          login_user: root
          login_password: "{{ mysql_root_password }}"
      - name: 重置复制状态
        mysql_query:
          login_user: root
          login_password: "{{ mysql_root_password }}"
          query: "RESET SLAVE ALL;"
      - name: 导入
        shell: |
               mysql -uroot -p"{{ mysql_root_password }}" < ~/mysql_master_full_backup.sql
        args:
          removes: ~/mysql_master_full_backup.sql
      - name: 配置
        mysql_replication:
          mode: changeprimary
          login_user: root
          login_password: "{{ mysql_root_password }}"
          primary_host: "{{ hostvars[groups['groupA'][0]].ansible_ssh_host }}"
          primary_port: 3306
          primary_user: "{{ mysql_copy_user }}"
          primary_password: "{{ mysql_copy_password }}"
          primary_log_file: "binlog.000001"
          primary_log_pos: 0
      - name: 启动复制线程
        mysql_replication:
          mode: startreplica
          login_user: root
          login_password: "{{ mysql_root_password }}"
      - name: 查看B节点的复制状态
        shell: mysql -uroot -p"{{ mysql_root_password }}" -e "show slave status \\G;"
        register: slave_status_output
        changed_when: false
      - debug:
          var: slave_status_output.stdout
    when: inventory_hostname in groups['groupB']

# ------------------------------------------------------------
  - name: A节点
    block:
      - name: 获取 binlog 信息
        mysql_query:
          login_user: root
          login_password: "{{ mysql_root_password }}"
          login_host: "{{ hostvars[groups['groupB'][0]].ansible_ssh_host }}"
          query: "show master status;"
        register: b_master_status
        changed_when: false
      - debug: var=b_master_status
      - set_fact:
          b_log_file: "{{ b_master_status.query_result[0][0].File }}"
          b_log_pos: "{{ b_master_status.query_result[0][0].Position }}"
      - name: 重置复制信息
        mysql_query:
          login_user: root
          login_password: "{{ mysql_root_password }}"
          query: "RESET SLAVE ALL;"
      - name: 停止复制线程
        mysql_replication:
          mode: stopreplica
          login_user: root
          login_password: "{{ mysql_root_password }}"
      - name: 配置双主复制
        mysql_replication:
          mode: changeprimary
          login_user: root
          login_password: "{{ mysql_root_password }}"
          primary_host: "{{ hostvars[groups['groupB'][0]].ansible_ssh_host }}"
          primary_port: 3306
          primary_user: "{{ mysql_copy_user }}"
          primary_password: "{{ mysql_copy_password }}"
          primary_log_file: "{{ b_log_file }}"
          primary_log_pos: "{{ b_log_pos }}"
      - name: 启动复制
        mysql_replication:
          mode: startreplica
          login_user: root
          login_password: "{{ mysql_root_password }}"
      - name: 查看A节点的复制状态
        shell: mysql -uroot -p"{{ mysql_root_password }}" -e "show slave status \\G;"
        register: slave_status_output
        changed_when: false
      - debug:
          var: slave_status_output.stdout
    when: inventory_hostname in groups['groupA']

```
## 二、DNS
### 目录结构
```
[root@Rocky9-3 tmp]# tree
.
├── install_DNS
│   ├── hosts.list
│   ├── install_dns.yaml
│   └── templates
│       ├── named.conf.j2
│       ├── test.com.zone.conf.j2
│       └── test.com.zone.j2
```
### 编辑文件

#### vi /tmp/install_DNS/hosts.list
```
dns01 ansible_ssh_host=192.168.10.170 ansible_ssh_user=root ansible_ssh_pass=a123456! role=master
mysql01 ansible_ssh_host=192.168.10.150 ansible_ssh_user=root ansible_ssh_pass=a123456! role=mysql
mysql02 ansible_ssh_host=192.168.10.160 ansible_ssh_user=root ansible_ssh_pass=a123456! role=mysql
zabbix_server ansible_ssh_host=192.168.10.180 ansible_ssh_user=root ansible_ssh_pass=a123456! role=zabbixServer
zabbix_agent ansible_ssh_host=192.168.10.190 ansible_ssh_user=root ansible_ssh_pass=a123456! role=zabbixAgent
[dns]
dns01
[all:vars]
dns_name=dns01
dns_ip="{{ hostvars[dns_name]['ansible_ssh_host'] }}"
```
#### vi /tmp/install_DNS/templates/named.conf.j2
```j2
options {
        listen-on port 53 { any;};
        listen-on-v6 port 53 { any;};
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query { any;};
        recursion yes;
        forward first;
        forwarders { 114.114.114.114; };
#       dnssec-enable no;    #Rocky9已弃用
        dnssec-validation no;
        bindkeys-file "/etc/named.root.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};
logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};
zone "." IN {
        type hint;
        file "named.ca";
};
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
#### vi /tmp/install_DNS/templates/test.com.zone.conf.j2
```j2
zone "{{ dnsDomainName }}" IN {
     type master;                      
     file "{{ dnsDomainName }}.zone";  
     notify yes;                      
     allow-update { none; };          
};
```
#### vi /tmp/install_DNS/templates/test.com.zone.j2
```j2
$TTL 3H
@       IN SOA  {{ inventory_hostname_short }}.{{ dnsDomainName }}. rname.invalid. (
                                              0       ; serial
                                              1D      ; refresh
                                              1H      ; retry
                                              1W      ; expire
                                              3H )    ; minimum

@       NS      {{ inventory_hostname_short }}.{{ dnsDomainName }}.
{{ inventory_hostname_short }}  A       {{ hostvars[inventory_hostname_short].ansible_ssh_host }}
```
#### vi /tmp/install_DNS/install_dns.yaml
```yaml
---
- hosts: dns
  tags: 安装服务
  tasks:
  - yum: name=bind,bind-utils state=present
  - shell: |
           nmcli connection modify ens33 ipv4.dns "127.0.0.1"
           nmcli connection up ens33
  - firewalld: zone=public port="{{ item }}" permanent=yes immediate=yes state=enabled
    with_items:
    - "53/tcp"
    - "53/udp"
  - systemd: name=named state=started enabled=yes
    ignore_errors: yes
  - template:
      src: "named.conf.j2"
      dest: /etc/named.conf
    notify:
    - restart_named
  handlers:
  - name: restart_named
    systemd: name=named state=restarted
#-------------------------------------------------------------------------------
- hosts: all
  tags: 配置
  vars_prompt:
  # ▼ 指定：<权威注册名称区域>的<自定义DNS域名>
  - name: dnsDomainName
    prompt: 请输入需创建的<权威注册名称区域>的<DNS域名>（格式为：xxx.yyy.zzz）？
    private: no
    default: test.com
    confirm: yes
  tasks:
  # 模板导入和配置
  - block:
    - template:
        src: "test.com.zone.conf.j2"
        dest: "/etc/{{ dnsDomainName }}.zone.conf"
    - template:
        src: "test.com.zone.j2"
        dest: "/var/named/{{ dnsDomainName }}.zone"
    - shell: |
             if ! grep -E "^\s*include\s+\"/etc/{{ dnsDomainName }}\.zone\.conf\";" /etc/named.conf &> /dev/null; then
                echo 'include "/etc/{{ dnsDomainName }}.zone.conf";' >> /etc/named.conf
             fi
             chown named:named /var/named/"{{ dnsDomainName }}.zone"
    when: hostvars[inventory_hostname]['role'] == 'master'
#-------------------------------------------------------------------------------
  - block:
    - name: 配置客户端DNS指向
      shell: |
             nmcli connection modify ens33 ipv4.dns "{{ dns_ip }}"
             nmcli connection up ens33
    - name: 转到DNS主机添加A记录
      shell: |
        if ! grep -E "{{ inventory_hostname }}\s+A" /var/named/{{ dnsDomainName }}.zone; then
          echo "{{ hostvars[inventory_hostname]['role'] }} A {{  hostvars[inventory_hostname]['ansible_ssh_host'] }}" >> /var/named/{{ dnsDomainName }}.zone
        fi
        systemctl restart named
      delegate_to: "{{ dns_name }}"
      delegate_facts: yes
    when: hostvars[inventory_hostname]['role'] != 'master'
```
## 三、nginx、zabbix前端（包括创建数据库用户和数据库）
### 目录结构
```
[root@Rocky9-3 tmp]# tree
.
├── install_web
│   ├── hosts.list
│   └── install_web.yaml
```
### 编辑文件
#### vi /tmp/hosts.list
```
web ansible_ssh_host=192.168.10.170 ansible_ssh_user=root ansible_ssh_pass=a123456!
mysql01 ansible_ssh_host=192.168.10.150 ansible_ssh_user=root ansible_ssh_pass=a123456!
mysql02 ansible_ssh_host=192.168.10.160 ansible_ssh_user=root ansible_ssh_pass=a123456!
[mysql]
mysql01
mysql02
```
#### vi /tmp/install_web.yaml
```
---
- hosts: mysql
  vars_prompt:
  - name: rootPassword
    prompt: 请输入MySQL的root密码
    private: no
  - name: zabbixPassword
    prompt: 请设置zabbix用户的密码
    default: a123456!
  tasks:
  - name: 登录到MySQL，创建数据库zabbix
    mysql_db:
      login_host: localhost
      login_port: 3306
      login_user: root
      login_password: "{{ rootPassword }}"
      name: zabbix
      state: present
      encoding: utf8mb4
      collation: utf8mb4_bin
    run_once: true
    
  - name: 创建zabbix用户并授权
    block:
    - mysql_user:
        login_host: localhost
        login_port: 3306
        login_user: root
        login_password: "{{ rootPassword }}"
        name: zabbix
        host: 'localhost'
        password: "{{ zabbixPassword }}"
        priv: 'zabbix.*:all,GRANT'
      run_once: true

    - mysql_user:
        login_host: localhost
        login_port: 3306
        login_user: root
        login_password: "{{ rootPassword }}"
        name: zabbix
        host: '%'
        password: "{{ zabbixPassword }}"
        priv: 'zabbix.*:all,GRANT'
      run_once: true

  
  - mysql_variables:
      login_host: localhost
      login_user: root
      login_password: "{{ rootPassword }}"
      variable: log_bin_trust_function_creators
      value: 1
      mode: global

# -------------------------------------------------------------------------
- hosts: web
  tasks:
  - name: 在 EPEL 源中禁用 zabbix 软件包
    lineinfile:
      path: /etc/yum.repos.d/epel.repo
      insertafter: '^enabled='
      regexp: '^excludepkgs='
      line: 'excludepkgs=zabbix*'
      firstmatch: yes
  - name: 获取zabbix的repo源配置文件
    shell: |
           rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/9/x86_64/zabbix-release-6.0-4.el9.noarch.rpm
           yum clean all
           yum makecache
  - name: 安装nginx web前端
    dnf: 
      name: 
        - zabbix-web-mysql
        - zabbix-nginx-conf
  
  - name: 
    lineinfile:
      path: /etc/nginx/conf.d/zabbix.conf
      regexp: '^(#|\s*)\s*listen'
      line: '        listen        80;'
      firstmatch: yes
  - name: 
    lineinfile:
      path: /etc/nginx/conf.d/zabbix.conf
      regexp: '^(#|\s*)\s*server_name'
      line: '        server_name        zabbix.test.com;'
      firstmatch: yes
  
  - name:
    lineinfile:
      path: /etc/php-fpm.d/zabbix.conf
      regexp: '^user'
      line: 'user = nginx'
  - name:
    lineinfile:
      path: /etc/php-fpm.d/zabbix.conf
      regexp: '^group'
      line: 'group = nginx'
  - name: 添加 PHP 时区配置
    lineinfile:
      path: /etc/php-fpm.d/zabbix.conf
      line: 'php_value[date.timezone] = Asia/Shanghai'
      insertafter: EOF
  
  - shell: setfacl -m u:nginx:rwx /etc/zabbix/web/
  - name: 
    systemd:
      name: "{{ item }}"
      enabled: yes
    loop:
      - nginx
      - php-fpm
  - name: 开启服务
    systemd:
      name: "{{ item }}"
      state: restarted
    loop:
      - nginx
      - php-fpm
  - name: 开放防火墙
    shell: |
           firewall-cmd --permanent --zone=public --add-port=80/tcp
           firewall-cmd --reload
  
```
## 四、zabbix server
### 目录结构
```
[root@Rocky9-3 tmp]# tree
.
├── install_zabbix_server
│   ├── hosts.list
│   └── install_zabbix_server.yaml
```
### 编辑文件
#### vi /tmpinstall_zabbix_server/hosts.list
```
zabbixServer ansible_ssh_host=192.168.10.180 ansible_ssh_user=root ansible_ssh_pass=a123456!
[mysql]
mysql1 ansible_ssh_host=192.168.10.150
mysql2 ansible_ssh_host=192.168.10.160
```
#### vi /tmpinstall_zabbix_server/install_zabbix_server.yaml
```yaml
---
- hosts: zabbixServer
  vars:
    db_host: "{{ hostvars[groups['mysql'][1]].ansible_ssh_host }}"
  vars_prompt:
  - name: zabbixPassword
    prompt: 请输入MySQL的zabbix用户密码
    default: a123456!
    private: no
  - name: rootPassword
    prompt: 请输入MySQL的root用户密码
    default: a123456!
    private: no
 
  tasks:
  - name: 获取repo源配置文件
    shell: |
           rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/9/x86_64/zabbix-release-6.0-4.el9.noarch.rpm
           yum clean all
           yum makecache
  - name: 安装server核心服务和agent客户端代理
    dnf:
      name: 
        - zabbix-server-mysql
        - zabbix-sql-scripts
        - zabbix-selinux-policy
        - zabbix-agent
        - mysql
      state: present
  
  - name: 导入初始架构和数据
    shell: |
           zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
             mysql --default-character-set=utf8mb4 -uzabbix -p"{{ zabbixPassword }}" -h {{ db_host }} zabbix
# ----------------------------------------------------------------------------------
  - name: 禁用log_bin_trust_function_creators
    tags: 禁用
    delegate_to: "{{ item }}"
    loop: "{{ groups['mysql'] }}"
    mysql_variables:
      login_user: root
      login_password: "{{ rootPassword }}"
      login_host: "{{ hostvars[item].ansible_ssh_host }}"
      variable: log_bin_trust_function_creators
      value: 0
      mode: global  
# ----------------------------------------------------------------------------------
  - name: 添加 Zabbix Server 的 DB 连接配置到文件末尾（仅当不存在）
    tags: 配置
    blockinfile:
      path: /etc/zabbix/zabbix_server.conf
      marker: "# {mark} ANSIBLE MANAGED ZABBIX DB CONFIG"
      block: |
        DBHost={{ db_host }}
        DBPassword={{ zabbixPassword }}
        DBPort=3306
      insertafter: EOF
  
  - name: 启用 zabbix-server 和 zabbix-agent 开机自启
    tags: 配置2
    service:
      name: "{{ item }}"
      enabled: yes
    loop:
      - zabbix-server
      - zabbix-agent

  - name: 重启 zabbix-server 和 zabbix-agent
    tags: 配置3
    service:
      name: "{{ item }}"
      state: restarted
    loop:
      - zabbix-server
      - zabbix-agent

  - name: firewalld 放行端口 (10050/10051)
    tags: 配置4
    shell: |
      firewall-cmd --permanent --zone=public --add-port=10050/tcp
      firewall-cmd --permanent --zone=public --add-port=10051/tcp
      firewall-cmd --reload

```
### 注意
- 在运行玩剧本完成网页端的引导之后，手动修改前端配置文件vi /etc/zabbix/web/zabbix.conf.php
```
# 明确指定serverip地址和端口
$ZBX_SERVER                     = '192.168.10.180';
$ZBX_SERVER_PORT                = '10051';

$ZBX_SERVER_NAME                = 'Zabbix Server';
```
## 五、zabbix agent
### 目录结构
```
[root@Rocky9-3 tmp]# tree
.
├── install_agent
│   ├── hosts.list
│   └── install_agent.yaml
```
### 编辑文件
#### vi /tmp/install_agent/hosts.list
```
zabbixAgent ansible_ssh_host=192.168.10.190 ansible_ssh_user=root ansible_ssh_pass=a123456!
[all:vars]
zabbixServer_ip=192.168.10.180
```
#### /tmp/install_agent/install_agent.yaml
```
---
- hosts: zabbixAgent
  tasks:
  - name: 获取repo源配置文件
    shell: |
           rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/9/x86_64/zabbix-release-6.0-4.el9.noarch.rpm
           yum clean all
           yum makecache
  - name: 安装agent客户端代理
    dnf:
      name: zabbix-agent
      state: present
  
  - name: 1
    lineinfile:
      path: /etc/zabbix/zabbix_agentd.conf
      regexp: '^#?Timeout='
      line: 'Timeout=30'
      state: present
      insertafter: EOF
  - name: 2
    lineinfile:
      path: /etc/zabbix/zabbix_agentd.conf
      regexp: '^#?ListenPort='
      line: 'ListenPort=10050'
      state: present
      insertafter: EOF
  - name: 3
    lineinfile:
      path: /etc/zabbix/zabbix_agentd.conf
      regexp: '^#?Server='
      line: 'Server={{ zabbixServer_ip }}'
      state: present
      insertafter: EOF
  - name: 4
    lineinfile:
      path: /etc/zabbix/zabbix_agentd.conf
      regexp: '^#?ServerActive='
      line: 'ServerActive={{ zabbixServer_ip }}:10051'
      state: present
      insertafter: EOF
  - name: 5
    lineinfile:
      path: /etc/zabbix/zabbix_agentd.conf
      regexp: '^(#?Hostname=)'
      line: '#\1'
      state: present
      insertafter: EOF
  
  - name: 启用 zabbix-agent 开机自启
    tags: 配置2
    service:
      name: zabbix-agent
      enabled: yes

  - name: 重启 zabbix-agent
    tags: 配置3
    service:
      name: zabbix-agent
      state: restarted

  - name: firewalld 放行端口 (10050)
    tags: 配置4
    shell: |
      firewall-cmd --permanent --zone=public --add-port=10050/tcp
      firewall-cmd --reload

  - name: 配置自动注册
    shell: |
           cat > /etc/zabbix/zabbix_agentd.d/autoActiveRegistration.conf <<EOF
           ServerActive={{ zabbixServer_ip }}:10051
           HostnameItem=system.hostname
           HostMetadata=ZabbixAgent1 on Linux
           EOF
           systemctl restart zabbix-agent.service
```
#### web页面手动创建自动注册动作
![](assets/ansible项目实验/file-20251225095258931.png)
#### 结果显示
![](assets/ansible项目实验/file-20251225095258928.png)