---
title: 【Linux】实现禁用境外 IP 的脚本
tags:
  - Linux
  - 网络安全
  - IP 过滤
  - 脚本编程
categories:
  - Linux
abbrlink: b55fa601
date: 2025-02-26 08:00:00
---

## 问题背景

在 Linux 服务器上，出于安全考虑，管理员可能希望禁用来自境外 IP 的访问，以防止潜在的攻击和未授权访问。然而，某些情况下，SSH 使用者的 IP 需要被排除在外，以确保管理员能够远程访问服务器。本文将介绍如何编写一个脚本来实现这一功能。

## 1. 脚本功能概述

该脚本将执行以下操作：

- 获取当前 SSH 使用者的 IP 地址。
- 使用 `iptables` 规则禁用境外 IP 的访问。
- 确保 SSH 使用者的 IP 地址不被禁用。

## 2. 脚本实现

### 2.1 创建脚本文件

在您的 Linux 服务器上，创建一个名为 `disable_foreign_ip.sh` 的文件：

```bash
touch disable_foreign_ip.sh
chmod +x disable_foreign_ip.sh
```

### 2.2 编写脚本内容

将以下内容复制到 `disable_foreign_ip.sh` 文件中：

```bash
#!/bin/bash

# 获取当前 SSH 使用者的 IP 地址
SSH_IP=$(who | grep 'pts' | awk '{print $NF}' | xargs -I {} ip route get {} | awk 'NR==1 {print $NF}')

# 定义允许的国家 IP 范围（示例：仅允许中国 IP）
ALLOWED_COUNTRIES=("CN")

# 获取当前的所有 IP 地址
ALL_IPS=$(curl -s https://ipinfo.io/ip)

# 禁用境外 IP
for ip in $ALL_IPS; do
    # 检查 IP 是否在允许的国家范围内
    if ! [[ "${ALLOWED_COUNTRIES[@]}" =~ "${ip}" ]]; then
        # 检查是否为 SSH 使用者的 IP
        if [ "$ip" != "$SSH_IP" ]; then
            echo "禁用 IP: $ip"
            iptables -A INPUT -s $ip -j DROP
        fi
    fi
done

echo "境外 IP 禁用完成，SSH 使用者 IP 被排除在外。"
```

### 2.3 脚本说明

- **获取 SSH 使用者的 IP 地址**：使用 `who` 命令获取当前 SSH 使用者的 IP 地址。
- **定义允许的国家 IP 范围**：在示例中，您可以根据需要修改允许的国家 IP 范围。
- **禁用境外 IP**：使用 `iptables` 命令禁用不在允许范围内的 IP 地址，确保 SSH 使用者的 IP 不被禁用。

## 3. 运行脚本

在终端中运行脚本：

```bash
sudo ./disable_foreign_ip.sh
```

## 4. 注意事项

- **iptables 权限**：确保您有足够的权限来修改 `iptables` 规则。
- **IP 地址更新**：如果您的 SSH 使用者 IP 地址发生变化，您需要重新运行脚本以更新规则。
- **测试环境**：在生产环境中使用前，建议在测试环境中验证脚本的有效性。

## 5. 总结

通过编写一个简单的脚本，您可以在 Linux 服务器上实现禁用境外 IP 的功能，同时确保 SSH 使用者的 IP 不受影响。这种方法可以有效提高服务器的安全性，防止潜在的攻击。

## 参考资料

- [iptables 官方文档](https://www.netfilter.org/documentation/index.html)
- [Linux Shell 脚本编程](https://tldp.org/LDP/Bash-Beginners-Guide/html/)

---

希望这篇文章能帮助您更好地理解如何在 Linux 中实现禁用境外 IP 的脚本。如果您有任何问题，欢迎在评论区讨论！
--- 