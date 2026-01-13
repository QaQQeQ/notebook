
---

## 第16章 IP 子网划分

*   **IP 子网划分的需求背景**：
    *   **早期 IP 地址分配**：TCP/IP 网络使用分类（Classful）IP 地址分配，分为 A、B、C 等类别。这种两级结构（网络号和主机号）简单易用，但随着 Internet 规模扩大，暴露出严重弊端。
    *   **低效性**：
        *   **IP 地址资源浪费严重**：例如，一个需要 300 个 IP 地址的公司会分配一个 B 类地址（提供 65534 个地址），导致大量地址浪费。
        *   **IP 网络数量不敷使用**：随着物理网络数量增加，C 类地址很快耗尽，难以满足 Internet 发展需求。
        *   **业务扩展缺乏灵活性**：申请新网络地址耗时，阻碍业务快速部署。
        *   **无法应对 Internet 的爆炸式增长**：传统的两级划分无法有效管理日益增长的 IP 地址需求。
*   **IP 子网划分基础知识**：
    *   **子网划分的方法 (Subnetting)**：
        *   从主机号（host-number）部分借用若干位作为子网号（subnet-number），将两级 IP 地址变为三级结构（网络号、子网号、主机号）。
        *   子网划分是机构内部事务，外部网络无需了解内部子网结构，但路由器需要具备识别子网的能力。
    *   **子网掩码 (Subnet Mask)**：
        *   用于区分 IP 地址中的网络号、子网号和主机号部分。
        *   由一串连续的二进制 1 和一串连续的二进制 0 组成，1 对应网络号和子网号，0 对应主机号。
        *   通过将 IP 地址与子网掩码进行逐位逻辑与运算，可以得出子网地址。
        *   **默认掩码**：A 类地址默认 255.0.0.0，B 类地址默认 255.255.0.0，C 类地址默认 255.255.255.0。
*   **IP 子网划分的常用计算**：
    *   **计算子网内可用地址数**：
        *   如果主机号位数为 N，则可用主机地址数为 2^N - 2 个。
        *   减去 2 是因为主机号全 0 表示网络地址，主机号全 1 表示广播地址，这两个地址不能分配给主机使用。
    *   **根据主机地址数划分子网**：
        1.  确定每个子网所需的主机地址数 Y。
        2.  计算主机号位数 N：满足 2^N ≥ Y + 2 的最小 N 值。
        3.  计算子网掩码位数：32 - N。
        4.  根据子网掩码位数推导出子网掩码。
    *   **根据子网掩码计算子网数**：
        *   如果子网号位数为 M，则子网数为 2^M 个。
        *   （早期 RFC 规定子网号不能全 0 或全 1，则为 2^M - 2，但后期 RFC1812 取消了此限制）。
    *   **根据子网数划分子网**：
        1.  确定需要划分的子网数 X。
        2.  计算子网号位数 M：满足 2^M ≥ X 的最小 M 值。
        3.  计算子网掩码位数：分类网络号位数 + M。
        4.  根据子网掩码位数推导出子网掩码，并划分出具体子网。
*   **VLSM 和 CIDR**：
    *   **VLSM (Variable Length Subnet Mask，变长子网掩码)**：
        *   **产生背景**：传统的子网划分要求整个网络使用相同的子网掩码，导致主机地址资源浪费（例如，一个小分支只需要 40 个主机，却分配了 1022 个地址的子网）。
        *   **作用**：允许一个网络使用不同大小的子网掩码来划分多个子网，更有效地利用 IP 地址空间。
    *   **CIDR (Classless Inter-Domain Routing，无类域间路由)**：
        *   **产生背景**：Internet 路由表迅速膨胀，IPv4 地址即将耗尽。
        *   **作用**：
            *   消除了传统分类地址和子网划分的界限。
            *   通过聚合具有相同网络前缀的连续 IP 地址块（超网），减少路由条目，提高路由效率。
            *   使用“斜线表示法”（如 192.168.1.1/27），表示 IP 地址后跟网络前缀的长度。

---
## 第17章 DNS

*   **DNS 域名系统概述**：
    *   **主机名与 IP 地址映射需求**：IP 地址（点分十进制数）难以记忆，需要一种将易于记忆的主机名（域名）映射到 IP 地址的方法。
    *   **Hosts 文件**：
        *   早期用于实现主机名与 IP 地址映射的本地文件。
        *   优点是查找响应快，但缺点是需要手动更新，且随着网络规模扩大，维护困难。
    *   **DNS 简介**：
        *   **定义**：DNS（Domain Name System，域名系统）是一种用于 TCP/IP 应用程序的分布式数据库，提供域名与 IP 地址之间的转换服务。
        *   **工作模式**：采用客户端/服务器模式，客户端发出查询请求，DNS 服务器负责解析。
        *   **传输协议**：使用 TCP 或 UDP 协议，DNS 服务器监听 53 端口。
        *   **结构**：具有树状层次结构，联机分布式数据库系统，解决了传统集中式设计的单点故障、性能瓶颈、维护困难等问题。
*   **DNS 域名解析过程**：
    *   **本地域名服务器**：通常离客户端较近，由运营商或大型机构自行管理。客户端首先向其发送 DNS 查询请求。
    *   **根域名服务器**：管理顶级域，不直接解析域名，但知道相关顶级域名服务器的地址。
    *   **域名解析完整过程**：
        1.  客户端向本地域名服务器查询 `www.h3c.com.cn` 的 IP 地址。
        2.  本地域名服务器查询本地数据库，若无记录，则向根域名服务器发送查询。
        3.  根域名服务器返回 .cn 顶级域名服务器的 IP 地址给本地域名服务器。
        4.  本地域名服务器向 .cn 顶级域名服务器查询 `www.h3c.com.cn` 的 IP 地址。
        5.  .cn 顶级域名服务器返回 .com.cn 域名服务器的 IP 地址给本地域名服务器。
        6.  本地域名服务器向 .com.cn 域名服务器查询 `www.h3c.com.cn` 的 IP 地址。
        7.  .com.cn 域名服务器返回 `h3c.com.cn` 域名服务器的 IP 地址给本地域名服务器。
        8.  本地域名服务器向 `h3c.com.cn` 域名服务器查询 `www.h3c.com.cn` 的 IP 地址。
        9.  `h3c.com.cn` 域名服务器返回 `www.h3c.com.cn` 对应的 IP 地址给本地域名服务器。
        10. 本地域名服务器将最终的 IP 地址返回给客户端。
*   **DNS 查询方式**：
    *   **递归查询**：DNS 服务器接收到递归查询请求后，会负责处理所有后续查询，直到找到最终结果并返回给请求方。客户端只需等待最终结果。
    *   **迭代查询**：DNS 服务器接收到迭代查询请求后，如果本地无法解析，会返回一个可能知道查询结果的 DNS 服务器地址给请求方，由请求方继续向新的服务器发送查询请求，直到找到最终结果。
*   **DNS 反向查询**：
    *   **定义**：根据已知的 IP 地址查找对应的主机名。
    *   **实现**：通过特殊的 **in-addr.arpa** 反向查询域来实现，IP 地址在查询时会倒序排列。
*   **H3C 设备 DNS 特性及配置**：
    *   **H3C 设备 DNS 功能**：
        *   **静态域名解析**：手动建立域名与 IP 地址的对应关系。
        *   **动态域名解析**：设备查询 DNS 域名服务器完成解析（支持域名后缀列表）。
        *   **DNS 代理 (DNS Proxy)**：路由器作为 DNS 客户端和 DNS 服务器之间的中继，转发 DNS 请求和应答报文，简化网络管理。
    *   **配置静态和动态域名解析**：
        *   配置静态解析表项：`ip host hostname ip-address [ vpn-instance vpn-instance-name ]`。
        *   配置指定 DNS 服务器：`dns server ip-address [ vpn-instance vpn-instance-name ]`。
        *   配置域名后缀：`dns domain domain-name [ vpn-instance vpn-instance-name ]`。
    *   **配置 DNS 代理**：
        1.  启用 DNS 代理功能：`dns proxy enable`。
        2.  配置指定 DNS 服务器：`dns server ip-address [ vpn-instance vpn-instance-name ]`。
    *   **域名解析显示及维护**：
        *   显示静态域名解析表：`display ip host [ ip | ipv6 ] [ vpn-instance vpn-instance-name ]`。
        *   显示 DNS 服务器：`display dns server [ dynamic ] [ vpn-instance vpn-instance-name ]`。
        *   显示域名后缀：`display dns domain [ dynamic ] [ vpn-instance vpn-instance-name ]`。

---
## 第18章 文件传输协议

*   **FTP 协议介绍**：
    *   **定义**：FTP (File Transfer Protocol，文件传输协议) 是互联网上广泛使用的文件传输协议，基于 TCP 协议。
    *   **工作模式**：客户端/服务器模式，支持用户登录认证和访问权限设置。
    *   **双 TCP 连接**：
        *   **控制连接**：使用 TCP 端口 21，用于传输 FTP 控制命令和应答信息，在整个 FTP 会话期间保持打开。
        *   **数据连接**：使用 TCP 端口 20 (主动模式) 或客户端随机端口 (被动模式)，用于传输实际数据（文件上传、下载、文件列表），数据传输完成后关闭。
    *   **文件传输模式**：
        *   **ASCII 模式**：默认模式，适用于传输文本文件。发送方将本地文件转换为标准 ASCII 码传输，接收方再转换成本地格式。
        *   **二进制流模式 (图像文件传输模式)**：适用于传输程序文件或其他非文本文件。发送方按比特流方式传输，不进行任何转换。
    *   **数据传输方式**：
        *   **主动方式 (PORT)**：服务器主动发起数据连接到客户端的临时端口。客户端若在防火墙内部，可能因防火墙阻断随机端口而失败。
        *   **被动方式 (PASV)**：客户端主动发起数据连接到服务器的临时端口。解决了主动模式下客户端防火墙的问题。
*   **TFTP 协议介绍**：
    *   **定义**：TFTP (Trivial File Transfer Protocol，简单文件传输协议) 是一种简单的文件传输协议，基于 UDP 协议。
    *   **特点**：
        *   客户端/服务器模式。
        *   适用于客户端和服务器之间不需要复杂交互的环境。
        *   使用 UDP 端口 69。
        *   只提供简单的文件上传、下载功能。
        *   不支持用户登录认证、访问权限设置和目录列表功能。
        *   通过超时重传机制确保数据传输的可靠性。
        *   传输由客户端发起。
    *   **文件传输过程**：
        *   文件被视为多个连续文件块。
        *   客户端发送读请求或写请求，服务器回应数据或确认。
        *   每个数据报文包含一个文件块和块编号，每次发送后等待确认。
        *   文件块大小通常为 512 字节，最后一个文件块小于 512 字节表示传输结束。
    *   **文件传输模式**：
        *   **netascii 模式**：对应 FTP 的 ASCII 模式，传输文本文件。
        *   **octet 模式**：对应 FTP 的二进制流模式，传输程序文件。
*   **配置 FTP 与 TFTP**：
    *   **FTP 客户端配置**：
        *   在用户视图下登录远程 FTP 服务器：`ftp ftp-server [ service-port ] [ vpn-instance vpn-instance-name ] [ dscp dscp-value | source { interface interface-name | interface-type interface-number | ip source-ip-address } ]`。
        *   常用命令：`ls` (查看目录/文件)、`get` (下载文件)、`put` (上传文件)、`bye` (断开连接)。
    *   **FTP 服务器端配置**：
        1.  在系统视图下启动 FTP 服务器功能：`ftp server enable`。
        2.  创建本地用户并设置密码和服务类型：`local-user user-name [ class { manage | network } ]`，`password { hash | simple } password`，`service-type { ftp | ssh | telnet | terminal }`。
    *   **TFTP 客户端配置**：
        *   在用户视图下执行 TFTP 文件传输命令：`tftp tftp-server { get | put | sget } source-filename [ destination-filename ] [ vpn-instance vpn-instance-name ] [ dscp dscp-value | source { interface interface-type interface-number | ip source-ip-address } ]`。
        *   `get`：普通下载。
        *   `put`：上传。
        *   `sget`：安全下载（先保存到内存，再写入存储设备）。

---
## 第19章 DHCP

*   **DHCP 简介**：
    *   **定义**：DHCP (Dynamic Host Configuration Protocol，动态主机配置协议) 是 BOOTP 协议的增强版本。
    *   **作用**：为局域网中的计算机自动分配 TCP/IP 配置信息，包括 IP 地址、子网掩码、网关和 DNS 服务器地址等。
    *   **优点**：终端主机无需手动配置，网络维护方便，减少 IP 地址配置错误。
    *   **协议特点**：
        *   **即插即用**：客户端无需配置即可获得 IP 地址及相关参数。
        *   **统一管理**：所有 IP 地址及参数由 DHCP 服务器统一管理和分配。
        *   **使用效率高**：通过 IP 地址租期管理，提高 IP 地址利用率。
        *   **可跨网段实现**：通过 DHCP 中继技术，使不同子网的客户端和 DHCP 服务器之间能进行报文交互。
    *   **系统组成**：
        *   **DHCP 服务器**：提供 DHCP 服务，负责集中管理 IP 配置信息。
        *   **DHCP 中继 (DHCP Relay)**：通常为路由器或三层交换机，转发跨网段的 DHCP 报文。
        *   **DHCP 客户端**：需要动态获取 IP 地址的主机或网络设备。
    *   **报文封装**：DHCP 报文使用 UDP 封装，服务器监听端口 67，客户端监听端口 68。
*   **DHCP 地址分配方式**：
    *   **手工分配**：管理员为特定主机（如服务器、打印机）绑定固定的 IP 地址，地址不会过期。
    *   **自动分配**：DHCP 服务器为连接到网络的某些主机分配 IP 地址，并长期由该主机使用。
    *   **动态分配**：DHCP 服务器为客户端指定 IP 地址，并规定租用期限。租期到期后，客户端需重新申请，地址可被回收。这是最常用的分配方式。
*   **IP 地址动态获取过程 (DORA 过程)**：
    1.  **发现 (Discover)**：DHCP 客户端以广播方式发送 `DHCP-DISCOVER` 报文，寻找 DHCP 服务器。
    2.  **提供 (Offer)**：DHCP 服务器接收到 `DHCP-DISCOVER` 报文后，选出一个 IP 地址及其他参数，以广播方式发送 `DHCP-OFFER` 报文给客户端。
    3.  **选择 (Request)**：DHCP 客户端接收到 `DHCP-OFFER` 报文后，选择第一个收到的 `DHCP-OFFER`，以广播方式发送 `DHCP-REQUEST` 报文，包含选择的 IP 地址。
    4.  **确认 (ACK)**：DHCP 服务器接收到 `DHCP-REQUEST` 报文后，确认将地址分配给客户端，发送 `DHCP-ACK` 报文。若无法分配，则发送 `DHCP-NAK` 报文。
*   **DHCP 租约更新**：
    *   当租约期限达到 50% 时，客户端会单播发送 `DHCP-REQUEST` 报文请求续约。
    *   若续约成功，服务器回应 `DHCP-ACK`。若失败，服务器回应 `DHCP-NAK`。
    *   若 50% 续约失败，客户端会在租约期限达到 7/8 时再次广播发送 `DHCP-REQUEST` 报文尝试续约。
    *   若租约期满仍未续约成功，客户端会放弃当前 IP 地址，重新开始 DORA 过程。
*   **DHCP 中继介绍**：
    *   **产生背景**：DHCP 广播报文无法跨越子网，若每个子网都部署 DHCP 服务器不经济。
    *   **作用**：DHCP 中继设备（路由器或三层交换机）接收客户端的广播 DHCP 报文，将其单播转发给指定 DHCP 服务器，并转发服务器回应给客户端。
    *   **工作原理**：
        1.  DHCP 中继收到客户端的 `DHCP-DISCOVER` 或 `DHCP-REQUEST` 广播报文。
        2.  DHCP 中继根据配置将报文单播转发给指定的 DHCP 服务器。
        3.  DHCP 服务器分配 IP 地址后，将配置信息单播发送给 DHCP 中继。
        4.  DHCP 中继将配置信息广播发送给客户端。
*   **DHCP 服务器配置**：
    *   **DHCP 服务器基本配置**：
        1.  启用 DHCP 服务：`dhcp enable`。
        2.  创建 DHCP 地址池：`dhcp server ip-pool pool-name`。
        3.  配置动态分配的 IP 地址范围：`network network-address [ mask-length | mask mask ]`。
        4.  配置为 DHCP 客户端分配的网关地址：`gateway-list ip-address &<1-8>`。
    *   **DHCP 服务器可选配置**：
        1.  配置为 DHCP 客户端分配的 DNS 服务器地址：`dns-list ip-address&<1-8>`。
        2.  配置 DHCP 地址池中不参与自动分配的 IP 地址：`dhcp server forbidden-ip start-ip-address [ end-ip-address ]`。
        3.  配置动态分配的 IP 地址租用期限：`expired { day day [ hour hour [ minute minute [ second second ] ] ] | unlimited }`。
    *   **DHCP 服务器的显示及维护**：
        *   显示 DHCP 地址池信息：`display dhcp server pool [ pool-name ]`。
        *   显示 DHCP 地址池的可用地址信息：`display dhcp server free-ip`。
        *   显示 DHCP 服务器的统计信息：`display dhcp server statistics [ pool pool-name ]`。
*   **DHCP 中继配置**：
    *   **DHCP 中继基本配置**：
        1.  启用 DHCP 服务：`dhcp enable`。
        2.  在接口视图下指定 DHCP 服务器的地址：`dhcp relay server-address ip-address`。
        3.  配置接口工作在 DHCP 中继模式：`dhcp select relay`。
    *   **DHCP 中继的显示及维护**：
        *   显示 DHCP 中继服务器地址信息：`display dhcp relay server-address [ interface interface-type interface-number ]`。
        *   显示 DHCP 中继的用户地址表项信息：`display dhcp relay client-information`。
        *   显示 DHCP 中继的相关报文统计信息：`display dhcp relay statistics [ interface interface-type interface-number ]`。

---
## 第20章 IPv6 基础

*   **IPv6 的特点**：
    *   **IPv4 的不足**：可用地址日益缺乏、配置不够简便、缺乏安全性和 QoS 支持。
    *   **IPv6 的优点**：
        *   **几乎无限的地址空间**：从 IPv4 的 32 位扩展到 128 位，提供了巨大的地址空间（3.4 x 10^38 个地址）。
        *   **终端用户无须任何配置**：支持地址自动配置。
        *   **设计时增强了安全性和 QoS**。
        *   取消了广播，增加了任播。
*   **IPv6 地址**：
    *   **IPv6 地址表示方式**：
        *   **冒号十六进制表示法**：128 位地址分成 8 段，每段 16 位，用十六进制表示，段之间用冒号分隔（如 2001:0410:0000:0001:0000:0000:0000:45FF）。
        *   **压缩表示法**：
            *   **段内前导 0 压缩**：每段中的前导 0 可以省略（如 0410 压缩为 410）。
            *   **全 0 段压缩**：一个或多个连续的全 0 段可以用 `::`（双冒号）表示，但一个地址中只允许有一个 `::`。
    *   **IPv6 地址构成**：
        *   取消了 IPv4 的网络号、主机号和子网掩码概念。
        *   取消了 A、B、C 类地址分类。
        *   由**前缀**、**接口标识符**和**前缀长度**组成。
            *   **前缀**：类似于 IPv4 的网络部分，标识地址所属的网络。
            *   **接口标识符**：类似于 IPv4 的主机部分，标识网络中的具体接口。
            *   **前缀长度**：表示前缀所占的位数。
    *   **IPv6 地址分类**：
        *   **单播地址 (Unicast Address)**：唯一标识一个接口，数据报文发送到此地址只被一个接口接收。
            *   **链路本地地址 (Link-Local Address)**：用于链路本地节点间的通信，前缀 FE80::/10，不会被路由器转发。
            *   **全球单播地址 (Global Unicast Address)**：全球唯一的地址，可在 Internet 上路由，前缀 2000::/3。
        *   **组播地址 (Multicast Address)**：标识一组接口，数据报文发送到此地址被组内所有接口接收。前缀 FF00::/8。
        *   **任播地址 (Anycast Address)**：标识一组接口，数据报文发送到此地址只被组内距离源节点最近的一个接口接收。从单播地址空间中分配。
    *   **常用的 IPv6 地址类型及格式**：
        *   未指定地址 `::/128`。
        *   环回地址 `::1/128`。
        *   链路本地地址 `FE80::/10`。
        *   全球单播地址 `2000::/3`。
        *   组播地址 `FF00::/8`。
*   **邻居发现协议 (NDP)**：
    *   **定义**：IPv6 中一个非常重要的协议，实现地址解析、路由器发现/前缀发现、地址自动配置、地址重复检测等功能。
    *   **IPv6 地址解析**：
        *   通过节点之间交互**邻居请求消息 (Neighbor Solicitation)** 和 **邻居通告消息 (Neighbor Advertisement)** 来获取同一链路上邻居节点的链路层地址。
        *   邻居请求消息以组播方式发送，邻居通告消息以单播方式返回。
    *   **IPv6 地址自动配置 (Stateless Address Autoconfiguration, SLAAC)**：
        *   **路由器发现/前缀发现**：主机通过发送**路由器请求消息 (Router Solicitation)**，接收**路由器通告消息 (Router Advertisement)**，获取邻居路由器及网络前缀和其他配置参数。
        *   **地址自动生成**：主机根据路由器通告消息中的网络前缀，结合自身的 EUI-64 格式接口标识符（由 MAC 地址自动生成）自动生成全球单播地址。
            *   **EUI-64 格式**：将 48 位的 MAC 地址在中间插入 FFFE (1111111111111110)，并将第 7 位 (U/L 位) 设置为 1，生成 64 位的接口标识符。
*   **IPv6 地址配置**：
    *   **IPv6 地址配置命令**：
        *   **手工指定接口的全球单播地址**：`ipv6 address { ipv6-address prefix-length | ipv6-address/prefix-length }`。
        *   **配置接口使用无状态自动配置 IPv6 地址**：`ipv6 address auto`。
        *   **手工指定接口的链路本地地址**：`ipv6 address ipv6-address link-local`。
        *   **配置接口自动生成链路本地地址**：`ipv6 address auto link-local`。
    *   **IPv6 地址显示与维护**：
        *   `display ipv6 interface [ interface-type [ interface-number ] ] [ brief ]`：查看接口的 IPv6 地址信息。