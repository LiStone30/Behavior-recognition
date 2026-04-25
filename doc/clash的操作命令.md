太好了！既然方案二成功，现在就把完整的配置整理成文档，方便你以后管理和迁移。

---

# Clash Mihomo 核心服务配置文档（无图形界面）

## 一、服务概述

采用 `verge-mihomo` 核心直接作为 systemd 服务运行，无需依赖 `clash-verge-service` 或图形界面。实现开机自启、后台稳定运行、可通过 HTTP API 控制。

### 环境信息
- **操作系统**：Ubuntu 24.04 LTS
- **用户**：`ilstone`
- **核心路径**：`/usr/bin/verge-mihomo`
- **配置文件目录**：`/home/ilstone/.local/share/io.github.clash-verge-rev.clash-verge-rev/`
- **主配置文件**：`clash-verge.yaml`
- **监听端口**：
  - `mixed-port: 7897`（HTTP / SOCKS5 混合）
  - `socks-port: 7898`（纯 SOCKS5）
  - `port: 7899`（纯 HTTP）

---

## 二、服务安装步骤

### 1. 确认核心与配置文件存在
```bash
# 检查 mihomo 核心
ls -l /usr/bin/verge-mihomo

# 检查配置文件
ls -l ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml
```

### 2. 创建 systemd 服务单元文件
```bash
sudo tee /etc/systemd/system/mihomo.service <<EOF
[Unit]
Description=Clash Mihomo Core
After=network.target

[Service]
Type=simple
User=ilstone
WorkingDirectory=/home/ilstone/.local/share/io.github.clash-verge-rev.clash-verge-rev
ExecStart=/usr/bin/verge-mihomo -f /home/ilstone/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml -d /home/ilstone/.local/share/io.github.clash-verge-rev.clash-verge-rev
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 3. 停止并禁用旧的 clash-verge-service（如果存在）
```bash
sudo systemctl stop clash-verge-service
sudo systemctl disable clash-verge-service
```

### 4. 启动新服务并设置开机自启
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mihomo
```

### 5. 验证服务状态与端口监听
```bash
# 查看服务状态
sudo systemctl status mihomo

# 检查端口是否监听
sudo ss -tlnp | grep -E '7897|7898|7899'
```

预期输出示例：
```
LISTEN 0      4096         0.0.0.0:7897       0.0.0.0:*    users:(("verge-mihomo",pid=1234,fd=8))
```

---

## 三、日常管理命令

### 服务启停
```bash
sudo systemctl start mihomo      # 启动
sudo systemctl stop mihomo       # 停止
sudo systemctl restart mihomo    # 重启
sudo systemctl status mihomo     # 查看状态
```

### 查看日志
```bash
# 实时日志
sudo journalctl -u mihomo -f

# 最近100行
sudo journalctl -u mihomo -n 100 --no-pager
```

### 重新加载配置（修改 `clash-verge.yaml` 后）
```bash
# 方法一：通过API重载（推荐）
curl -X PUT http://127.0.0.1:9090/configs?force=true

# 方法二：重启服务（会短暂中断）
sudo systemctl restart mihomo
```
> 注意：API 端口默认为 9090，如果配置文件未改动则保持默认。

### 切换代理模式（Rule / Global / Direct）
```bash
# 查看当前模式
curl http://127.0.0.1:9090/configs | jq .mode

# 切换为规则模式
curl -X PUT -H "Content-Type: application/json" -d '{"mode":"Rule"}' http://127.0.0.1:9090/configs

# 切换为全局模式
curl -X PUT -H "Content-Type: application/json" -d '{"mode":"Global"}' http://127.0.0.1:9090/configs

# 切换为直连模式
curl -X PUT -H "Content-Type: application/json" -d '{"mode":"Direct"}' http://127.0.0.1:9090/configs
```

---

## 四、配置终端代理环境变量

为了在 SSH 会话中自动使用代理，将以下内容追加到 `~/.bashrc`：

```bash
cat >> ~/.bashrc <<'EOF'

# Clash 代理配置
export http_proxy="http://127.0.0.1:7897"
export https_proxy="http://127.0.0.1:7897"
export all_proxy="socks5://127.0.0.1:7898"
export no_proxy="localhost,127.0.0.1,::1"
EOF
```

然后执行：
```bash
source ~/.bashrc
```

**测试代理是否生效**：
```bash
curl -I https://www.google.com
```

若返回 `200 OK` 或 `301 Moved Permanently`，则说明成功。

---

## 五、故障排查

| 现象 | 可能原因 | 解决方法 |
|------|----------|----------|
| `systemctl status mihomo` 显示 `inactive` | 服务未启动 | `sudo systemctl start mihomo` |
| 端口未监听（`ss` 无输出） | 核心启动失败或配置文件错误 | 查看日志：`sudo journalctl -u mihomo -n 50` |
| 日志中出现 `country.mmdb` 错误 | 缺少 GeoIP 数据库 | 从订阅链接下载或从图形界面复制 `Country.mmdb` 到配置目录 |
| 日志中出现 `yaml: unmarshal errors` | 配置文件语法错误 | 使用 `yamllint clash-verge.yaml` 检查格式 |
| 代理不生效（`curl` 无响应） | 环境变量未设置或端口配错 | `echo $http_proxy` 确认值；`sudo ss -tlnp \| grep 7897` 确认监听 |
| 重启后服务未启动 | 未 enable | `sudo systemctl enable mihomo` |

---

## 六、卸载与清理

```bash
# 停止并禁用服务
sudo systemctl stop mihomo
sudo systemctl disable mihomo

# 删除服务文件
sudo rm /etc/systemd/system/mihomo.service
sudo systemctl daemon-reload

# （可选）删除配置文件目录
rm -rf ~/.local/share/io.github.clash-verge-rev.clash-verge-rev

# （可选）清除环境变量配置
# 编辑 ~/.bashrc 删除添加的代理行
```

---

## 七、进阶：开机延时启动（避免网络未就绪）

如果服务启动时网络尚未完全准备就绪，可添加延时：
```bash
sudo mkdir -p /etc/systemd/system/mihomo.service.d
sudo tee /etc/systemd/system/mihomo.service.d/delay.conf <<EOF
[Service]
ExecStartPre=/bin/sleep 5
EOF
sudo systemctl daemon-reload
sudo systemctl restart mihomo
```

---

## 八、文件清单

- **服务单元**：`/etc/systemd/system/mihomo.service`
- **核心二进制**：`/usr/bin/verge-mihomo`
- **配置文件**：`~/.local/share/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml`
- **工作目录**：`~/.local/share/io.github.clash-verge-rev.clash-verge-rev/`
- **日志**：`journalctl -u mihomo`
- **API 端点**：`http://127.0.0.1:9090`（可配合 `curl` 或 Web UI 使用）

---

**文档版本**：2.0（基于独立 mihomo 服务）  
**适用系统**：Ubuntu 24.04 LTS  
**最后更新**：2026-04-25

---

现在，你的远程开发服务器已经拥有了一个完全自动化、开机自启、可通过命令行控制的代理服务。配合 Tailscale 虚拟组网，你可以从任何地方的笔记本上无缝 SSH 进入 Ubuntu 并享受代理加速。如有其他问题，随时提问！