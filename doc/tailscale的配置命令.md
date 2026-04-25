好的，以下是为您整理的 **Tailscale 虚拟组网配置文档**，涵盖 Windows 笔记本和 Ubuntu 服务器之间的安装、配置、开机自启以及 SSH 连接步骤，适用于跨局域网远程开发场景。

---

# Tailscale 跨局域网虚拟组网配置文档

## 一、概述

Tailscale 基于 WireGuard 协议，能在两台或更多设备之间建立安全的点对点虚拟局域网，无需公网 IP，自动 NAT 穿透。配置后，Windows 笔记本可以通过虚拟 IP 或设备名直接 SSH 访问 Ubuntu 台式机，如同在同一局域网内。

## 二、安装与注册

### 1. Ubuntu 24.04 服务端安装
```bash
# 一键安装脚本
curl -fsSL https://tailscale.com/install.sh | sh

# 启动并登录（会生成一个链接，需在浏览器中完成认证）
sudo tailscale up
```
执行 `sudo tailscale up` 后会输出类似：
```
To authenticate, visit:
    https://login.tailscale.com/a/xxxxx
```
在任意浏览器打开该链接，使用 **Microsoft/GitHub/Google/Apple 账号** 登录。成功后终端显示 `Successfully added device.`

### 2. Windows 10 笔记本客户端安装
- 访问 [Tailscale 下载页](https://tailscale.com/download/windows)，下载 `.exe` 安装包。
- 双击安装，完成后会在系统托盘出现 Tailscale 图标。
- 右键点击图标 → **Log in**，使用 **与 Ubuntu 服务端完全相同的账号** 登录。
- 登录后，管理后台即会显示两台设备。

## 三、关键配置（确保长期稳定）

### 1. 禁用密钥过期（防止每隔数月需重新登录）
登录 [Tailscale 管理后台](https://login.tailscale.com/admin/machines)，找到 Ubuntu 设备：
- 点击设备右侧的 `...` 菜单
- 选择 **Disable Key Expiry** 🟢

> 同样可以对 Windows 设备禁用，但 Windows 通常有人值守，非必需。

### 2. 设置开机自启（Ubuntu 服务端）
```bash
sudo systemctl enable tailscaled
```

> 此步骤必须在 Ubuntu 上执行。启用后，即使服务器重启且无人登录，Tailscale 也会自动重新连接，虚拟 IP 保持不变。

### 3. 启用 MagicDNS（可选但强烈推荐）
在管理后台 **DNS** 页面，开启 **MagicDNS** 开关（新账号默认已开启）。开启后，可以使用设备名代替虚拟 IP 进行连接：
- `ssh ilstone@ilstone-ms-7d36`（设备名）
- 或 `ssh ilstone@100.126.200.35`（虚拟 IP）

## 四、连接验证与日常使用

### 1. 在 Windows 笔记本上检查设备状态
打开 PowerShell 或 CMD，运行：
```bash
tailscale status
```
输出示例：
```
100.126.200.35   ilstone-ms-7d36   ilstone@   linux   active
100.64.0.1       windows-pc        listone@   windows active
```
如果状态为 `active`，则说明虚拟网络已正常连接。

### 2. 通过 SSH 连接到 Ubuntu 服务器
```bash
ssh ilstone@ilstone-ms-7d36
```
或使用 IP：
```bash
ssh ilstone@100.126.200.35
```
首次连接会提示验证主机指纹，输入 `yes` 并输入 Ubuntu 用户密码即可登录。

### 3. 配置 SSH 密钥免密登录（提升安全与便利）
在 Windows 笔记本上生成密钥对（如已存在可跳过）：
```powershell
ssh-keygen -t ed25519
# 一路回车，默认保存在 C:\Users\你的用户名\.ssh\id_ed25519
```
将公钥复制到 Ubuntu 服务器：
```bash
# 方法一：使用 ssh-copy-id（Windows 10 新版 OpenSSH 支持）
ssh-copy-id ilstone@ilstone-ms-7d36

# 方法二：手动复制
type C:\Users\你的用户名\.ssh\id_ed25519.pub | ssh ilstone@ilstone-ms-7d36 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
后续 SSH 将不再要求输入密码。

## 五、故障排查

| 现象 | 可能原因 | 解决方法 |
|------|----------|----------|
| `tailscale status` 显示设备 `offline` | 网络中断或服务未运行 | 在 Ubuntu 上 `sudo systemctl restart tailscaled`；检查两台设备是否都能访问互联网 |
| SSH 连接拒绝 (`Connection refused`) | Ubuntu 未安装或未启动 SSH 服务 | `sudo apt install openssh-server -y`<br>`sudo systemctl enable --now ssh` |
| SSH 密码验证失败 | 用户名错误或密码错误 | 确认 Ubuntu 用户名为 `ilstone`；重置密码 `sudo passwd ilstone` |
| MagicDNS 无法解析设备名 | MagicDNS 未启用或 DNS 缓存问题 | 在后台确认 MagicDNS 开启；尝试 `ping ilstone-ms-7d36.tailxxxxx.net`；或改用虚拟 IP 连接 |
| 重启后虚拟 IP 改变 | 未禁用密钥过期或未设置开机自启 | 按第三章节配置；Tailscale 正常情况下 IP 不会变化（除非节点过期删除） |

## 六、常用管理命令

### Ubuntu 端
```bash
sudo tailscale up                 # 启动并登录
sudo tailscale down               # 断开 Tailscale
tailscale status                  # 查看网络状态
tailscale ip -4                   # 显示本机虚拟 IPv4
sudo systemctl restart tailscaled # 重启 Tailscale 服务
sudo tailscale logout             # 登出并删除当前设备
```

### Windows 端（PowerShell / CMD）
```bash
tailscale status
tailscale ip
tailscale logout
```

## 七、卸载
### Ubuntu
```bash
sudo apt remove tailscale
sudo rm -rf /var/lib/tailscale
```
### Windows
通过“设置 → 应用”卸载即可。

---

**文档版本**：1.0  
**适用系统**：Windows 10 / 11, Ubuntu 24.04 LTS  
**最后更新**：2026-04-25

---