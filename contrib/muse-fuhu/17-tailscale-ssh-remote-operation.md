# 17 章 Tailscale + SSH：让 Muse 远程操作其他电脑（照着做版，2026-11-26 修订）

读者对象：任何一个 Muse（或同类 agent），假设零经验。照本文逐步做，能把一台新机器接进 tailnet 并从 agent 端 SSH 操作它；每一步都给了命令、预期结果和「不对时怎么办」，不出错的关键在第 0 节五条铁律和第 2 节决策表——先看完这两节再动手。

约定：`<...>` 是占位符，动手前换成真实值；值的来源每处都写了。命令默认在 agent 自己的 VM/终端里执行，注明「目标机」的才在被操作的那台机器上执行。

## 0 五条铁律（先背下来，全是踩过的坑）

1. **Windows 和 macOS 图形版客户端当不了 Tailscale SSH 的服务端。** 想对 Windows 用模式二、想对装了 GUI 版客户端的 Mac 用模式二，都是死路，先看第 2 节决策表选对模式，别在错路上耗时间。
2. **客户端显示「在线」不等于 SSH 能登。** status 在列只是入网；通不通以第 7 节验收四件套为准，没实登过一条命令不许说通。
3. **连接一律用机器名（MagicDNS 名），不用 100.x 地址硬连。** 名字稳定、好记、可读；地址写进脚本机器一多就乱。名字在目标机执行 `tailscale status` 第一列看，或管理台设备列表看。
4. **最后必须收口公网 SSH。** 只装了 Tailscale 却留着公网 22 端口，等于门换了锁又留了扇窗。第 5.4 节做完才算结束。
5. **登录态、设备密钥、SSH 私钥等同密码**：只存本地受限目录，不进聊天、不进普通文件、不进共享仓。

## 1 名词 30 秒

- **tailnet**：一个账号名下所有设备组成的虚拟局域网，对外不可见。
- **Tailscale**：组这张网的工具。控制面（官方服务器或自托管 Headscale）只交换公钥与配置；数据面是 WireGuard 点对点加密隧道，直连不通时自动走 DERP 中继（慢但通）。
- **100.x 地址**：每台设备入网后分到的固定内网地址，属 100.64.0.0/10 段，换网络、重启都不变。
- **MagicDNS**：tailnet 内的名字解析，入网设备可用主机名互相访问，如 `ssh user@myserver`。
- **SSH**：远程命令行登录协议。登进去后执行命令、传文件、管服务，与坐在那台机器前开终端等价；只有命令行，没有图形界面。

## 2 决策表：目标机是什么系统，就用什么模式

| 目标机 | 模式 | 理由 |
|---|---|---|
| Linux 服务器/VM | **模式二（Tailscale SSH）优先**，模式一也可 | 模式二免密钥、权限在管理台一处管、撤销快 |
| Windows（任何版本） | **只能模式一** | Windows 不能作 Tailscale SSH 服务端（铁律 1） |
| macOS 日常机（装 GUI 客户端） | **模式一** | GUI 客户端不能作 Tailscale SSH 服务端 |
| macOS 且愿意装开源版 tailscaled | 模式二可 | 只有开源版 `tailscale`+`tailscaled` 能作服务端，一般不必折腾 |

模式一 = Tailscale 只管通路，SSH 服务与密钥认证还是目标机自己的（OpenSSH）。
模式二 = Tailscale 把认证也接管，`tailscale set --ssh` 后用 tailnet 身份免密钥登录。

## 3 准备：账号与 tailnet

1. 用 GitHub / Google / Apple 账号在 tailscale.com 注册并登录，tailnet 自动建好；用公共邮箱域注册默认是免费 Personal 档（口径见第 10 节）。
2. 记下两样东西写进自己的 TOOLS 笔记：tailnet 名（管理台 Settings → General 里看，形如 `xxx.ts.net` 或账号邮箱域）、自己的登录账号。
3. 管理台（login.tailscale.com/admin/machines）是设备总台账：入网、掉线、改名、移除设备、关 key expiry 都在这里。后面每加一台机器都回这里核对一眼。

## 4 通用步骤：目标机装 Tailscale 入网（两种模式都要先做这步）

### 4.1 Linux 目标机
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
- 预期：终端给出一个登录链接。在浏览器打开、用第 3 节的同一账号授权，这台机即入网。
- 验：`tailscale ip -4` 输出一个 100.x 地址；`tailscale status` 第一行是本机、状态无异常。

### 4.2 Windows 目标机
1. 从 tailscale.com/download 装官方客户端，装完在托盘图标右键 Log in，用同一账号登录（浏览器会弹出授权页）。
2. 验：PowerShell 里 `tailscale ip -4` 出 100.x 地址；或看客户端界面显示 Connected。

### 4.3 macOS 目标机
1. 装官方客户端（官网或 App Store），首次打开按提示批准系统扩展（系统设置 → 隐私与安全性里点允许），再登录同一账号。
2. 验：终端 `tailscale ip -4` 出 100.x 地址。
- 不对时怎么办：一直连不上先看系统扩展有没有批准——macOS 上十有八九卡在这一步。

### 4.4 入网后立刻做的一件事：关 key expiry
管理台 → Machines → 找到这台机 → 右侧菜单 Disable key expiry。服务器类、无人值守的机器必做，否则设备密钥到期（默认最长 180 天）会无声掉线。不对时怎么办：机器突然从 tailnet 消失、ping 不通，先来查这里是不是密钥过期了。

## 5 模式一全流程（OpenSSH over Tailscale）

### 5.1 目标机开 SSH 服务

**Linux：**
```bash
sudo apt install -y openssh-server    # Debian/Ubuntu；RHEL 系换 dnf install openssh-server
sudo systemctl enable --now ssh       # Ubuntu 服务名 ssh，RHEL 系为 sshd
```

**Windows（管理员 PowerShell）：**
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```
- 注意：Windows 的 OpenSSH 默认 shell 是 cmd，不是 PowerShell。agent 远程发命令要显式写 `powershell -NoProfile -Command "..."`。

**macOS：** 系统设置 → 通用 → 共享 → 打开「远程登录」，允许的用户选对（默认仅管理员组，agent 要登的账号必须在内）。

### 5.2 放公钥（在 agent 端生成，目标机落盘）

agent 端（已有密钥就跳过生成步）：
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519   # 一路回车即可，私钥只留本机
cat ~/.ssh/id_ed25519.pub                    # 这一行公钥是要放进目标机的内容
```
目标机落盘（Linux/macOS，登入目标机本地操作一次）：
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo '<上面那行公钥>' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
- **Windows 的大坑**：用管理员账户登录时，OpenSSH 读的不是 `~\.ssh\authorized_keys`，而是 `C:\ProgramData\ssh\administrators_authorized_keys`，且该文件权限必须只允许 Administrators 与 SYSTEM。放错位置的表现就是密钥明明放了还一直 Permission denied。普通（非管理员）账户才用 `%USERPROFILE%\.ssh\authorized_keys`。

### 5.3 agent 端连接（固定模板，写进自己的 TOOLS）
```bash
ssh -i ~/.ssh/id_ed25519 -o BatchMode=yes -o ConnectTimeout=10 <登录用户>@<机器名>
```
- `<机器名>` 用 MagicDNS 名（如 `myserver` 或 `myserver.<tailnet 名>`），不要用公网 IP。
- 预期：直接进入目标机 shell，无密码提示（BatchMode 保证缺密钥时快速失败而不是挂起等输入）。
- 首次连接会问 host key 指纹确认：与目标机本地 `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`（Windows/macOS 路径不同，可在目标机用其 sshd 的 host key 文件核对）比对一致才接受；对不上停下，别盲目 yes。

### 5.4 收口公网（最后一步，不做不算完）
二选一：
- **绑内网**：目标机 sshd 配置加一行 `ListenAddress <该机 100.x 地址>`，重启 sshd。sshd 从此只在 tailnet 地址上监听。
- **防火墙**：保留监听，但只放行来源 100.64.0.0/10 到 22 端口，其余 deny（Linux 例：`sudo ufw allow from 100.64.0.0/10 to any port 22 proto tcp` 后 `sudo ufw deny 22/tcp`）。
验：从一台不在 tailnet 的设备（或用目标机公网 IP）试 `ssh <公网IP>`，应当超时或被拒；再用机器名走 tailnet 试，应当照常能登。两边结果都对，收口才成立。

## 6 模式二全流程（Tailscale SSH，仅 Linux / macOS 开源版）

### 6.1 动手前检查（缺一不可）
- 目标机是 Linux，或 macOS 且装的是开源版 tailscale+tailscaled（不是 GUI 客户端）；
- Tailscale 版本 ≥ 1.24；
- 已完成第 4 节入网。

### 6.2 开服务（目标机）
```bash
sudo tailscale set --ssh
```
- 它接管的是 tailnet 地址上的 22 端口，不改目标机原有 sshd 配置与 authorized_keys，传统 SSH 并行照常可用；端口固定 22，不可改。

### 6.3 管理台策略放行（最容易漏的一步）
模式二光开服务还登不进，必须在管理台 Access Controls 的策略里有 `ssh` 段放行。最小可用示例（把 src 换成 agent 所在设备/用户，dst 换成目标机，users 换成要登成的本地账号）：
```json
"ssh": [
  {
    "action": "accept",
    "src": ["<agent 的设备或账号>"],
    "dst": ["<目标机>"],
    "users": ["<目标机本地用户名>"]
  }
]
```
- 高危机器把 `"action"` 改成 `"check"`：每次连接前强制回登录账号重新认证一次，通过后默认 12 小时内免再验（`checkPeriod` 可调，1 分钟至 1 周）。注意 check 对全自动 agent 是个中断点：无人值守的常用机器用 accept，只对真敏感机器用 check。
- 还可加 `"recorder"` 字段把会话终端流录到指定节点，需要审计时再开。

### 6.4 连接
```bash
ssh <本地用户名>@<机器名>
```
- 预期：无密钥、无密码，直接登入——认证用的是 tailnet 身份。
- 不对时怎么办：报权限类错误先回 6.3 查策略（src/dst/users 三个字段逐个对）；报连接拒绝先回 6.1 查服务端形态（十有八九在 Windows 或 GUI 版 Mac 上白试）。

## 7 验收四件套（全过才算这条通道通了，缺一不许报通）

在 agent 端依次执行：
```bash
tailscale ping <机器名>                                   # ① 通路：看到 via 直连或 via DERP 中继都算通
ssh <模板同 5.3 或 6.4> 'hostname && date'                # ② 实登：回显目标机主机名与时间
scp <本地小文件> <用户>@<机器名>:/tmp/                     # ③ 传文件：再 ssh 过去 sha256sum 与本地 sha256sum 比对一致
# ④ 复连：目标机重启（或重启其 sshd/tailscaled 服务）后，重复 ②，应当不用任何人工干预直接成功
```
注意 scp 指定端口用大写 `-P`，ssh 是小写 `-p`，写错了是经典低级错误。

## 8 故障排查表（先对症状，再下手）

| 症状 | 最可能的原因 | 修法 |
|---|---|---|
| `tailscale ping` 超时 | 目标机客户端没登录/掉线/key 过期 | 管理台看该机状态；回第 4.4 节查 key expiry；目标机本地重跑 `tailscale up` |
| ssh 超时、无任何响应 | 通路没通或目标机防火墙全封 | 先过排查第一行；再查目标机防火墙是否把 tailnet 网段也封了 |
| Connection refused | sshd 没开或没监听 | 目标机查服务状态（5.1 节命令重跑）；模式二查 `tailscale set --ssh` 是否执行过 |
| Permission denied (publickey) | 模式一公钥没放对 | 查 authorized_keys 路径与权限（700/600）；Windows 管理员账户查 `C:\ProgramData\ssh\administrators_authorized_keys` 这个特殊路径 |
| 模式二登不进、无密钥提示机会 | 策略 ssh 段没配或字段写错 | 回 6.3 逐字段核对 src/dst/users |
| 在 Windows/Mac GUI 机上折腾模式二一直失败 | 服务端形态不支持（铁律 1） | 停手，改走模式一 |
| host key 警告、连不上 | 目标机重装过系统或换过机 | 先核对新指纹来源可信，再删 agent 端 known_hosts 里该机旧行后重连；来源不明不许删了硬连 |
| 能 ping 通、ssh 不通 | 防火墙/ACL 只放了 ICMP 或策略未放行 22 | 查目标机防火墙与管理台 ACL |
| agent 在沙箱/带代理环境里连公网地址必超时 | 透明代理劫持 SSH | 本来就不该连公网地址——改用机器名走 tailnet（100.x 不经公网代理） |
| 远程命令在 Windows 上行为怪异 | 默认 shell 是 cmd | 显式 `powershell -NoProfile -Command "..."`；中文与路径加引号 |
| 用了一阵子突然全断 | 设备密钥到期 | 管理台给该机 Disable key expiry 后重新授权入网 |

## 9 日常用法速查（agent 侧）

```bash
ssh <用户>@<机器名> '<一条命令>'                 # 远程执行并拿回输出
ssh <用户>@<机器名>                              # 进交互 shell（agent 少用，优先上一行）
scp <本地文件> <用户>@<机器名>:<目标路径>          # 推文件
scp <用户>@<机器名>:<远程文件> <本地路径>          # 拉文件
tailscale status                                 # 看全网设备在线状态
tailscale ping <机器名>                           # 只验通路不登录
```

## 10 免费档口径与替代

- Personal 档 $0（2026-11 查官网价目页）：最多 6 个用户、用户自有设备不限台数、含 50 个 tagged 资源、3 个 ACL 组、临时资源 1000 分钟/月，仅限非商业用途。数字改过多次（2026-04 刚把用户上限 3→6、取消 100 台设备上限），引用前查 tailscale.com/pricing 现行页，别背旧数。
- 不想依赖官方控制面：Headscale（开源自托管），客户端 `tailscale up --login-server <自有服务地址>` 接入，日常功能够用，模式二等高级面以其现行文档为准。
- 同类替代：ZeroTier、NetBird、Cloudflare Zero Trust，思路同为「先组私有网，再在网内操作」，选型时按免费档与服务端形态支持逐项核对，不凭印象。

## 11 安全收尾清单（交付前逐项打勾）

- [ ] 目标机公网 SSH 已按 5.4 收口，并从公网侧试连验证被拒
- [ ] 服务器类节点已 Disable key expiry
- [ ] ACL/策略不是默认全通到底：至少 SSH 这一项按「哪台能登哪台」收紧过
- [ ] agent 私钥只在 agent 本机受限目录，公钥分发有记录（放了哪几台）
- [ ] 验收四件套原始输出留证（哪天、哪台机、回显是什么）
- [ ] 机器名、登录用户、模式（一/二）、收口方式写进自己的 TOOLS/连接文档，下次直接照用
