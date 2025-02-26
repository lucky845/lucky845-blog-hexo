---
title: 【Linux】如何扩容已使用的磁盘
tags:
  - Linux
  - 磁盘管理
  - 扩容
  - 系统管理
categories:
  - Linux
abbrlink: b55fa595
date: 2025-02-26 02:00:00
---

## 问题背景

在服务器运行过程中，磁盘空间可能会逐渐不足，导致系统性能下降或无法正常工作。扩容磁盘是解决这一问题的有效方法。本文将介绍如何在 Linux 服务器上扩容已使用的磁盘，将新挂载的磁盘扩展到之前的磁盘。

## 1. 准备工作

在开始扩容之前，请确保您有以下准备：

- 具有 root 权限或 sudo 权限的用户。
- 新的磁盘已经物理连接到服务器。
- 备份重要数据，以防在操作过程中出现意外。

## 2. 查看当前磁盘信息

使用 `lsblk` 命令查看当前磁盘和分区信息：

```bash
lsblk
```

您将看到类似以下的输出：

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   100G  0 disk 
└─sda1   8:1    0   100G  0 part /
sdb      8:16   0   50G   0 disk 
```

在这个例子中，`sda` 是主磁盘，`sdb` 是新挂载的磁盘。

## 3. 创建新分区

使用 `fdisk` 或 `parted` 创建新分区：

```bash
sudo fdisk /dev/sdb
```

在 `fdisk` 中，您可以使用以下命令：

- `n`：创建新分区。
- `p`：选择主分区。
- `1`：选择分区号。
- 按照提示设置分区大小。

完成后，使用 `w` 命令保存更改并退出。

## 4. 格式化新分区

使用 `mkfs` 命令格式化新分区：

```bash
sudo mkfs.ext4 /dev/sdb1
```

## 5. 挂载新分区

创建挂载点并挂载新分区：

```bash
sudo mkdir /mnt/newdisk
sudo mount /dev/sdb1 /mnt/newdisk
```

## 6. 扩展现有文件系统

### 6.1 使用 LVM（逻辑卷管理）

如果您使用 LVM 管理磁盘，可以将新磁盘添加到现有卷组中。

#### 6.1.1 创建物理卷

```bash
sudo pvcreate /dev/sdb1
```

#### 6.1.2 将物理卷添加到卷组

```bash
sudo vgextend myvg /dev/sdb1
```

#### 6.1.3 扩展逻辑卷

```bash
sudo lvextend -l +100%FREE /dev/myvg/mylv
```

#### 6.1.4 扩展文件系统

对于 ext4 文件系统，使用以下命令扩展文件系统：

```bash
sudo resize2fs /dev/myvg/mylv
```

### 6.2 不使用 LVM

如果不使用 LVM，您需要使用 `parted` 或 `gparted` 工具来调整分区大小。

#### 6.2.1 使用 `parted`

```bash
sudo parted /dev/sda
```

在 `parted` 中，使用以下命令：

- `resizepart`：调整分区大小。
- 输入分区号和新大小。

完成后，使用 `resize2fs` 扩展文件系统：

```bash
sudo resize2fs /dev/sda1
```

## 7. 更新 `/etc/fstab`

如果您希望在系统重启后自动挂载新分区，请编辑 `/etc/fstab` 文件：

```bash
sudo nano /etc/fstab
```

添加以下行：

```
/dev/sdb1 /mnt/newdisk ext4 defaults 0 2
```

## 8. 总结

通过以上步骤，您可以在 Linux 服务器上成功扩容已使用的磁盘。无论是使用 LVM 还是不使用 LVM，掌握这些操作都能帮助您更好地管理磁盘空间，确保系统的稳定运行。

## 参考资料

- [LVM 官方文档](https://www.sourceware.org/lvm2/)
- [Linux 文件系统管理](https://linuxcommand.org/lc3_man_pages/mount8.html)

---

希望这篇文章能帮助您更好地理解如何在 Linux 服务器上扩容已使用的磁盘。如果您有任何问题，欢迎在评论区讨论！
--- 