
---

### **一. Ceph 简介与核心概念**

*   **定义**: Ceph是一个**开源的、统一的、软件定义的分布式存储系统**。它设计之初就旨在提供卓越的性能、高可靠性和无限的可扩展性。
*   **统一存储 (Unified Storage)**: Ceph最大的特点是可以在**同一个存储集群**上同时提供三种不同类型的存储服务，以满足各种应用场景的需求：
    1.  **对象存储 (Object Storage/RGW)**: 提供与 Amazon S3 和 OpenStack Swift 兼容的 RESTful API 接口，适用于非结构化数据，如图片、视频、备份文件等。
    2.  **块存储 (Block Storage/RBD)**: 提供网络块设备（虚拟硬盘），常用于为虚拟机（如 OpenStack, KVM）提供云硬盘或挂载到物理服务器作为高性能磁盘。
    3.  **文件存储 (File System/CephFS)**: 提供一个与 POSIX 兼容的网络文件系统（类似 NFS），支持多客户端并发读写共享文件。
*   **核心架构 - RADOS**: Ceph 的所有存储服务都构建在其核心基础之上，即 **RADOS (Reliable Autonomic Distributed Object Store)**。RADOS 是一个可靠、自动、分布式的对象存储系统，负责数据的实际存储、复制、故障检测和自我修复等底层工作。

### **二. Ceph 的核心组件**

*   **OSD (Object Storage Daemon)**: **存储守护进程**。每个 OSD 负责管理服务器上的一块物理磁盘。它执行所有的数据存储、复制、再平衡和恢复等实际工作。集群的容量和性能基本由 OSD 的数量和性能决定。
*   **MON (Monitor)**: **监控器**。集群的“大脑”，负责维护整个集群的状态图谱（Cluster Map），包括 OSD map、MON map、PG map 和 CRUSH map。它管理着集群的成员关系和认证信息。生产环境至少需要3个（奇数个）MON 以实现高可用。
*   **MGR (Manager)**: **管理器**。负责收集集群的各种指标和状态信息，并提供给外部系统（如 Ceph Dashboard 监控面板）。
*   **MDS (Metadata Server)**: **元数据服务器**。**仅为 CephFS 文件存储服务**。它负责存储文件系统的元数据（如目录结构、文件名、权限等），使得 CephFS 能够提供 POSIX 文件系统语义。

### **三. Ceph 单节点集群部署**

#### **第一步：环境准备**

在开始部署 Ceph 之前，需要对节点（在本例中为 `ceph.test.com`）进行一系列准备工作。

1.  **配置DNS服务 (可选但推荐)，在实际单机测试中，也可以通过修改 `/etc/hosts` 文件来代替。**
2.  **节点基础设置**:
    *   **设置主机名 (FQDN)**: 确保节点的主机名是完整的域名格式。
        ```bash
        hostnamectl set-hostname ceph.test.com
        ```
    *   **安装前置软件包**: `ceph-deploy` 工具依赖 Python 3。
        ```bash
        yum install -y python3 python3-pip
        ```
    *   **关闭防火墙**: 为简化部署，教程中选择关闭并禁用防火墙。
        ```bash
        systemctl disable firewalld
        systemctl stop firewalld
        ```
    *   **确保SSH主机密钥一致性**: 确保 `ceph-deploy` 工具能顺利通过 SSH 连接节点。

#### **第二步：使用 `ceph-deploy` 部署 Ceph 集群**
略
### **四. Ceph 存储池与三种存储服务的使用**

**存储池 (Pool)** 是存储数据的逻辑分区。在使用任何 Ceph 服务前，必须先创建对应的 Pool。

1.  **CephFS (文件存储)**:
    *   **创建**: 需要创建两个 Pool（一个用于数据，一个用于元数据），然后使用 `ceph fs new` 命令创建文件系统。
    *   **客户端挂载**: 客户端安装 `ceph-common` 包，使用 `mount -t ceph` 命令将 CephFS 挂载到本地目录，像使用本地文件夹一样操作。

2.  **RBD (块存储)**:
    *   **创建**: 创建一个 Pool，然后使用 `rbd create` 命令在 Pool 中创建一个块设备镜像（image）。
    *   **客户端使用**:
        *   `rbd map <image-name>`: 将远程的块设备映射到本地，生成一个 `/dev/rbd*` 设备文件。
        *   之后可以像操作本地硬盘一样，对其进行格式化（如 `mkfs.xfs`）和挂载（`mount`）。

3.  **RGW (对象存储)**:
    *   **部署**: 部署 RGW 服务实例，它会监听一个 HTTP 端口。
    *   **使用**:
        *   创建 RGW 用户。
        *   使用 S3 兼容的工具（如 `s3cmd`）或编程语言的 SDK，通过 HTTP API 进行操作，如创建存储桶（bucket）、上传/下载/删除对象（object）。

### **五. 用户管理与访问控制 (`ceph auth`)**

Ceph 拥有精细的权限控制系统。

*   **流程**: `创建用户` -> `为用户授权 (caps)` -> `获取用户密钥 (keyring)`。
*   **授权 (Capabilities)**: 可以精确地为用户分配对 MON、OSD、MDS 以及特定 Pool 的读、写、执行权限。
*   **客户端认证**: 客户端必须持有有效的 `ceph.conf` 配置文件和对应的用户 `keyring` 文件，才能与 Ceph 集群通信并访问被授权的资源。

### **六. 集成与扩展 (NFS-Ganesha)**

*   资料中还提到了通过 **NFS-Ganesha** 服务将 CephFS 导出为 NFS 共享。这是一种兼容传统应用的方案，让不支持 CephFS 协议但支持 NFS 协议的客户端也能访问 Ceph 存储，进一步增强了 Ceph 的通用性。