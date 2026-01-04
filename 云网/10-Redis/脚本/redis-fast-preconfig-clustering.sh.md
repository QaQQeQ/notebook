```sh
#!/bin/bash
echo "### 0 firewalld"
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
echo "### 1 install base rpm ###"
dnf install -y gcc make tcl vim bash-completion net-tools wget zip unzip bzip2 gcc-c++ tar
echo "### 2 download software ###"
wget -P /root/ https://gitee.com/tianhairui/sh/raw/master/Software/8.2.0.tar.gz
echo "### 3 user && directory ###"
groupadd -g 820 redis
useradd -u 8200 -g 820 redis
echo '123456' |passwd --stdin redis
mkdir -p /redis/install/{7001,7002}/log
mkdir -p /redis/install/{7001,7002}/conf
mkdir -p /redis/install/{7001,7002}/data
chown -R redis.redis /redis/install
echo "### 4 resources limit ###"
cat <<"EOF" > /etc/security/limits.conf
redis        hard    nofile          10240
redis        soft    nofile          10240
redis        hard    nproc           8192
redis        soft    nproc           8192
EOF
echo "### 5 download jemalloc && make install ###"
wget -P /root/ https://gitee.com/tianhairui/sh/raw/master/Software/jemalloc-5.3.0.tar.bz2
tar -jxvf /root/jemalloc-5.3.0.tar.bz2
cd /root/jemalloc-5.3.0
./configure
make && make install
mv /root/8.2.0.tar.gz /redis/install/
chmod 777 -R /redis/install/*
echo "### 6 redis install ###"
su - redis -c "
cd /redis/install &&
tar -zxvf 8.2.0.tar.gz &&
cd /redis/install/redis-8.2.0/deps &&
make fast_float jemalloc linenoise lua hiredis &&
cd /redis/install/redis-8.2.0 &&
make &&
cd /redis/install/redis-8.2.0/src &&
make PREFIX=/redis/install install &&
cp ../redis.conf /redis/install/7001/conf/redis7001.conf &&
cp ../redis.conf /redis/install/7002/conf/redis7002.conf &&
echo 'export PATH=\$PATH:\$HOME/.local/bin:/redis/install/bin:\$HOME/bin' >> ~/.bash_profile
"
```