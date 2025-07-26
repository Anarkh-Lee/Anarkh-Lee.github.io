---
layout: post
title: 'Neo4j社区版在win下安装教程（非docker环境）'
subtitle: "Neo4j社区版在win下安装教程（非docker环境）"
date: 2025-01-19
author: Anarkh-Lee
cover: './assets/img//cover_img/Neo4j.png'
tags: 图数据库



---

要在 Windows 10 上安装 Neo4j 社区版数据库并且不使用 Docker Desktop，你可以按照以下步骤操作：

# 1. 安装 Java Development Kit (JDK)

Neo4j 需要 Java 运行环境。推荐安装 JDK 17 或 JDK 11（请根据你下载的 Neo4j 版本查看具体的兼容性要求）。

Neo4j与JDK版本对应关系：

![](E:/MyNotes/06%20%E6%99%BA%E8%83%BD%E5%8C%96/01%20%E5%9B%BE%E6%95%B0%E6%8D%AE%E5%BA%93/img/01.png)

- 检查是否已安装 JDK: 打开命令提示符（CMD）或 PowerShell，输入以下命令：

```bash
java -version
```

   如果显示了 Java 版本信息（例如 11.x.x 或 17.x.x），并且版本符合 Neo4j 的要求，则可以跳过此步。

- 下载并安装 JDK: 如果未安装或版本不符，请前往 Oracle JDK 下载页面 或其他 OpenJDK 发行版（如 Adoptium Temurin）下载适用于 Windows 的 JDK 安装程序（推荐 .msi 安装包）。

按照安装向导完成安装。安装程序通常会自动设置 JAVA_HOME 环境变量和系统 Path 。

- 验证安装: 重新打开一个新的命令提示符窗口，再次运行 java -version 确认安装成功。

# 2. 下载 Neo4j 社区版

- 访问 Neo4j 官方下载页面： https://neo4j.com/download-center/#community
- 找到 Neo4j Community Server 部分。
- 选择适用于 Windows 的 .zip 压缩包进行下载。

![]({{ '/assets/img/neo4j_img/02.png' | prepend: '' }})
![](.\img\neo4j_img\02.png)

# 3. 解压 Neo4j

- 将下载的 .zip 文件解压到你希望安装 Neo4j 的目录。例如，你可以解压到 C:\neo4j-community-5.x.x (请将 5.x.x 替换为你下载的具体版本号)。

# 4. (可选) 配置环境变量

为了方便在任何目录下运行 Neo4j 命令，建议配置环境变量：

- 设置 NEO4J_HOME:
  - 右键点击“此电脑” -> “属性” -> “高级系统设置” -> “环境变量”。
  - 在“系统变量”下点击“新建”。
  - 变量名： NEO4J_HOME
  - 变量值：你解压 Neo4j 的路径 (例如 C:\neo4j-community-5.x.x )
  - 点击“确定”。
- 将 Neo4j bin 目录添加到 Path:
  - 在“系统变量”中找到 Path 变量，选中并点击“编辑”。
  - 点击“新建”，添加 %NEO4J_HOME%\bin 。
  - 点击“确定”保存所有更改。

# 5. 启动 Neo4j 服务器

- 打开一个新的命令提示符（CMD）或 PowerShell 窗口（如果配置了环境变量，需要新窗口才能生效）。
- 导航到 Neo4j 的 bin 目录（如果未配置环境变量）：

```bash
cd C:\neo4j-community-5.x.x\bin
```

- 运行以下命令启动 Neo4j（它会在当前控制台窗口运行，关闭窗口会停止服务）：

```bash
neo4j console
```

   或者，如果你想以后台服务方式运行（首次可能需要管理员权限来安装服务）：

```bash
neo4j install-service
```

   然后启动服务：

```bash
neo4j start
```

# 6. 访问 Neo4j Browser

- 当 Neo4j 启动后，在浏览器中打开 http://localhost:7474 。
- 首次连接时，默认用户名为 neo4j ，默认密码为 neo4j 。
- 系统会强制要求你修改初始密码。请设置一个新密码并妥善保管。

1. 停止 Neo4j 服务器

- 如果使用 neo4j console 启动：在命令提示符窗口中按 Ctrl + C 。
- 如果使用 neo4j start 启动：在命令提示符（可能需要管理员权限）中运行：

```bash
neo4j stop
```

   如果不再需要服务，可以卸载：

```bash
neo4j uninstall-service
```

注意事项:

- 防火墙: Windows 防火墙可能会阻止 Neo4j 的网络连接。如果无法访问 http://localhost:7474 ，请检查防火墙设置，确保 Java(TM) Platform SE binary 或特定端口（如 7474 和 7687）被允许通过。
- 端口冲突: 如果 7474 或 7687 端口已被其他应用程序占用，Neo4j 将无法启动。你需要修改 Neo4j 配置文件 ( conf/neo4j.conf ) 中的端口设置或停止占用端口的程序。

这样，你就成功在 Windows 10 上安装并运行了 Neo4j 社区版，而无需使用 Docker。







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>