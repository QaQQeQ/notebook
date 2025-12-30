
---

### 一. Nginx 简介

*   **定位**: Nginx 是一款轻量级、高性能的 Web 服务器、反向代理服务器及电子邮件（IMAP/POP3/SMTP）代理服务器。
*   **与 HTTPD (Apache) 的对比**:
    *   **Nginx**: 采用**事件驱动**的异步非阻塞架构。占用内存少，能够支持更高的并发连接（轻松应对数万并发），特别擅长处理**静态文件**和作为**反向代理**。
    *   **HTTPD**: 采用多进程/多线程模型（如 prefork/worker）。功能模块丰富，生态成熟，在处理**动态内容**方面有悠久历史。相对 Nginx 来说，内存占用更高。
*   **应用场景**:
    1.  **静态 Web 服务器**: 高效地提供 HTML、图片、CSS 等静态资源。
    2.  **反向代理**: 将客户端请求转发给后端的多个应用服务器（如 Tomcat, uWSGI），隐藏后端服务细节。
    3.  **负载均衡**: 将流量分发到多个后端服务器，提高网站的可用性和性能。

### 二. Nginx Make 安装

这部分讲述了通过编译源码的方式来安装和管理 Nginx。

*   **安装方式**: 不同于 YUM，源码安装提供了更高的定制化，可以自由选择需要编译的模块。
*   **部署流程**:
    1.  下载 Nginx 源码包。
    2.  执行 `./configure` 脚本，配置编译参数（如安装路径、启用的模块）。
    3.  执行 `make` 进行编译。
    4.  执行 `make install` 进行安装。
*   **服务管理**:
    *   **命令行方式(需要将nginx添加到环境变量才可以直接使用)**:
		```
			cat > /etc/profile.d/nginx.sh <<EOF
			export PATH="/usr/local/nginx/sbin:\$PATH"
			EOF
			source /etc/profile.d/nginx.sh
		```
		*   启动: 直接执行 Nginx 程序（如 `/usr/local/nginx/sbin/nginx`）。
        *   停止: `nginx -s stop` (快速停止) 或 `nginx -s quit` (优雅停止)。
        *   重新加载配置: `nginx -s reload` (平滑重载，不中断服务)。
        *   检查配置: `nginx -t`。
    *   **systemd 脚本**: 可以编写 `.service` 文件，以便使用 `systemctl start/stop/reload nginx` 进行管理。
*   **平滑升级**: Nginx 允许在不中断现有服务连接的情况下升级版本，这是其重要特性。大致流程是：编译新版本的二进制文件，然后通过信号（如 `USR2`, `WINCH`, `QUIT`）逐步替换旧的主进程和工作进程。

### 三. Nginx 基本管理

*   **进程模型**:
    *   **Master Process (主进程)**: 负责读取和验证配置、管理 Worker 进程。不处理实际请求。
    *   **Worker Process (工作进程)**: 由 Master 进程创建。实际处理客户端的连接和请求，通常配置为与 CPU 核心数相同。
*   **日志文件**:
    *   **access.log (访问日志)**: 记录所有客户端的请求信息。
    *   **error.log (错误日志)**: 记录服务运行中的错误、警告等诊断信息。
    *   **日志切割**: 为了防止日志文件过大，通常会使用 `logrotate` 等工具定期对日志进行分割、压缩和归档。
*   **主配置文件 `nginx.conf`**:
    *   采用**块状结构**，清晰易读。主要包含：
        *   **main/global**: 全局配置，如运行用户、worker进程数。
        *   **events**: 配置网络连接相关的参数，如 `worker_connections`。
        *   **http**: HTTP 协议相关配置的容器。
        *   **server**: 定义一个**虚拟主机**。
        *   **location**: 匹配特定的 URL 请求路径，并为其定义处理规则。

### 四. Nginx 目录浏览

*   **功能**: 当访问一个目录，而该目录下没有找到指定的索引文件（如 `index.html`）时，允许在浏览器上以列表形式显示该目录下的所有文件和子目录。
*   **实现指令**: `autoindex on;`
*   **配置示例**:
    ```nginx
    # 在server中添加
        autoindex on; # 开启目录浏览
        autoindex_exact_size   off;##以友好的<文件尺寸大小>显示<目录列表中的文件大小>
        autoindex_localtime    on;## 以<本地时间>显示<目录列表中的文件>
        charset   utf-8;## 支持<中文>显示<目录列表中的文件名>
    }
    ```

### 五. Nginx 虚拟目录

虚拟目录用于将一个 URL 路径映射到服务器上的一个物理路径。

*   **`root` 指令**:
    *   **功能**: 定义了 `location` 匹配到的请求的**根目录**。Nginx 会将 `root` 指定的路径与 URL 路径**拼接**起来寻找文件。
		```nginx
		location /img {
			root /var/www; 
			# 请求 /img/logo.png 会寻找物理文件 /var/www/img/logo.png
		} 
		```
*   **`alias` 指令**:
    *   **功能**: 为 `location` 匹配到的路径定义一个**别名**。Nginx 会用 `alias` 指定的路径**替换**掉 URL 中匹配到的部分。
		```nginx
		location /img {
		    alias /var/www/images; 
		    # 请求/img/logo.png 会寻找物理文件 /var/www/images/logo.png
		} 
		```

### 六. Nginx 虚拟主机

虚拟主机让一台服务器可以运行多个网站。

*   **实现方式**: 使用 `http` 块内的 `server` 块。每个 `server { ... }` 块代表一个独立的虚拟主机。
*   **主要类型**:
    1.  **基于端口(相同IP不同端口)**: 在 `listen` 指令中指定不同的端口号。
    2.  **基于IP(不同IP相同端口)**: 在 `listen` 指令中指定不同的 IP 地址。
    3.  **基于域名(相同IP相同端口)**: 使用 `server_name` 指令来匹配请求头中的 `Host` 字段。
*   **基于域名的虚拟主机配置示例**:
    ```nginx
    http {
        # ... 其他 http 配置 ...

        # 网站1: site1.example.com
        server {
            listen 80;
            server_name site1.example.com;
            charset  utf-8;
            location / {
	            root /var/www/site1;
	            index index.html;
            }   
        }

        # 网站2: site2.example.com
        server {
            listen 80;
            server_name site2.example.com;
            charset  utf-8;
            location / {
	            root /var/www/site2;
	            index index.html;
            }
        }
    }
    ```

### 七. Nginx 安全加密

*   **启用模块**: 源码编译安装时，需要通过 `--with-http_ssl_module` 参数启用 SSL 模块。
*   **配置流程**:
    1.  **获取/创建证书**: 准备好 SSL 证书文件 (`.crt`) 和私钥文件 (`.key`)。
    2.  **配置 `server` 块**:
        *   使用 `listen 443 ssl;` 来监听 HTTPS 默认端口并启用 SSL。
        *   使用 `ssl_certificate` 指令指向证书文件路径。
        *   使用 `ssl_certificate_key` 指令指向私钥文件路径。
    *   **推荐配置**: 通常还会配置 SSL 协议版本、加密套件等，以增强安全性。
*   **示例**:
    ```nginx
    server {
        listen 443 ssl;
        server_name localhost;
        
        ssl_certificate private/server.crt;
        ssl_certificate_key private/server.key;
        
        ssl_session_cache   shared:SSL:1m;
        ssl_session_timeout 5m;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        
        location / {
	        root  html;
	        index index.html;
	    }
    }
    ```

### 八. Nginx 与 PHP 集成

Nginx 本身不能执行 PHP，它需要通过 **FastCGI** 协议将 PHP 请求转发给外部的 **PHP-FPM** (FastCGI Process Manager) 服务来处理。

*   **架构**: **`Client` <-> `Nginx` <-> `PHP-FPM` <-> `PHP Script`**
*   **配置流程**:
    1.  **安装 PHP组件 PHP-FPM管理器实例**:
        ```bash
        yum -y install php php-devel php-mysql
        yum -y install php-fpm
        systemctl start php-fpm
        systemctl enable php-fpm
        ## 设置：php-fpm 管理器实例的运行用户账户、运行组账户
        sed -i -r 's/^\s*user\s*=.*/user = nginx/' /etc/php-fpm.d/www.conf
        sed -i -r 's/^\s*group\s*=.*/group = nginx/' /etc/php-fpm.d/www.conf
        systemctl restart php-fpm
        ```
    2.  **配置 Nginx**:
        *   在 Nginx 的 `server` 块中添加一个 `location` 来匹配 `.php` 文件请求。
        *   使用 `fastcgi_pass` 指令将请求转发给 PHP-FPM 监听的地址（通常是 `127.0.0.1:9000` 或一个 unix socket 文件）。
        *   设置必要的 FastCGI 参数，如 `fastcgi_param SCRIPT_FILENAME`。
*   **示例**:
    ```nginx
    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        root           html;
        fastcgi_index  index.php;
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        ## 设置：错误页
        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }
    }
    ```