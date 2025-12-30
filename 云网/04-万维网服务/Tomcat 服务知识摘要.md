
---

### 一. Tomcat 简介

*   **概念**: Tomcat 是由 Apache 软件基金会下属的 Jakarta 项目开发的**开源、轻量级的 Web 应用服务器**。它是一个 **Servlet 容器**，实现了对 Java Servlet、JavaServer Pages (JSP)、Java Expression Language (EL) 和 WebSocket 等 Java EE 规范的支持。
*   **定位**: Tomcat 主要用于运行和托管**基于 Java 的动态网站**。它本身具备处理 HTTP 请求的能力，但性能和并发能力不如 Nginx。
*   **应用场景**:
    1.  **独立运行**: 中小型项目可以直接使用 Tomcat 作为 Web 服务器来发布 Java Web 应用（通常是 `.war` 包）。
    2.  **作为后端应用服务器**: 在生产环境中，Tomcat 通常被置于 Nginx 或 Apache HTTPD 之后。由前端的 Nginx/HTTPD 负责处理静态资源、SSL 加密和负载均衡，然后将动态请求（如 JSP、Servlet）**反向代理**给后端的 Tomcat 集群处理。这种 **动静分离** 的架构是主流。

### 二. Tomcat 安装

Tomcat 的运行依赖于 Java 环境，因此部署前必须先安装 **JDK (Java Development Kit)**。

*   **YUM 安装 (以 CentOS 7 为例)**:
    1.  **安装**: `yum -y install java-1.8.0-openjdk* tomcat*`。这种方式会自动处理依赖关系。
    2.  **特点**:
        *   **优点**: 简单快捷，能自动注册为 `systemd` 服务，方便使用 `systemctl start/stop/status tomcat` 管理。
        *   **缺点**: 版本通常比较旧（如 CentOS 7 仓库中为 Tomcat 7），定制性差。
    3.  **配置**: 需要配置防火墙开放端口（默认为 8080），并可能需要调整 SELinux 策略或禁用它。
    4.  **设置**: 针对Tomcat的<用户登录环境变量>
    	```
		cat <<'EOF' > /etc/profile.d/tomcat.sh
		export JAVA_HOME="/usr/lib/jvm/java"
		export CATALINA_HOME="/usr/share/tomcat"
		export CATALINA_BASE="$CATALINA_HOME"
		EOF
		source /etc/profile.d/tomcat.sh
		```
    5.  **允许管理性访问**
		```
		adminName='admin'
		adminPassword='a123456!'
		cat > $CATALINA_BASE/conf/tomcat-users.xml <<EOF
		<?xml version="1.0" encoding="utf-8"?>
		<tomcat-users>
		<role rolename="admin-gui"/>
		<role rolename="admin-script"/>
		<role rolename="manager-gui"/>
		<role rolename="manager-script"/>
		<role rolename="manager-jmx"/>
		<role rolename="manager-status"/>
		<user name="$adminName" password="$adminPassword" roles="admin-gui, \
		admin-script,manager-gui,manager-script,manager-jmx, \
		manager-status" />
		</tomcat-users>
		EOF
		systemctl restart tomcat
		```

*   **二进制安装 (推荐)**:
    1.  **步骤**:
        *   手动下载并安装指定版本的 JDK，并配置好 `JAVA_HOME` 环境变量。
        *   从 Apache Tomcat 官网下载所需版本的二进制压缩包（`.tar.gz`）。
        *   解压到指定目录（如 `/usr/local/tomcat10`）。
        *   配置 `CATALINA_HOME` 环境变量指向 Tomcat 的安装目录。
    2.  **特点**:
        *   **优点**: **版本灵活**，可以安装任意新版本；目录结构清晰，便于多实例部署和管理。
        *   **缺点**: 需要手动配置启动脚本或 `systemd` 服务。
    3.  **启停**: 通过执行安装目录下 `bin/` 文件夹中的脚本来管理：
        *   启动: `./startup.sh`
        *   停止: `./shutdown.sh`

### 三. Tomcat 管理

*   **核心配置文件**:
    *   **`server.xml` (最重要)**: Tomcat 的主配置文件，定义了整个服务器的结构。
        *   **`<Connector>`**: 配置连接器，如监听的端口（HTTP 默认 8080，AJP 默认 8009）、协议、超时等。
        *   **`<Engine>`**: 核心 Servlet 引擎。
        *   **`<Host>`**: 配置一个虚拟主机（网站）。
        *   **`<Context>`**: 配置一个 Web 应用程序。
    *   **`web.xml`**: 全局部署描述符，为所有部署在 Tomcat 上的应用提供默认配置。

*   **管理功能**:
    *   **虚拟目录 (Alias)**: 通过在 `server.xml` 的 `<Context>` 元素中进行配置，可以将一个 URL 路径映射到 `webapps` 目录之外的物理路径上。
    *   **虚拟主机**: 类似 Nginx，Tomcat 允许通过配置多个 `<Host>` 元素来在单个实例上运行多个网站。同样支持基于端口、IP 和域名（Host 主机头）的虚拟主机。
    *   **相同IP相同端口配置示例**:
		```xml
		# 编辑 $CATALINA_BASE/conf/server.xml,在Engine标签中添加
		<Host name="www.example1.com" appBase="/www.example1.com/webapps" 
		unpackWARs="true" autoDeploy="true">
			<Valve className="org.apache.catalina.valves.AccessLogValve"
			directory="logs"
				prefix="localhost_access_log" suffix=".txt"
				pattern="%h %l %u %t &quot;%r&quot; %s %b" />
		</Host>
		
		<Host name="www.example2.com" appBase="/www.example2.com/webapps" 
		unpackWARs="true" autoDeploy="true">
			<Valve className="org.apache.catalina.valves.AccessLogValve"
			directory="logs"
				prefix="localhost_access_log" suffix=".txt"
				pattern="%h %l %u %t &quot;%r&quot; %s %b" />
		</Host>
		```
    *   **安全加密 (SSL/TLS)**:
        *   **JSSE方式 (默认)**: 使用 Java 的安全套接字扩展。需要生成或获取一个 Java 密钥库文件（`.jks`），然后在 `server.xml` 的 `<Connector>` 中配置密钥库路径和密码来启用 HTTPS。
        *   **APR方式**: 使用 Apache Portable Runtime，可以调用 OpenSSL 库来处理加密，性能更高。需要安装 `tomcat-native` 库。

### 四. 综合项目实验

这部分展示了 Tomcat 在真实生产环境中的典型应用。

*   **架构**: **`Nginx + HTTPD + Tomcat`**
    *   **Nginx (前端)**: 暴露在公网，作为整个架构的入口。负责接收所有用户请求，处理**静态资源**（如图片、CSS），并作为**反向代理**和**负载均衡器**。
    *   **HTTPD (中间层)**: 部署 PHP 环境（如 WordPress 博客系统）。
    *   **Tomcat (后端)**: 部署 Java Web 应用（如 Java 商城系统）。
*   **工作流程**:
    1.  用户的浏览器请求域名。
    2.  DNS 将域名解析到 Nginx 服务器的 IP。
    3.  Nginx 接收请求：
        *   如果请求的是静态文件（如 `*.jpg`, `*.css`），Nginx 直接从本地返回。
        *   如果请求的是 PHP 页面（如访问 `blog.example.com`），Nginx 将请求**反向代理**给后端的 HTTPD 服务器。
        *   如果请求的是 Java 应用（如访问 `shop.example.com`），Nginx 将请求**反向代理**给后端的 Tomcat 服务器。
    4.  HTTPD 或 Tomcat 处理完动态请求后，将结果返回给 Nginx，再由 Nginx 返回给客户端。

这个实验完美地诠释了不同 Web 服务器的优势互补：**Nginx 的高并发和静态处理能力 + Tomcat 的 Java 应用执行能力 + HTTPD 成熟的 PHP 生态**。