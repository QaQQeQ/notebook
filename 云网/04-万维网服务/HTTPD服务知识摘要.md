
---

### 一. HTTP 协议概述

这是理解所有Web服务的基础。

*   **工作流程**: HTTP是一个**客户端-服务器**协议。客户端（如浏览器）发送一个 **HTTP 请求 (Request)** 到服务器，服务器处理请求后返回一个 **HTTP 响应 (Response)**。
*   **核心特点**:
    *   **无连接**: 每次请求-响应周期结束后，连接默认会断开。
    *   **无状态**: 服务器不会记录之前客户端的任何信息。每次请求都是独立的。（Session和Cookie技术用于弥补这一点）。
*   **URI, URL, URN**:
    *   **URL (统一资源定位符)**: 我们最常用的“网址”，它不仅标识了资源，还提供了如何找到它的方法（例如 `http://www.example.com/page.html`）。
*   **HTTP 协议包**:
    *   **请求包 (Request)**: 主要由三部分组成：
        1.  **请求行**: 包含请求方法 (如`GET`, `POST`)、URL 和 HTTP协议版本。
        2.  **请求头**: 包含客户端环境、认证等附加信息。
        3.  **请求体**: `POST` 请求时携带的数据。
    *   **响应包 (Response)**: 主要由三部分组成：
        1.  **状态行**: 包含HTTP协议版本、**状态码** (如 `200 OK`) 和状态描述。
        2.  **响应头**: 包含服务器信息、内容类型等。
        3.  **响应体**: 实际返回给客户端的数据内容（如HTML代码）。
*   **常用状态码**:
    *   `200 OK`: 请求成功。
    *   `301/302`: 永久/临时重定向。
    *   `403 Forbidden`: 服务器拒绝访问。
    *   `404 Not Found`:请求的资源不存在。
    *   `500 Internal Server Error`: 服务器内部错误。
    *   `503 Service Unavailable`: 服务器过载或正在维护。

### 二. HTTPD YUM 安装部署

这部分讲述如何在 CentOS/RHEL 系统上快速部署 HTTPD 服务。

*   **安装**: 使用 YUM 包管理器一键安装。
    ```bash
    yum -y install httpd
    ```
*   **核心文件**:
    *   **主配置文件**: `/etc/httpd/conf/httpd.conf`
    *   **模块化配置文件目录**: `/etc/httpd/conf.d/`
    *   **默认网站根目录**: `/var/www/html/`
*   **服务管理**: 使用 `systemctl` 命令管理服务。
    ```bash
    systemctl start httpd   # 启动服务
    systemctl restart httpd   # 重启服务
    systemctl enable httpd  # 设置开机自启
    systemctl status httpd  # 查看服务状态
    systemctl stop httpd   # 停止服务
    ```
*   **部署流程**:
    1.  安装 `httpd` 软件包。
    2.  启动并设置开机自启。
    3.  **配置防火墙**，允许外部访问 HTTP (80端口) 服务。
        ```bash
        firewall-cmd --permanent --zone=public --add-port=80/tcp
        firewall-cmd --reload
        ```
    4.  在 `/var/www/html` 目录下创建或放置网页文件（如 `index.html`）。
    5.  通过浏览器访问服务器的 IP 地址。

### 三. HTTPD 基本管理

这部分涉及 `httpd.conf` 主配置文件的核心指令。

*   **配置文件结构**:
    *   采用模块化设计，主配置文件 `httpd.conf` 通过 `IncludeOptional conf.d/*.conf` 指令加载 `conf.d` 目录下的所有 `.conf` 文件，便于管理和扩展。
*   **核心配置指令**:
    *   `Listen 80`: 设置HTTPD服务监听的端口号，默认为80。
    *   `DocumentRoot "/var/www/html"`: 指定网站文件的根目录。
    *   `ServerName www.example.com:80`: 设置服务器的域名，在虚拟主机配置中尤其重要。
    *   `DirectoryIndex index.html`: 当用户只访问目录时，默认显示的首页文件名。
    *   `<Directory>` 容器: 用于定义特定目录的访问控制策略、权限和功能。
    *   **日志文件**:
        *   `ErrorLog "logs/error_log"`: 错误日志路径。
        *   `CustomLog "logs/access_log" combined`: 访问日志路径和格式。

### 四. HTTPD 虚拟目录

虚拟目录允许你将网站根目录之外的物理路径映射到一个URL路径上。

*   **实现指令**: `Alias`
*   **功能**: 将一个URL路径（别名）指向服务器上一个完全不同的物理文件系统路径。
*   **配置示例**:
    ```apacheconf
    # 将 URL /data/ 映射到物理路径 /var/ftp/pub/
    Alias /data/ "/var/ftp/pub/"

    # 必须为被映射的物理路径配置访问权限
    <Directory "/var/ftp/pub/">
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ```
    配置后，访问 `http://服务器IP/data/` 实际看到的是 `/var/ftp/pub/` 目录下的内容。

### 五. HTTPD 虚拟主机

虚拟主机技术让一台物理服务器可以托管多个独立的网站。

*   **定义**: 在单个HTTPD服务实例上运行多个网站（如 `site1.com`, `site2.com`），每个网站有自己独立的域名、根目录和配置。
*   **实现方式**: 使用 `<VirtualHost>` 容器指令。
*   **三种主要类型**:
    1.  **基于IP(相同端口不同IP)**: 每个网站使用一个独立的IP地址。（不常用）
    2.  **基于端口(不同端口相同IP)**: 所有网站共享一个IP，但使用不同的端口号访问。（常用于内部或测试环境）
    3.  **基于域名(相同端口相同IP)**: 所有网站共享一个IP和端口（通常是80），HTTPD通过请求头中的 `Host` 字段来区分用户想访问哪个网站。
*   **基于域名的虚拟主机配置示例**:
    ```
	mkdir /www.example1.test.com
	echo "这是：example1 网站首页" > /www.example1.test.com/index.html

	cat > /etc/httpd/conf.d/vhost01.conf <<EOF
	<VirtualHost *:80>
		ServerName "www.example1.test.com"
		DocumentRoot "/www.example1.test.com"
		<Directory "/www.example1.test.com">
		    Options Indexes FollowSymLinks
		    AllowOverride None
		    Require all granted
		</Directory>
	</VirtualHost>
	EOF
	
	mkdir /www.example2.test.com
	echo "这是：example2 网站首页" > /www.example2.test.com/index.html
	
	cat > /etc/httpd/conf.d/vhost01.conf <<EOF
	<VirtualHost *:80>
		ServerName "www.example2.test.com"
		DocumentRoot "/www.example2.test.com"
		<Directory "/www.example2.test.com">
		    Options Indexes FollowSymLinks
		    AllowOverride None
		    Require all granted
		</Directory>
	</VirtualHost>
	EOF
	
	systemctl restart httpd
	firewall-cmd --permanent --zone=public --add-port=80/tcp
	firewall-cmd --reload
	```

### 六. HTTPD 安全加密 (HTTPS/SSL)

为网站启用 HTTPS，实现数据传输的加密。

*   **工作原理**: 通过SSL/TLS协议，在客户端和服务器之间建立一个加密通道。它需要使用**数字证书**来验证服务器身份。
*   **安装模块**: 需要 `mod_ssl` 模块的支持。
    ```bash
    yum -y install mod_ssl
    ```
*   **核心配置**:
    1.  **获取/创建证书**: 包括私钥 (`.key` 文件) 和公钥证书 (`.crt` 文件)。
    2.  **配置虚拟主机**: 创建一个监听443端口的 `<VirtualHost>`。
    3.  **关键指令**:
        *   `SSLEngine on`: 在此虚拟主机上启用SSL。
        *   `SSLCertificateFile /path/to/your/certificate.crt`: 指定证书文件路径。
        *   `SSLCertificateKeyFile /path/to/your/private.key`: 指定私钥文件路径。
*   **默认端口**: HTTPS 的标准端口是 **443**。
*   **示例**：
```bash

mkdir /etc/httpd/conf.d/key
cd /etc/httpd/conf.d/key
# 生成自签名证书和私钥
openssl req -new -x509 -keyout my-key.pem -out my-cert.pem -nodes -batch
# 设置证书和私钥的权限
chmod 400 ./my-key.pem ./my-cert.pem
# 查看私钥文件内容
cat ./my-key.pem

# 配置示例 向/etc/httpd/conf.d/vhost01.conf配置文件<VirtualHost>中添加以下内容
SSLEngine on
SSLCertificateKeyFile conf.d/key/my-key.pem
SSLCertificateFile conf.d/key/my-cert.pem

```

### 七. HTTPD 工作模式 (MPM)

MPM (Multi-Processing Module) 决定了 Apache 如何处理客户端请求，直接影响其性能和资源消耗。

*   **三种主要模式**:
    1.  **prefork (默认)**: 一个<父进程>生成多个<子进程>，每个<子进程>仅包含一个**线程**，最终由**线程**处理<用户请求>。稳定、兼容性好，但内存消耗大，不适合高并发场景。
    2.  **worker**: 一个<父进程>生成多个<子进程>，每个<子进程>包含多个**线程**。最终由**线程**处理<用户请求>。内存效率比 `prefork` 高，能处理更多并发连接。
    3.  **event**: `worker` 模式的变种。将一个线程作为**侦听器线程**来管理持久连接(Keep-Alive)，以实现其复用，进一步提升了高并发场景下的性能，特别适合处理大量长连接。
*   **配置与切换**: 在 `/etc/httpd/conf.modules.d/00-mpm.conf` 文件中，通过注释和取消注释来选择加载哪个MPM模块。修改后需要重启httpd服务。

### 八. HTTPD PHP 集成

让 Apache 能够解析和执行 PHP 脚本。

*   **集成方式**: Apache 通常通过一个名为 `mod_php` 的模块来与 PHP 协同工作。当 Apache 收到对 `.php` 文件的请求时，它不会直接发送文件内容，而是将请求交给 PHP 解释器处理。
*   **安装组件**:
    1.  安装 `httpd`。
    2.  安装 `php` 支持组件。
    ```bash
    yum -y install php* -x php-mysqlnd
    ```
*   **工作流程**:
    1.  安装PHP后，会自动在 `conf.d` 目录下生成 `php.conf` 文件。
    2.  该配置文件告诉 Apache: 当遇到以 `.php` 结尾的文件时，使用 `application/x-httpd-php` 处理器（即PHP解释器）来处理它。
    3.  重启 `httpd` 服务使配置生效。
    4.  在网站根目录下创建一个 `index.php` 文件，写入 `<?php phpinfo(); ?>` 进行测试。