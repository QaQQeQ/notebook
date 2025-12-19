虚拟化：提高资源使用率，是云计算的基础

#kvm虚拟化平台安装
1、虚拟机设置（内存、cpu（勾选虚拟化功能、cpu虚拟化功能需要开启））
1）检查cpu是否具备虚拟化功能
grep -E "vmx|svm" /proc/cpuinfo

2）检查是否加载kvm内核模块
lsmod | grep kvm  #如果没有需要手动加载：modprobe -a kvm
-----------静态ip、firewalld、selinux、yum源

2、安装虚拟化平台及管理工具
   系统版本低的需要先升级：yum upgrade -y 
   uname  -r  #查看内核版本
   cat /etc/centos-release  #查看发行版本

1）安装桌面，可以只安装gui
yum groups install -y "Server with GUI"

2）安装KVM虚拟化平台组件
yum groups install -y "Virtualization Host" "network-file-system-client"  "remote-system-management" "virtualization-platform"

3）安装kvm管理工具
yum install -y virt-manager acpid virt-install virt-viewer openssh-askpass libguestfs-tools NetworkManager

4）启动虚拟化服务
systemctl enable libvirtd.service
systemctl start libvirtd.service

5）重启进入桌面
startx #手动切到图形化

6）进入vmm管理平台
1、远程进入：virt-manager 
2、本地进入：桌面应用工具进入

#新建虚拟机
1）图形创建虚拟机
1、先把系统安装镜像放到宿主机（centos7）
2、virt-manager  #文件---虚拟机


2）命令行创建虚拟机 virt-install
virt-install --os-variant centos7.0  --accelerate --autostart \
             --name server03 \
             --vcpus 1,maxvcpus=2 \
             --memory 1024,maxmemory=2048 \
             --disk device=disk,bus=scsi,path=/var/lib/libvirt/images/server03.qcow2,format=qcow2,size=5,boot_order=1\
             --network network=default,model=virtio,boot_order=2 \
             --location /tmp/CentOS-7-x86_64-DVD-2207-02.iso \            #换成自己安装文件
             --extra-args "console=ttyS0" \
             --nographics \     # 没有图形，文本安装界面
             &

virt-install --os-variant centos7.0  --accelerate --autostart \
             --name server03 \
             --vcpus 1,maxvcpus=2 \
             --memory 1024,maxmemory=2048 \
             --disk device=disk,bus=scsi,path=/var/lib/libvirt/images/server03.qcow2,format=qcow2,size=5,boot_order=1\
             --network network=default,model=virtio,boot_order=2 \
             --location /tmp/CentOS-7-x86_64-DVD-2207-02.iso \     #换成自己安装文件
             --extra-args "console=ttyS0" \
             --graphics vnc,port=-1,keymap='en-us'  \     有图形，使用vnc
             &

#kvm虚拟机管理
1）图形化：（开机关机重启、快照、克隆、调整配置（cpu、内存、网卡、硬盘））自己完成

2）命令行
#虚拟机基本管理
virsh list  #查看所有运行的虚拟机
virsh list --all  # 查看所有包括关机的
virsh start centos01  #启动虚拟机
virsh shutdown centos01  ## 正常关闭
virsh destroy centos01   ## 强制关闭
virsh undefine centos01         # 删除<虚拟机>，但是，保留<关联的存储卷>
virsh undefine --remove-all-storage centos01     # 删除<虚拟机>，同时，删除<关联的存储卷>
virsh dumpxml centos01 > aaa.xml#导出虚拟机的配置
virsh define aaa.xml #使用配置创建虚拟机











[[补充：根分区扩容]]