# 1. 文件说明

- `config.yaml`：配置文件
- `ui`：UI 文件

---

# 2.使用方式

> 这里以 Arch Linux 为例。

## 2.1 安装二进制文件

首先安装 `mihomo`（Clash Meta）和 Web 界面：

```bash
sudo pacman -S mihomo
```

> mihomo 只有在 archlinuxcn 源、chaotic-aur 源、aur 源中才有。没有网络代理的化，建议先添加 archlinuxcn 源后，再进行下载安装。
> WebUI 只是静态文件，你可以用本项目的 `ui` 或者从 [GitHub MetaCubeX/metacubexd](https://github.com/MetaCubeX/metacubexd?tab=readme-ov-file) 下载。还可以通过 pacman 安装：`sudo pacman -S metacubexd-bin`（archlinuxcn 源和 aur 源才有 MetaCubexXD）。

配置文件位置是 `/etc/mihomo/config.yaml` 或 `~/.config/mihomo/config.yaml`。一般是将 `config.yaml` 复制到 `~/.config/mihomo/config.yaml` 下，而 `ui` 目录是放在和 `config.yaml` 同级目录下。

## 2.2 配置 Systemd 服务权限

### 2.2.1 方式一（推荐）

如果通过 `pacman` 安装，通常自带了 service 文件，你可以使用 systemd 自带的交互式命令，它会自动为你创建目录和文件：

```bash
sudo systemctl edit mihomo
```

在打开的编辑器中，输入以下内容（这会自动合并到主配置中）：

```ini
[Service]
# 如果你想以 root 运行以避开所有权限烦恼，取消下面两行的注释
# User=root
# Group=root

# 为 TUN 模式和 53 端口添加权限（针对非 root 用户）
AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_ADMIN

# 如果你想指定特定的配置文件路径
# ExecStart=
# ExecStart=/usr/bin/mihomo -d /etc/mihomo
```

> **注意**：如果你要覆盖 `ExecStart` 这种带有命令的参数，必须先写一行空的 `ExecStart=` 来清除旧指令，然后再写一行新的。

保存退出编辑器并重启服务：

```bash
sudo systemctl restart mihomo

```

你可以运行以下命令来查看 systemd 最终合并后的配置（有效配置）：

```bash
systemctl cat mihomo
```

你会看到文件底部出现了一个 `Drop-In: /etc/systemd/system/mihomo.service.d/override.conf` 区域。

### 2.2.2 方式二

编辑（或创建）服务文件 `/etc/systemd/system/mihomo.service`：

```ini
[Unit]
Description=Mihomo Daemon
After=network.target

[Service]
Type=simple
# 指定运行用户，如果是 root 则不需要下面的 Capabilities
User=your-user-name  # 填你的用户名
Group=your-group-name  # 填你的用户组
ExecStart=/usr/bin/mihomo -d /etc/mihomo
Restart=always

# 对应 NixOS 中的能力设置
AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_ADMIN

[Install]
WantedBy=multi-user.target
```

## 2.3 开启内核转发

创建文件 `/etc/sysctl.d/99-ip-forward.conf`：

```conf
net.ipv4.ip_forward = 1
# 如果需要 IPv6
net.ipv6.conf.all.forwarding = 1
```

然后执行 `sudo sysctl --system` 生效。

## 2.4 防火墙与路由设置 (nftables/iptables)

创建文件 `/etc/sysctl.d/99-rp-filter.conf`：

```conf
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
# 针对 config.yaml 中设置的 tun 接口（Meta）
net.ipv4.conf.Meta.rp_filter = 2
```

## 2.5 信任接口 (Firewalld/UFW)

如果你使用了防火墙（如 `firewalld`），需要手动放行：

- **Firewalld**:

```bash
sudo firewall-cmd --permanent --zone=trusted --add-interface=Meta
sudo firewall-cmd --reload
```

- **UFW**:

```bash
sudo ufw allow in on Meta
```
