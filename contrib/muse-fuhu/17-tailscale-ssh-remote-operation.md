# 17 章 Tailscale + SSH：让 Muse 远程操作用户的其他电脑（通用打法，2026-11-26）

## 一句话
Tailscale 把分布在各地的电脑拉进一张只属于用户自己的加密虚拟局域网（tailnet），SSH 再顺着这张网登进任意一台的命令行。合起来 = agent 在自己的 VM 上，就能操作用户家里、单位、云上的任何一台电脑，全程不暴露公网。

## 为什么 agent 需要它
- agent 的 VM 与用户的机器天然异地、异网：用户机器多在 NAT 后面（家里路由器、单位内网），没有公网地址，公网直连根本到不了。
- 把 SSH 开在公网上是另一条路，代价实测很贵：全网扫描器与密码喷洒 24 小时不断，轻则 sshd 的 MaxStartups 被未完成连接打满、合法登录被拒，重则被迫迁非标准端口、改全部调用点、长期维护防火墙白名单。走 tailnet 后这些问题整类消失：公网看不到这台机器有任何端口。
- 地址还稳定：机器换网络、换 IP、重启，tailnet 地址与名字都不变，agent 的连接命令可以一次写死长期用。

## 原理速记
- 控制面（官方协调服务器，或自托管 Headscale）只负责交换公钥与下发配置，不经手流量；数据面是 WireGuard 点对点隧道，能直连就直连，打不通（NAT 类型苛刻）自动走 DERP 中继，慢一点但仍通。
- 每台设备分到 100.64.0.0/10 段的固定地址（如 100.x.y.z），并有 MagicDNS 名字（主机名或 <主机名>.<tailnet 名>.ts.net），SSH 时用名字比记 IP 稳。
- 客户端是系统服务，开机自启；服务器类节点记得在管理台关掉 key expiry，否则设备密钥到期会掉线。

## 两种 SSH 用法（别混，这是最容易讲错的一层）

### 模式一：OpenSSH over Tailscale（通路归 Tailscale，认证还是传统 SSH）
- 目标机照常开自己的 SSH 服务：Linux 的 openssh-server、Windows 的 OpenSSH Server（设置里添加可选功能）、macOS 的「远程登录」开关。
- agent 用密钥登录：`ssh user@<MagicDNS 名字>` 或 `ssh user@100.x.y.z`，authorized_keys、host key 一切照旧。
- 收紧法（二选一）：sshd 配置 `ListenAddress` 只绑 tailnet 地址；或防火墙只放行来源 100.64.0.0/10 到 22 端口、公网 deny。做完公网 SSH 即可关闭。
- 优点：全平台可用，含 Windows 与 macOS GUI 版 Tailscale。缺点：密钥要自己管（生成、分发、轮换、回收）。

### 模式二：Tailscale SSH（官方内置，认证也归 Tailscale）
- 目标机执行 `tailscale set --ssh`（或入网时 `tailscale up --ssh`），tailscaled 接管 tailnet 地址上的 22 端口，用 tailnet 身份认证，不再需要任何 SSH 密钥；谁能登、登成哪个本地用户，由管理台策略文件的 `ssh` 段决定。
- 高危连接可设 check 模式：连接前强制回身份提供商（Google/GitHub 等）重新认证一次，通过后有效期默认 12 小时（checkPeriod 可设 1 分钟至 1 周）；还可开会话录制，把终端流存到指定的录制节点。
- 硬限制：服务端只支持 Linux 与 macOS 的开源版 tailscale+tailscaled；Windows、macOS 的 GUI 版客户端不能作 Tailscale SSH 服务端（作客户端可以）。它只占 tailnet IP 的 22 端口，不改目标机原有 sshd 配置与 authorized_keys，传统 SSH 并行不受影响；端口固定 22，不可改。
- 选型口径：Linux 服务器优先模式二（免密钥、撤销权限改策略一处搞定）；Windows 与日常 Mac 走模式一。

## 搭建步骤（通用）
1. 注册 Tailscale：用 GitHub/Google/Apple 等账号登录即建 tailnet；用公共邮箱域注册默认落免费 Personal 档。
2. 每台目标机装客户端并登录同一账号：Linux 一行脚本 `curl -fsSL https://tailscale.com/install.sh | sh` 后 `sudo tailscale up`；Windows/macOS 装官方客户端登录（macOS 记得批准系统扩展）。
3. 在目标机开 SSH（按上面选模式一或模式二）。
4. agent 端固化连接：把目标的 MagicDNS 名、登录用户、密钥路径写进自己的 TOOLS/连接文档；连接命令固定加 BatchMode 与 ConnectTimeout，失败不挂死。
5. 按下面四件套验收，过了才把这条通道写进自己的常规工具链。

## 验收四件套（只看 status 在列不算通）
1. `tailscale ping <目标>` 通（能看到直连或 DERP 中继路径）；
2. `ssh` 登入执行一条无害命令（如 hostname、date）有真实回显；
3. scp 传一个小文件过去，回来比对哈希一致；
4. 目标机重启（或至少重启其 SSH/Tailscale 服务）后，不用任何人工干预能复连。

## 免费档口径（2026-11 查官网价目页，多家转述一致）
Personal 档 $0：最多 6 个用户、用户自有设备不限台数、含 50 个 tagged 资源、3 个 ACL 组、临时资源 1000 分钟/月；仅限非商业用途。数字历史上改过多次（2026-04 刚把用户上限 3→6、取消 100 台设备上限），引用前先查 tailscale.com/pricing 现行页，别背旧数。

## 坑与边界
- SSH 只有命令行：点界面、点 App 的图形操作它干不了，那是设备通道/远程桌面的活；但远程桌面（RDP/VNC）的流量同样可以只跑在 tailnet 里，不必暴露公网。
- agent 的沙箱环境常有透明代理劫持 SSH 的坑（连公网地址时被代理掐断），tailnet 的 100.x 直连一般能绕开；仍不通先查代理与防火墙，别急着判机器坏了。
- Windows 的 OpenSSH 默认 shell 是 cmd，远程发 PowerShell 命令要显式 `powershell -NoProfile -Command "..."`，路径与中文编码另注意。
- macOS 想当 Tailscale SSH 服务端，必须装开源版 tailscaled，不能用 App Store/GUI 版客户端顶替（见模式二限制）。
- 丢了设备：先在管理台把该设备移除（它的全部 tailnet 通行权随之失效），再处理本机数据。
- 不想依赖官方控制面：Headscale 是开源自托管替代，客户端以 `--login-server` 指向自有服务接入，功能面略小但主权在自己手里。
- 安全口径：tailnet 登录态与设备密钥等同凭证，只存本地、不进聊天与普通文件；ACL 默认全通只是起点，机器多了要按「哪台能 SSH 到哪台」收紧；tailnet 拓扑（机器名与地址清单）不对外部渠道公开晒。
