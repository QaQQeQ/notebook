
---

### 1. NFS 概念与工作原理

*   **定义**: NFS (Network File System，网络文件系统) 是一种分布式文件系统协议，属于 **NAS (Network Attached Storage)** 存储技术的一种。它允许网络中的计算机（客户端）通过网络像访问本地文件一样，访问远程服务器上的共享文件和目录。
*   **核心功能**: 实现跨网络的文件共享，为多台服务器提供一个统一、集中的数据存储位置，确保数据的一致性。
*   **工作模型 (Client/Server)**:
    *   **NFS Server**: 提供文件存储，将本地的物理目录“导出”（export）给网络中的其他机器。
    *   **NFS Client**: 访问服务器导出的目录，将其“挂载”（mount）到自己本地的一个空目录上，之后对该本地目录的操作都会被透明地传送到远程服务器。
*   **依赖于 RPC (Remote Procedure Call)**:
    *   NFS 本身是一个应用程序，它并不知道数据如何在网络中传输。它通过调用 **RPC (远程过程调用)** 协议来完成客户端和服务端之间的通信。
    *   **工作流程**: 客户端发起请求 -> 本地 RPC 服务 -> 网络 -> 远程 RPC 服务 -> 远程 NFS 服务处理请求 -> 返回结果。
    *   **`rpcbind` (或 `portmapper`) 服务**: 由于 NFS 的许多服务（如 `mountd`, `statd`）默认使用随机端口，`rpcbind` 作为一个“端口中介”，运行在固定的 **111** 端口上。客户端首先连接 `rpcbind`，查询所需 NFS 服务的当前端口号，然后再与该端口建立连接。

### 2. NFS 核心组件与部署

**服务端 (Server) 操作流程:**

1.  **安装软件包**:
    *    `yum -y install rpcbind nfs-utils` 

2.  **配置共享目录 (`/etc/exports`)**:
    *   这是 NFS 服务器的核心配置文件，定义了哪个目录共享给哪个客户端，以及使用何种权限，在CentOS7中可通过`/etc/exports.d/xxx.exports`来定义共享目录。
    *   **格式**: `共享的目录路径 客户端(权限选项1,权限选项2,...)`
		```
        cat > /etc/exports.d/sharedirs.exports <<EOF
        /share01 *(rw)
        /share02 *(rw)
        /share03 *(rw)
        EOF
		```
    *   **常用权限选项**:
        *   `rw` / `ro`: 读写 / 只读权限。
        *   `sync`: 同步写入，数据会先写入硬盘再响应客户端，数据安全性高。
        *   `async`: 异步写入，性能高但有数据丢失风险。
        *   **`root_squash` (默认)**: 安全选项。当客户端以 `root` 用户身份写入文件时，在服务器端会将其身份压缩为匿名用户（如 `nfsnobody`），以防止客户端 `root` 滥用权限。
        *   **`no_root_squash`**: 不压缩 `root` 身份，客户端 `root` 在共享目录中拥有 `root` 权限。**非常危险，需谨慎使用**。

3.  **启动服务**:
    *   必须先启动 `rpcbind` 服务，再启动 `nfs-server` 服务。
    *   `systemctl start rpcbind`
    *   `systemctl start nfs-server`

4.  **配置防火墙**:
    *   需要放行 `rpc-bind`、`nfs`以及 `mountd`、`lockd` 等服务的端口。为便于管理，通常会**固化这些服务的随机端口**（通过修改 `/etc/sysconfig/nfs` 文件）。
		```bash
		firewall-cmd --permanent --zone=public --add-service=nfs --add-service=rpc-bind --add-service=mountd
		firewall-cmd --reload
		```

5.  **检查**:
    *   `exportfs -v`: 在服务器上查看当前导出的共享目录及其权限。
    *   `showmount -e <服务器IP>`: 在客户端或服务器上查看指定服务器导出了哪些目录。

**客户端 (Client) 操作流程:**

1.  **安装软件包**:
    *   同样需要安装 `nfs-utils`。

2.  **手动挂载**:
    *   创建一个本地空目录作为挂载点。
    *   使用 `mount` 命令进行挂载：`mount -t nfs <服务器IP>:<共享目录> <本地挂载点>`
    *   示例: `mount -t nfs -o _netdev 192.168.10.10:/share /mnt/nfs_data`

3.  **自动挂载**:
    *   **`fstab` 方式**: 将挂载信息写入 `/etc/fstab` 文件，实现开机自动挂载，适用于需要永久挂载的场景。
        *   格式: `<服务器IP>:<共享目录> <本地挂载点> nfs defaults 0 0`
    *   **`autofs` 方式 (推荐)**: 一种更智能的按需挂载服务。只有当用户尝试访问挂载点时，系统才会自动进行挂载；在一段时间没有访问后，会自动卸载，节省系统资源。
