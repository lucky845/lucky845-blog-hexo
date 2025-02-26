---
title: 【Linux】磁盘清理指南
tags:
  - Linux
  - 磁盘清理
  - 系统管理
  - 性能优化
categories:
  - Linux
abbrlink: b55fa600
date: 2025-02-26 07:00:00
---

## 问题背景

随着时间的推移，Linux 系统中的磁盘空间可能会被临时文件、日志文件和不再使用的程序占用。定期清理磁盘可以帮助释放空间，提高系统性能。本文将介绍一些常用的磁盘清理方法和工具。

## 1. 查看磁盘使用情况

在开始清理之前，首先需要查看磁盘的使用情况。可以使用 `df` 命令查看各个分区的使用情况：

```bash
df -h
```

输出示例：

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   80G   15G  85% /
```

## 2. 清理临时文件

### 2.1 使用 `tmpwatch`

`tmpwatch` 是一个用于清理临时文件的工具，可以删除在指定时间内未被访问的文件。

```bash
sudo apt install tmpreaper  # Debian/Ubuntu
sudo yum install tmpwatch    # RHEL/CentOS
```

使用示例：

```bash
sudo tmpwatch --mtime 24 /tmp
```

这将删除 `/tmp` 目录中 24 小时内未被访问的文件。

### 2.2 手动清理

您也可以手动清理 `/tmp` 目录：

```bash
sudo rm -rf /tmp/*
```

## 3. 清理日志文件

### 3.1 使用 `logrotate`

`logrotate` 是一个用于管理日志文件的工具，可以自动轮换、压缩和删除旧的日志文件。确保您的系统已安装并配置 `logrotate`。

### 3.2 手动清理

您可以手动查看和清理 `/var/log` 目录中的日志文件：

```bash
sudo du -sh /var/log/*
```

使用 `rm` 命令删除不再需要的日志文件：

```bash
sudo rm /var/log/old-log-file.log
```

## 4. 清理包管理器缓存

### 4.1 Debian/Ubuntu

使用以下命令清理 APT 缓存：

```bash
sudo apt clean
```

### 4.2 RHEL/CentOS

使用以下命令清理 YUM 缓存：

```bash
sudo yum clean all
```

## 5. 查找和删除大文件

### 5.1 使用 `find`

使用 `find` 命令查找大于 100MB 的文件：

```bash
find / -type f -size +100M
```

### 5.2 删除不需要的文件

在确认不再需要的文件后，可以使用 `rm` 命令删除它们：

```bash
sudo rm /path/to/large-file
```

## 6. 使用 `ncdu` 进行磁盘使用分析

`ncdu` 是一个基于文本的磁盘使用分析工具，可以帮助您快速找到占用磁盘空间的文件和目录。

```bash
sudo apt install ncdu  # Debian/Ubuntu
sudo yum install ncdu  # RHEL/CentOS
```

使用示例：

```bash
ncdu /
```

## 7. 总结

定期清理 Linux 磁盘可以帮助释放空间，提高系统性能。通过使用上述工具和方法，您可以有效地管理磁盘空间，保持系统的整洁和高效运行。

## 参考资料

- [Linux Disk Cleanup Guide](https://www.tecmint.com/clean-linux-disk-space/)
- [Logrotate Documentation](https://linux.die.net/man/8/logrotate)

---

希望这篇文章能帮助您更好地理解如何清理 Linux 磁盘。如果您有任何问题，欢迎在评论区讨论！
--- 