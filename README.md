```markdown
# 在 DigitalOcean 上部署 WireGuard VPN (wg-easy)

> 从零开始，手把手教你用 DigitalOcean VPS + Docker + wg-easy 搭建带 Web 管理面板的 WireGuard，实现个人设备安全互联与科学上网。

---

## 📋 前置条件
- 已激活的 DigitalOcean 账户（需绑定 PayPal 或信用卡；新用户有 **$200/60 天** 免费试用金）。
- 本地 SSH 终端（Mac/Linux 自带；Windows 推荐 PowerShell 或 WSL）。

---

## 🚀 第一步：创建 DigitalOcean Droplet（云服务器）

1. 登录 [DigitalOcean 控制台](https://cloud.digitalocean.com/)。
2. 点击右上角 **Create** → **Droplets**。
3. **选择机房**：推荐 `San Francisco` → **SFO3**（避开 SFO1，容易不可用）。
4. **系统镜像**：`Ubuntu 24.04 LTS`。
5. **选择方案**：
   - 推荐 `Basic` → **$6/月**（1 GB RAM / 25 GB SSD），稳定无忧。
   - 如果预算极紧，也可选 **$4/月**（512 MB / 10 GB），但 ⚠️ **必须按后续步骤创建 Swap 并限制容器内存，否则极易因内存不足导致服务崩溃**。
6. **认证方式**：
   - 切换到 **Password** 选项卡（不用 SSH Key）。
   - 设置 `root` 密码，**首尾必须是英文字母**（如 `MyVPN!234a`），否则创建时会报错。
7. **其他选项**：
   - 取消勾选 `Enable automated backups`（额外收费）。
   - 取消勾选 `Enable IPv6`（暂不需要）。
8. 点击 **Create Droplet**，等待状态变为 `Running`。

---

## 🖥️ 第二步：连接服务器并初始化

先在 DigitalOcean 控制台找到 Droplet 的 **Public IPv4**。

### 2.1 SSH 登录
```bash
ssh root@你的服务器IP
```
首次连接输入 `yes`，然后粘贴 root 密码（输入时不显示字符，正常）。

### 2.2 针对 512 MB 内存方案：创建 Swap（1 GB 方案可跳过，但也可以执行）
**如果不做这一步，512 MB 内存很可能在运行 Docker 后触发 OOM，导致容器被杀。**
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```
执行完后即拥有额外的 1 GB 虚拟内存。

---

## 🐳 第三步：安装 Docker

```bash
curl -fsSL https://get.docker.com | bash
```
Docker 安装完成后会自动运行。

---

## ⚙️ 第四步：启动 wg-easy 容器

**请将命令中的 `WG_HOST` 和 `PASSWORD` 替换为你自己的值：**

```bash
sudo docker run -d \
  --name=wg-easy \
  --memory=256m \
  --memory-swap=512m \
  -e WG_HOST=143.198.78.242 \
  -e PASSWORD=你的管理员密码 \
  -v ~/.wg-easy:/etc/wireguard \
  -p 51820:51820/udp \
  -p 51821:51821/tcp \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  --restart unless-stopped \
  weejewel/wg-easy
```

**参数说明 & 易错点：**
- `WG_HOST`：**必须**填服务器的公网 IPv4，不能填 `127.0.0.1`。
- `PASSWORD`：Web 管理面板的登录密码，**首尾必须是英文字母**（如 `MyWG!2024a`）。
- `--memory=256m --memory-swap=512m`：限制容器物理内存 256 MB，允许交换空间 512 MB，防止 512 MB 主机因内存不足杀死容器。
- 端口映射：`51820/udp` 为 WireGuard 通信端口，`51821/tcp` 为 Web 管理面板端口。

检查容器运行状态：
```bash
sudo docker ps
```
应能看到 `wg-easy` 容器，状态为 `Up`。

---

## 🔥 第五步：配置 DigitalOcean 云防火墙

云防火墙默认拦截所有入站流量，必须手动放行端口。

1. 控制台左侧菜单 → **Networking** → **Firewalls** → **Create Firewall**。
2. **名称**：随意，如 `wg-easy-rules`。
3. **Apply to Droplet**：搜索并**务必勾选**你的 Droplet（否则规则不会生效）。
4. **Inbound Rules**（入站规则）：
   - 保留默认的 `SSH`（TCP 22）。
   - 点击 **Add another** 添加：
     - Type: `Custom`，Protocol: `UDP`，Port: `51820`，Destinations: `All IPv4, All IPv6`
   - 再次点击 **Add another** 添加：
     - Type: `Custom`，Protocol: `TCP`，Port: `51821`，Destinations: `All IPv4, All IPv6`
5. **Outbound Rules**：保持默认（允许所有出站）。
6. 点击 **Create Firewall**。

---

## 🌐 第六步：登录 Web 管理后台

浏览器访问 `http://你的服务器IP:51821`，输入你设置的管理员密码登录。

- 点击 **+ New Client** 创建客户端（如 `Phone`, `Laptop`），创建后可**下载配置文件**或**扫描二维码**。
- ⭐ **重要优化（避免内网断流）**：  
  在客户端列表中点击刚创建的客户端名称，将 `AllowedIPs` 从 `0.0.0.0/0, ::/0` 修改为：
  ```
  0.0.0.0/1, 128.0.0.0/1
  ```
  然后保存并**重新下载配置文件**。  
  这样 VPN 只接管外网流量，本地局域网（如 `10.x.x.x` 或 `192.168.x.x`）不受影响。

---

## 📱 第七步：客户端连接与验证

### 桌面端（macOS / Windows / Linux）
安装官方 [WireGuard 客户端](https://www.wireguard.com/install/)，导入从后台下载的 `.conf` 文件即可。
- **Ubuntu 命令行导入**：
  ```bash
  sudo nmcli connection import type wireguard file /path/to/your.conf
  sudo nmcli connection up 配置名
  ```

### 手机端（iPhone / Android）
安装 WireGuard App，直接扫描后台生成的二维码。

### 验证是否成功
```bash
curl ifconfig.me
```
若显示的 IP 变为你服务器的公网 IP，则 VPN 已生效。

---

## 🔧 常见问题排查

### ① Web 面板打不开（`http://IP:51821`）
- 检查容器是否运行：`sudo docker ps -a`，若已停止则 `sudo docker start wg-easy`。
- 确认云防火墙已放行 TCP 51821，且已应用到正确的 Droplet。
- 在服务器内部测试：`curl -v telnet://127.0.0.1:51821`，内部通外部不通 → 问题在防火墙。

### ② 连上 VPN 但无法访问互联网
- 检查客户端的 `AllowedIPs` 是否包含 `0.0.0.0/1, 128.0.0.0/1` 或 `0.0.0.0/0`。
- 检查服务器 IP 转发是否开启：
  ```bash
  sysctl net.ipv4.ip_forward
  ```
  应输出 `1`。若不是，则执行：
  ```bash
  echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
  sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
  ```
- 确保云防火墙放行了 UDP 51820。

### ③ 连接 VPN 后本地内网无法访问
- 请确认已将 `AllowedIPs` 改为 `0.0.0.0/1, 128.0.0.0/1`，而不是 `0.0.0.0/0`。
- 若仍不行，可尝试在客户端配置中手动追加内网路由（例如 `10.0.0.0/8, 192.168.0.0/16`）。

### ④ 内存不足导致容器频繁重启/停止
- 查看日志：`sudo docker logs wg-easy`，若出现 `OOM killed` 则是内存不足。
- 512 MB 方案必须**同时创建 Swap 并加上 `--memory=256m` 限制**。
- 如无法解决，建议销毁当前 Droplet，重建一台 **1 GB 内存**的方案。

### ⑤ DigitalOcean 控制台找不到 Droplet
- 核对登录邮箱是否正确，检查左上角是否选中了正确的 **Team/Project**。
- 新账户需绑定支付方式才能完整显示资源。
- 联系 [DigitalOcean 官方支持](https://cloud.digitalocean.com/support/create)。

---

## 📄 许可证
本指南涉及的软件（WireGuard、wg-easy、Docker）均遵循各自的开源协议。  
请遵守当地法律法规，本教程仅供学习交流使用。
```

现在这整段就是一个完整的 Markdown 文件，没有断裂，可直接粘贴到仓库 `README.md`。如果有细节需要调整，随时告诉我。
