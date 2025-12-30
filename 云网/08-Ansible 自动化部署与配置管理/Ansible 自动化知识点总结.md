
---
### 第一部分：Ansible 核心概念与基础


1.  **Ansible简介**：
    *   一款极其简单的IT自动化引擎，用于应用部署、配置管理、任务编排等。
    *   **核心特性**：
        *   **无代理 (Agentless)**：被管理节点无需安装任何客户端软件，通过标准的SSH协议通信。
        *   **幂等性 (Idempotent)**：多次执行同一个任务，结果始终保持一致，不会产生副作用。
        *   **YAML语法**：使用YAML语言编写Playbook（剧本），结构清晰，可读性强。
        *   **模块化**：通过调用不同功能的模块来完成具体任务。

2.  **环境准备与连接**：
    *   **安装**：在主控端安装Ansible即可，被控端只需开启SSH服务并安装Python。
    *   **连接方式**：基于SSH密钥对实现免密登录是标准实践。
    *   **Inventory (主机清单)**：一个定义了被管理主机及其分组的配置文件（通常是INI或YAML格式），是Ansible操作对象的基础。
    *   **提权 (`become`)**：通过配置`become: yes`和`become_user: root`，可以让普通用户以root权限在被控端执行任务。

3.  **两种执行方式**：
    *   **Ad-Hoc 命令**：用于执行临时的、一次性的简单任务，快速方便。
    *   **Playbook (剧本)**：用于执行复杂的、需要编排的、可重复的任务集合，是Ansible自动化的核心。

---

### 第二部分：Ansible 核心模块详解

#### 1. `ping` 模块

*   **功能**：测试主控端与被控端之间的网络连通性和Python解释器是否可用。
*   **特点**：最基础的模块，是验证环境配置是否正确的首选。返回`"pong"`表示成功。
*   **缺点**：功能单一，仅用于连通性测试，不执行任何实际操作。
*   **举例**：
    ```bash
    # 测试webservers主机组所有主机的连通性
    ansible webservers -m ping
    ```

#### 2. `raw` 模块

*   **功能**：在远程主机上执行原生、底层的Shell命令。
*   **特点**：极少数不依赖Python的模块之一。主要用于在尚未安装Python的裸机上安装Python环境。
*   **缺点**：功能非常有限，返回的是原始输出（stdout/stderr），没有结构化的数据。不支持`become`提权等高级功能。
*   **举例**：
    ```bash
    # 在 192.168.1.10 主机上使用yum安装python3
    ansible 192.168.1.10 -m raw -a "yum install -y python3"
    ```

#### 3. `command` 模块

*   **功能**：在远程主机上执行简单的命令。
*   **特点**：默认模块，如果不指定`-m`参数，就使用此模块。出于安全考虑，它不会通过shell执行，因此不支持管道符`|`、重定向`>`、`&`等shell特性和变量（如`$HOME`）。
*   **缺点**：功能受限，无法使用shell的强大功能，复杂的命令无法执行。
*   **举例**：
    ```bash
    # 查看dbservers主机组中主机的内核版本
    ansible dbservers -m command -a "uname -r"
    # 或者省略-m command
    ansible dbservers -a "uname -r"
    ```

#### 4. `shell` 模块

*   **功能**：在远程主机上通过`/bin/sh`执行命令，功能与直接登录shell操作一样。
*   **特点**：支持所有shell特性，如管道符、重定向、变量等。是执行复杂命令的首选。
*   **缺点**：如果命令可以被其他专用模块（如`yum`, `file`）替代，则不推荐使用`shell`，因为它破坏了幂等性原则。
*   **举例**：
    ```bash
    # 查找/var/log下包含 "error" 字符串的日志文件数量
    ansible webservers -m shell -a "grep 'error' /var/log/*.log | wc -l"
    ```

#### 5. `script` 模块

*   **功能**：将主控端本地的一个脚本文件传输到远程主机并执行。
*   **特点**：方便执行预先编写好的复杂脚本，无需先`copy`再`shell`执行。脚本执行后会在远程主机上被删除。
*   **缺点**：脚本本身需要保证幂等性，Ansible只负责执行。
*   **举例**：
    ```bash
    # 在所有主机上运行本地的 /data/scripts/init.sh 脚本
    ansible all -m script -a "/data/scripts/init.sh"
    ```

#### 6. `file` 模块

*   **功能**：管理文件、目录、软链接的属性。
*   **特点**：功能强大，可以创建文件/目录 (`state=touch/directory`)、删除 (`state=absent`)、创建软链接 (`state=link`)、修改权限和属主/属组。完全幂等。
*   **缺点**：不能处理文件内容。
*   **举例**：
    ```bash
    # 在webservers组上创建目录 /data/www，并设置权限和属主
    ansible webservers -m file -a "path=/data/www state=directory owner=nginx group=nginx mode=0755"
    ```

#### 7. `copy` 模块

*   **功能**：将主控端的文件或目录复制到远程主机。
*   **特点**：最常用的文件传输模块。支持直接指定文件内容 (`content`参数)，支持复制后校验文件完整性。
*   **缺点**：只能从主控端推送到被控端，不能反向拉取。
*   **举例**：
    ```bash
    # 将本地的nginx.conf文件复制到所有web服务器的指定位置
    ansible webservers -m copy -a "src=/etc/ansible/files/nginx.conf dest=/etc/nginx/nginx.conf"
    ```

#### 8. `fetch` 模块

*   **功能**：从远程主机拉取文件到主控端。
*   **特点**：`copy`模块的反向操作。拉取回来的文件会保存在以主机名命名的目录中，保持了原始的目录结构。
*   **缺点**：不能自定义本地保存的文件名。
*   **举例**：
    ```bash
    # 从web01主机拉取/var/log/nginx/access.log日志文件到本地的/tmp/logs目录
    ansible web01 -m fetch -a "src=/var/log/nginx/access.log dest=/tmp/logs/"
    ```

#### 9. `user` / `group` 模块

*   **功能**：管理远程主机上的用户和组。
*   **特点**：可以创建、删除用户/组，管理用户附加组、家目录、UID/GID等。幂等。
*   **缺点**：功能相对专一，无法进行密码管理等复杂操作（通常用`user`模块的`password`参数设置密码哈希值）。
*   **举例**：
    ```bash
    # 在所有主机上创建一个名为 'devuser' 的用户
    ansible all -m user -a "name=devuser state=present"

    # 删除 'devuser' 用户
    ansible all -m user -a "name=devuser state=absent"
    ```

#### 10. `unarchive` 模块

*   **功能**：在远程主机上解压压缩包。
*   **特点**：支持`.tar.gz`, `.zip`等多种格式。可以先从主控端复制压缩包再解压，也可以解压远程主机上已有的压缩包。
*   **缺点**：对于不支持的压缩格式无能为力。
*   **举例**：
    ```bash
    # 将本地的 myapp.tar.gz 复制到远程的 /tmp 并解压到 /opt
    ansible webservers -m unarchive -a "src=myapp.tar.gz dest=/opt/"
    ```

---
