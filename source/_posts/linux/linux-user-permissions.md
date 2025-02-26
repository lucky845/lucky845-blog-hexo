---
title: 【Linux】用户和权限系统详解
tags:
  - Linux
  - 用户管理
  - 权限管理
  - 系统安全
categories:
  - Linux
abbrlink: b55fa596
date: 2025-02-26 03:00:00
---

## 问题背景

Linux 是一个多用户操作系统，用户和权限管理是确保系统安全和资源合理分配的重要组成部分。理解 Linux 的用户和权限系统，可以帮助管理员有效管理系统资源，保护敏感数据。本文将介绍 Linux 的用户管理、组管理以及权限管理的基本概念和操作方法。

## 1. 用户管理

### 1.1 用户概念

在 Linux 中，用户是指可以登录系统并执行操作的实体。每个用户都有一个唯一的用户名和用户 ID（UID）。系统中的每个用户都可以拥有自己的文件和目录。

### 1.2 查看当前用户

使用 `whoami` 命令查看当前登录的用户：

```bash
whoami
```

### 1.3 添加用户

使用 `useradd` 命令添加新用户：

```bash
sudo useradd -m newuser  # 创建新用户并创建主目录
```

### 1.4 设置用户密码

使用 `passwd` 命令设置用户密码：

```bash
sudo passwd newuser
```

### 1.5 删除用户

使用 `userdel` 命令删除用户：

```bash
sudo userdel -r newuser  # 删除用户及其主目录
```

## 2. 组管理

### 2.1 组概念

组是用户的集合，允许对多个用户进行统一管理。每个组都有一个组名和组 ID（GID）。用户可以属于一个或多个组。

### 2.2 查看当前组

使用 `groups` 命令查看当前用户所属的组：

```bash
groups
```

### 2.3 添加组

使用 `groupadd` 命令添加新组：

```bash
sudo groupadd newgroup
```

### 2.4 将用户添加到组

使用 `usermod` 命令将用户添加到组：

```bash
sudo usermod -aG newgroup username
```

### 2.5 删除组

使用 `groupdel` 命令删除组：

```bash
sudo groupdel newgroup
```

## 3. 权限管理

### 3.1 文件和目录权限

在 Linux 中，每个文件和目录都有三种基本权限：读取（r）、写入（w）和执行（x）。这些权限可以分配给三类用户：

- **文件所有者**（User）
- **同组用户**（Group）
- **其他用户**（Others）

### 3.2 查看文件权限

使用 `ls -l` 命令查看文件和目录的权限：

```bash
ls -l
```

输出示例：

```
-rw-r--r-- 1 user group  4096 Feb 26 00:00 file.txt
```

- 第一列表示权限：`-rw-r--r--`
  - 第一个字符表示文件类型（`-` 表示文件，`d` 表示目录）。
  - 接下来的三个字符表示所有者的权限（`rw-` 表示可读和可写）。
  - 中间三个字符表示同组用户的权限（`r--` 表示可读）。
  - 最后三个字符表示其他用户的权限（`r--` 表示可读）。

### 3.3 更改文件权限

使用 `chmod` 命令更改文件或目录的权限：

```bash
chmod 755 file.txt  # 设置权限为 rwxr-xr-x
```

### 3.4 更改文件所有者

使用 `chown` 命令更改文件或目录的所有者：

```bash
chown user:group file.txt  # 将文件的所有者和组更改为 user 和 group
```

### 3.5 设置特殊权限

- **SUID**：设置用户 ID 位，允许用户以文件所有者的身份执行文件。
- **SGID**：设置组 ID 位，允许用户以文件所属组的身份执行文件。
- **Sticky Bit**：仅允许文件所有者删除文件，通常用于 `/tmp` 目录。

设置 SUID 示例：

```bash
chmod u+s file.txt
```

## 4. 总结

Linux 的用户和权限管理系统是确保系统安全和资源合理分配的重要组成部分。通过掌握用户和组的管理、文件权限的设置，您可以有效地管理 Linux 系统，保护敏感数据。

## 参考资料

- [Linux 用户和组管理](https://linuxcommand.org/lc3_man_pages/useradd1.html)
- [Linux 权限管理](https://www.tldp.org/LDP/abs/html/)

---

希望这篇文章能帮助您更好地理解 Linux 的用户和权限系统。如果您有任何问题，欢迎在评论区讨论！
--- 