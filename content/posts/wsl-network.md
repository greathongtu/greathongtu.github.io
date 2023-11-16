+++
title = 'wsl 代理问题'
date = 2023-11-17T00:41:53+08:00
draft = false
tags = ['wsl']
+++

## 问题
win11 使用 vs code 连接 wsl 的时候，一些 vs code 的拓展无法连接网络，报错：
`Failed to establish a socket connection to proxies: ["PROXY 127.0.0.1:7890"]`

或者 go build 因为网络问题无法继续。

## 原因
Windows 开启 Clash 之后，Windows 系统代理一般被设置为 127.0.0.1:7890，VS Code 会继承这个代理设置，wsl 中的 vscode-server 也继承了这个代理设置。但是 wsl 在 127.0.0.1:7890 上并没有代理服务，导致 vscode-server 联网失败。所以 wsl 中正确设置 “http_proxy” 和 “https_proxy” 环境变量可以解决问题。

## 解决办法
1. 在 wsl 中正确设置 path
```bash []
# ~/.bashrc

# 获取 Host IP
WINDOWS_IP=$(ip route | grep default | awk '{print $3}')
PROXY_HTTP="http://${WINDOWS_IP}:7890"
# 设置环境变量
export http_proxy="${PROXY_HTTP}"
export https_proxy="${PROXY_HTTP}"
```
```bash
# ~/.config/fish/config.fish

set -l WINDOWS_IP (ip route | grep default | awk '{print $3}')
set -l PROXY_HTTP "http://$WINDOWS_IP:7890"
set -gx http_proxy $PROXY_HTTP
set -gx https_proxy $PROXY_HTTP
```

2. Clash 打开 allow lan （表示允许局域网（Local Area Network, LAN）访问。当启用 “allow lan” 选项时，局域网内的其他设备可以连接到运行 Clash 的设备，并通过其代理服务访问互联网。这样，你可以将运行 Clash 的设备作为一个代理服务器，供局域网内的其他设备使用。）

3. 为防止 Windows 防火墙阻止 Ubuntu 子系统访问 Host IP，使用管理员权限打开powershell，在 Windows 防火墙中增加一条规则：
```pwsh
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound -InterfaceAlias "vEthernet (WSL)" -Action Allow
```
4. 可能需要重启