---
layout: post
title: 'Neo4j在win下安装教程（docker环境）'
subtitle: "Neo4j在win下安装教程（docker环境）"
date: 2025-01-01
author: Anarkh-Lee
cover: './assets/img//cover_img/Neo4j.png'
tags: 图数据库



---

# 1. 安装命令

## 1.1 基于正式neo4j安装--不用

```bash
docker run --name neo4j-container -p 7474:7474 -p 7687:7687 -d neo4j
```

![]({{ '/assets/img/neo4j_img/03.png' | prepend: '' }})
![](.\img\neo4j_img\03.png)

## 1.2 基于community安装

需要部署两个Neo4j，一个正式库prod，一个测试库dev。

neo4j默认监听7474（HTTP-也就是浏览器端口）和7687（Bolt-也就是服务器接口）端口。

首先需要明确的是，通过docker部署的服务，**容器内部的7474和7687端口不会冲突**，因为Docker的端口映射机制是基于**宿主机****端口->容器端口**的隔离。

### 1.2.1 端口映射原理

- **容器内部端口：**每个 Neo4j 容器内部默认监听 `<font style="background-color:rgb(242,243,245);">7474</font>`（HTTP）和 `<font style="background-color:rgb(242,243,245);">7687</font>`（Bolt）端口。
- **宿主机****端口：**通过 `<font style="background-color:rgb(242,243,245);">-p</font>` 参数将宿主机的端口映射到容器的端口。例如：
  - 第一个容器：`<font style="background-color:rgb(242,243,245);">-p 7474:7474</font>`（宿主机7474 → 容器7474）
  - 第二个容器：`<font style="background-color:rgb(242,243,245);">-p 7475:7474</font>`（宿主机7475 → 容器7474）

**关键点：** 即使容器内部都使用 `<font style="background-color:rgb(242,243,245);">7474</font>` 和 `<font style="background-color:rgb(242,243,245);">7687</font>`，只要宿主机的映射端口不同（如 `<font style="background-color:rgb(242,243,245);">7475</font>` 和 `<font style="background-color:rgb(242,243,245);">7688</font>`），两个容器就能同时运行且互不冲突。

### 1.2.2 两种创建方式

#### 1.2.2.1 绑定挂载（windows绝对路径-需要手动创建路径）

```plain
docker run --name neo4j-dev -p 17475:7474 -p 17688:7687 -v C:\neo4j\dev:/data -v C:\neo4j\dev:/logs -d neo4j :community
```

- 优点：
  - 直观可控：可以直接在宿主机的文件系统中查看和修改数据文件（如 `<font style="background-color:rgb(242,243,245);">C:\neo4j\data2</font>`）。
  - 适合调试：方便直接修改配置文件（如 `<font style="background-color:rgb(242,243,245);">neo4j.conf</font>`）或检查日志文件（如 `<font style="background-color:rgb(242,243,245);">neo4j.log</font>`）。
  - 完全控制目录位置：自由选择宿主机的存储路径。
- 缺点：
  - 需手动处理目录权限：如果宿主机目录权限设置不当，容器可能无法写入。
  - 跨平台兼容性问题：Windows 路径格式（如 `<font style="background-color:rgb(242,243,245);">C:\...</font>`）在 Docker 中需要特别处理，可能与其他系统不兼容。
  - 依赖宿主目录存在性：必须预先手动创建目录，否则启动失败（尤其是 Windows）。

**适用场景：**

- 需要直接操作宿主机文件（如开发阶段修改配置文件或分析日志）。
- 数据需存放在宿主机的特定位置（如已有存储系统需要挂载）。

#### 1.2.2.2 Docker卷（不需要手动创建路径）-本次使用的方案

正式：

```plain
docker run --name neo4j-prod -p 17474:7474 -p 17687:7687 -v neo4j_prod_data:/data -v neo4j_prod_logs:/logs -d neo4j:community
```

测试：

```plain
docker run --name neo4j-dev -p 17475:7474 -p 17688:7687 -v neo4j_dev_data:/data -v neo4j_dev_logs:/logs -d neo4j:community
```

- 优点：
  - 自动管理：Docker 会自动创建卷并处理文件权限，无需手动干预。
  - 跨平台一致性：路径格式统一（如 `<font style="background-color:rgb(242,243,245);">/data</font>`），避免 Windows/Linux 路径差异问题。
  - 适合生产环境：数据由 Docker 托管，更安全且支持加密、备份等高级功能。
  - 容器无缝迁移：容器重建或迁移时，卷可以快速复用。
- 缺点：
  - 隐藏文件位置：默认卷存储在 Docker 的私有路径中（如 `<font style="background-color:rgb(242,243,245);">C:\ProgramData\Docker\volumes</font>`），需要命令行才能查看。
  - 不直接修改文件：需通过容器内部或 Docker 命令访问数据，对普通用户不够直观。

适用场景：

- 生产环境或需要自动化部署的场景。
- 无需直接访问底层数据文件，注重数据安全性和一致性。

**推荐使用哪种？--本次直接使用Docker卷的方式进行创建**

- 推荐：生产环境使用 Docker 卷，开发环境使用绑定挂载：
  - 生产环境 → 优先选 Docker 卷 安全便捷，避免路径和权限问题，适合长期运行的稳定服务。
  - 开发环境 → 优先选绑定挂载 方便直接查看和修改配置文件、日志文件，适合调试和测试。

# 2. 访问

http://localhost:7474

第一次进入页面，需要输入用户名密码neo4j/neo4j，并且需要设置新密码

![]({{ '/assets/img/neo4j_img/04.png' | prepend: '' }})
![](.\img\neo4j_img\04.png)

# 3. 服务起上的用户密码

neo4j/neo4jneo4j

# 4. 停止和启动 Neo4j 容器

要停止正在运行的 Neo4j 容器，可以使用以下命令：

```bash
docker stop neo4j-container
```

要再次启动容器，使用：

```bash
docker start neo4j-container
```







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>