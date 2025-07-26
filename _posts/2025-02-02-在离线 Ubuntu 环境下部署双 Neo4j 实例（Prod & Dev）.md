---
layout: post
title: '在离线 Ubuntu 环境下部署双 Neo4j 实例（Prod & Dev）'
subtitle: "在离线 Ubuntu 环境下部署双 Neo4j 实例（Prod & Dev）"
date: 2025-02-02
author: Anarkh-Lee
cover: './assets/img//cover_img/Neo4j.png'
tags: 图数据库



---

在许多开发和生产场景中，我们可能需要在同一台服务器上运行多个独立的 Neo4j 数据库实例，例如一个用于生产环境 (Prod)，一个用于开发测试环境 (Dev)。本文将详细介绍如何在 离线 的 Ubuntu 服务器上，使用 tar.gz 包部署两个 Neo4j 4.4 实例，并配置不同的端口以供外部访问，同时涵盖了部署过程中可能遇到的常见问题及其解决方案。

# 1. 需求

ubuntu系统下，离线环境，我想要部署两套neo4j的数据库（一套prod（服务端口17687，浏览器端口17474），一套dev（服务端口17688，浏览器端口17475），都需要别的主机能访问到）(下载的neo4j版本为neo4j-community-4.4.42-unix.tar.gz)

**前提条件：**Java 11 运行环境

# 2. 场景目标

- 环境: 离线 Ubuntu 服务器。
- 软件: Neo4j Community Edition 4.4.42 (以 neo4j-community-4.4.42-unix.tar.gz 形式提供)。
- 目标:
  - 部署一个生产实例 (Prod)，服务端口 17687 ，浏览器端口 17474 。
  - 部署一个开发实例 (Dev)，服务端口 17688 ，浏览器端口 17475 。
  - 两个实例均需配置为可从其他主机访问。
  - 解决部署过程中可能遇到的常见问题。

# 3. 准备工作

在开始部署之前，请确保满足以下前提条件：

1. 操作系统: Ubuntu Server (本文以 Ubuntu 为例，其他 Linux 发行版类似)。
2. Neo4j 安装包: 已将 neo4j-community-4.4.42-unix.tar.gz 文件传输到目标服务器。
3. Java 运行环境 (JRE/JDK): 至关重要的一点 ，Neo4j 4.4.x 版本明确要求 Java 11 。由于是离线环境，你需要预先将 Java 11 的安装包（例如 OpenJDK 11 JRE/JDK 的 .deb 包或压缩包）传输到服务器并完成安装。可以通过 java -version 命令检查当前 Java 版本。版本不匹配是导致启动失败或功能异常的常见原因。

# 4. 部署步骤

## 4.1 必须的配置

### **4.1.1 创建安装目录: **为 prod 和 dev 环境分别创建独立的目录。建议放在 /opt 目录下。

```bash
sudo mkdir -p /opt/neo4j-prod
```

```bash
sudo mkdir -p /opt/neo4j-dev
```

### 4.1.2 解压 Neo4j: 将 neo4j-community-4.4.42-unix.tar.gz 分别解压到对应的目录。假设你的压缩包位于当前用户的 home 目录下 ( ~ )。

```bash
tar -xzf neo4j-community-4.4.42-unix.tar.gz -C /opt/neo4j-prod --strip-components=1
```

```bash
tar -xzf neo4j-community-4.4.42-unix.tar.gz -C /opt/neo4j-dev --strip-components=1
```

- --strip-components=1 用于去除压缩包内顶层的目录结构。

### **4.1.3 创建数据和日志目录:** 为两个实例分别创建数据和日志存储目录，并确保 Neo4j 进程有权限访问（如果使用特定用户运行 Neo4j，需要调整权限）。

```bash
sudo mkdir -p /var/lib/neo4j-prod/data
```

```bash
sudo mkdir -p /var/log/neo4j-prod
```

```bash
sudo mkdir -p /var/lib/neo4j-dev/data
```

```bash
sudo mkdir -p /var/log/neo4j-dev
```

### **4.1.4 配置 Prod 实例: **编辑 prod 实例的配置文件 /opt/neo4j-prod/conf/neo4j.conf 。你可以使用 nano 或 vim 编辑器。

```bash
sudo nano /opt/neo4j-prod/conf/neo4j.conf

```

找到并修改或取消注释以下行：

```plain
# ... 其他配置 ...

# Bolt connector
# Bolt 监听地址和端口，0.0.0.0 表示监听所有网络接口
dbms.connector.bolt.listen_address=0.0.0.0:17687

# HTTP connector
# HTTP 浏览器监听地址和端口
dbms.connector.http.listen_address=0.0.0.0:17474

# HTTPS connector
# HTTPS 浏览器监听地址和端口 (如果启用HTTPS)
# dbms.connector.https.listen_address=0.0.0.0:17473 # 可以保持注释或配置一个不同于dev的端口

# 数据目录
dbms.directories.data=/var/lib/neo4j-prod/data

# 日志目录
dbms.directories.logs=/var/log/neo4j-prod

# 导入目录 (如果需要)
# dbms.directories.import=/path/to/prod/import

# 内存配置 (根据服务器资源调整)
# dbms.memory.heap.initial_size=1g
# dbms.memory.heap.max_size=1g
# dbms.memory.pagecache.size=2g

# 允许远程访问浏览器 (4.x 版本通常默认允许，但明确设置更保险)
# dbms.security.allow_csv_import_from_urls=true # 如果需要从URL导入CSV
# dbms.security.procedures.unrestricted=my.extensions.example,my.procedures.* # 如果有自定义插件

# ... 其他配置 ...

```

保存并关闭文件 (在 nano 中是 Ctrl+X , 然后 Y , 然后 Enter )。

### **4.1.5 配置 Dev 实例:** 编辑 dev 实例的配置文件 /opt/neo4j-dev/conf/neo4j.conf 。

```bash
sudo nano /opt/neo4j-dev/conf/neo4j.conf

```

找到并修改或取消注释以下行：

```plain
# ... 其他配置 ...

# Bolt connector
dbms.connector.bolt.listen_address=0.0.0.0:17688

# HTTP connector
dbms.connector.http.listen_address=0.0.0.0:17475

# HTTPS connector
# dbms.connector.https.listen_address=0.0.0.0:17476 # 可以保持注释或配置一个不同于prod的端口

# 数据目录
dbms.directories.data=/var/lib/neo4j-dev/data

# 日志目录
dbms.directories.logs=/var/log/neo4j-dev

# 导入目录 (如果需要)
# dbms.directories.import=/path/to/dev/import

# 内存配置 (通常 dev 环境可以配置得比 prod 低)
# dbms.memory.heap.initial_size=512m
# dbms.memory.heap.max_size=512m
# dbms.memory.pagecache.size=1g

# ... 其他配置 ...

```

保存并关闭文件 (在 nano 中是 Ctrl+X , 然后 Y , 然后 Enter )。

### **4.1.6 配置防火墙:** 如果你的 Ubuntu 服务器启用了防火墙 (如 ufw )，需要允许外部访问配置的端口。

```bash
# 允许 Prod 实例端口
sudo ufw allow 17687/tcp

```

```bash
sudo ufw allow 17474/tcp

```

```bash
# 如果启用了 HTTPS for Prod，也需要允许对应端口
# sudo ufw allow 17473/tcp

```

```bash
# 允许 Dev 实例端口
sudo ufw allow 17688/tcp

```

```bash
sudo ufw allow 17475/tcp

```

```bash
# 如果启用了 HTTPS for Dev，也需要允许对应端口
# sudo ufw allow 17476/tcp

```

```bash
# 重新加载防火墙规则使之生效
sudo ufw reload

```

```bash
# 查看防火墙状态和规则
sudo ufw status verbose

```

### 4.1.7 启动 Neo4j 实例: 分别启动两个实例。

启动 Prod 实例：

```bash
sudo /opt/neo4j-prod/bin/neo4j start

```

启动 Dev 实例：

```bash
sudo /opt/neo4j-dev/bin/neo4j start

```

### **4.1.8 检查状态: **检查两个实例的运行状态。

检查 Prod 实例：

```bash
sudo /opt/neo4j-prod/bin/neo4j status

```

检查 Dev 实例：

```bash
sudo /opt/neo4j-dev/bin/neo4j status

```

如果看到 "Neo4j is running" 或类似信息，表示启动成功。

### **4.1.9 访问和初始设置:**

至此，就算是部署完成了。

- **Prod 实例: **在其他机器的浏览器中访问 http://<你的服务器IP>:17474 。
- **Dev 实例: **在其他机器的浏览器中访问 http://<你的服务器IP>:17475 。

首次连接时，会要求使用默认用户名 neo4j 和默认密码 neo4j 登录。登录后，系统会强制你修改密码。请为每个实例设置不同的强密码。

![]({{ '/assets/img/neo4j_img/05.png' | prepend: '' }})
![](.\img\neo4j_img\05.png)

### 4.1.10 停止实例 (如果需要):

停止 Prod 实例：

```bash
sudo /opt/neo4j-prod/bin/neo4j stop

```

停止 Dev 实例：

```bash
sudo /opt/neo4j-dev/bin/neo4j stop

```

## 4.2 可选的配置

**可选：配置为 Systemd 服务 (推荐)**

为了方便管理（开机自启、后台运行、重启等），建议将 Neo4j 实例配置为 systemd 服务。

### 4.2.1 创建 Prod 服务文件:

```bash
sudo nano /etc/systemd/system/neo4j-prod.service

```

粘贴以下内容 (如果使用特定用户运行，请修改 User 和 Group ):

```plain
[Unit]
Description=Neo4j Graph Database (Prod)
After=network.target

[Service]
Type=forking
User=root  # 或者你创建的专用用户，如 neo4j
Group=root # 或者你创建的专用用户组，如 neo4j
ExecStart=/opt/neo4j-prod/bin/neo4j start
ExecStop=/opt/neo4j-prod/bin/neo4j stop
ExecStatus=/opt/neo4j-prod/bin/neo4j status
PIDFile=/opt/neo4j-prod/run/neo4j.pid # 确认 PID 文件路径是否正确，可能在 /var/run/neo4j-prod 等
Restart=on-failure
LimitNOFILE=65536 # 增加文件描述符限制

[Install]
WantedBy=multi-user.target

```

注意: PIDFile 的路径可能需要根据实际情况调整，检查 Neo4j 启动后 run 目录下的 neo4j.pid 文件位置。如果以非 root 用户运行，确保该用户对 /opt/neo4j-prod 及其子目录有读写执行权限，特别是 run , logs , data 目录。

### 4.2.2 创建 Dev 服务文件:

```bash
sudo nano /etc/systemd/system/neo4j-dev.service

```

粘贴以下内容 (同样注意 User 和 Group ):

```plain
[Unit]
Description=Neo4j Graph Database (Dev)
After=network.target

[Service]
Type=forking
User=root  # 或者你创建的专用用户
Group=root # 或者你创建的专用用户组
ExecStart=/opt/neo4j-dev/bin/neo4j start
ExecStop=/opt/neo4j-dev/bin/neo4j stop
ExecStatus=/opt/neo4j-dev/bin/neo4j status
PIDFile=/opt/neo4j-dev/run/neo4j.pid # 确认 PID 文件路径
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

### 4.2.3 启用和管理服务:

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

```

```bash
# 启动 Prod 服务
sudo systemctl start neo4j-prod

```

```bash
# 启动 Dev 服务
sudo systemctl start neo4j-dev

```

```bash
# 查看 Prod 服务状态
sudo systemctl status neo4j-prod

```

```bash
# 查看 Dev 服务状态
sudo systemctl status neo4j-dev

```

```bash
# 设置 Prod 服务开机自启
sudo systemctl enable neo4j-prod

```

```bash
# 设置 Dev 服务开机自启
sudo systemctl enable neo4j-dev

```

```bash
# 停止服务 (示例: 停止 prod)
# sudo systemctl stop neo4j-prod

```

```bash
# 重启服务 (示例: 重启 dev)
# sudo systemctl restart neo4j-dev

```

# 5. 总结

通过以上步骤，我们成功地在离线的 Ubuntu 服务器上部署了两个独立的 Neo4j 实例，并配置了不同的端口供外部访问。关键在于为每个实例提供独立的安装、数据和日志目录，并仔细配置 neo4j.conf 文件中的监听地址、端口和目录路径。同时，确保运行环境满足 Neo4j 的要求（特别是 Java 11），并掌握了通过检查日志、访问地址和端口监听状态来排查启动问题的基本方法。使用 systemd 服务管理 Neo4j 实例是推荐的最佳实践，可以大大简化运维工作。

# 附录

Neo4j下载地址：

[https://neo4j.com/deployment-center/](https://neo4j.com/deployment-center/)







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>