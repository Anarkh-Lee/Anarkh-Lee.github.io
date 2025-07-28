---
layout: post
title: '在 Ubuntu 上离线安装 OpenJDK 11 并设置为默认版本'
subtitle: "在 Ubuntu 上离线安装 OpenJDK 11 并设置为默认版本"
date: 2025-04-06
author: Anarkh-Lee
cover: './assets/img//cover_img/Ubuntu.png'
tags: Ubuntu



---

在软件开发和运维过程中，我们有时需要在同一台服务器上管理多个 Java 开发工具包（JDK）版本。例如，系统可能已经通过包管理器安装了较新的 OpenJDK 17，但某些项目或应用仍需要使用 OpenJDK 11。此外，在无法直接访问互联网的环境下，离线安装成为必要选择。本文将详细介绍如何在已经安装了 OpenJDK 17 的 Ubuntu 系统上，通过离线方式安装 OpenJDK 11，并将其配置为系统默认的 JDK 版本。

# 1. 前提条件

1. 获取 OpenJDK 11 安装包: 你需要预先在一台可以联网的机器上下载适用于 Linux x64 的 OpenJDK 11 压缩包。推荐从 Adoptium (Temurin) 下载 .tar.gz 或 .tar.xz 格式的归档文件。本文以 java-11-openjdk-11.0.23.0.9-2.portable.jdk.el.x86_64.tar.xz 为例。
2. 传输安装包: 将下载好的 OpenJDK 11 压缩包传输到目标 Ubuntu 服务器上，例如通过 U 盘、 scp 或其他文件传输方式。假设文件已放置在用户的主目录 ( ~/ ) 下。

# 2. 安装步骤

## 2.1 创建 JDK 安装目录

标准的做法是将 JDK 安装在 /usr/lib/jvm 目录下。如果该目录不存在，使用以下命令创建：

```Bash
sudo mkdir -p /usr/lib/jvm
```

## 2.2 解压 OpenJDK 11 压缩包

将 .tar.xz 格式的压缩包解压到 /usr/lib/jvm 目录。使用 tar 命令的 -xJf 选项进行解压。

```Bash
sudo tar -xJf ~/java-11-openjdk-11.0.23.0.9-2.portable.jdk.el.x86_64.tar.xz -C /usr/lib/jvm
```

解压后，会在 /usr/lib/jvm 下生成一个包含版本号的目录（例如 jdk-11.0.23.0.9-2.portable.jdk.el.x86_64 ）。为了方便管理，建议将其重命名为一个更简洁的名称，如 jdk-11 。

首先，确认解压后的实际目录名（可以使用 ls /usr/lib/jvm 查看），然后执行重命名（请将下面的源目录名替换为实际名称）：

```Bash
sudo mv /usr/lib/jvm/java-11-openjdk-11.0.23.0.9-2.portable.jdk.el.x86_64 /usr/lib/jvm/jdk-11
```

## 2.3 使用 update-alternatives 配置系统链接

update-alternatives 是 Debian/Ubuntu 系统中用于管理多个版本软件符号链接的工具。我们需要用它来告知系统新安装的 JDK 11 的位置，并设置其优先级。优先级是一个整数，数值越大表示优先级越高。我们将为 JDK 11 设置一个较高的优先级（如 1111），使其能成为默认选项。

- **注册 java 命令:**

```Bash
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-11/bin/java 1111
```

- **注册 javac 命令 (Java 编译器):**

```Bash
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk-11/bin/javac 1111
```

- **(可选) 注册 jar 命令 (打包工具):**

```Bash
sudo update-alternatives --install /usr/bin/jar jar /usr/lib/jvm/jdk-11/bin/jar 1111
```

你可以根据需要，为 JDK bin 目录下的其他工具（如 javadoc , jps 等）也执行类似的 --install 操作。



## **这里，如果没有安装jdk17的话就可以结束了**



## 2.4 选择默认 JDK 版本

执行以下命令，系统会列出所有已通过 update-alternatives 注册的 java 和 javac 版本，并允许你交互式地选择默认版本。

- **选择默认 java:**

```Bash
sudo update-alternatives --config java
```

- **选择默认 javac:**

```Bash
sudo update-alternatives --config javac
```

- **(可选) 选择默认 jar:**

```Bash
sudo update-alternatives --config jar

```

在每个命令的提示中，输入与 /usr/lib/jvm/jdk-11/bin/java (或 javac , jar ) 对应的编号，然后按 Enter 确认。如果 JDK 11 的优先级最高，它可能已经是“自动模式”下的默认选项，但手动选择可以确保设置正确。

## 2.5 验证安装结果

运行以下命令检查当前默认的 Java 版本：

```Bash
java -version
javac -version

```

如果输出显示为 OpenJDK 11 的版本信息，则表示配置成功。

## (可选但推荐) 配置 JAVA_HOME 环境变量

许多 Java 应用程序和开发工具依赖 JAVA_HOME 环境变量来定位 JDK 的安装路径。虽然 update-alternatives 处理了命令行的链接，但设置 JAVA_HOME 是一个好习惯。

- **全局配置 (推荐，对所有用户生效):** 编辑 /etc/environment 文件：

  - ```Bash
    sudo nano /etc/environment
    
    ```

  - 在该文件中添加或修改以下行：

  - ```Plain
    JAVA_HOME="/usr/lib/jvm/jdk-11"
    
    ```

  - 保存并关闭文件（在 nano 编辑器中，按 Ctrl+X ，然后按 Y 确认，最后按 Enter ）。 注意： 此更改需要重新登录或重启系统才能完全生效。

- **用户个人配置 (仅对当前用户生效):** 编辑用户主目录下的 .bashrc 文件：

  - ```Bash
    nano ~/.bashrc
    
    ```

  - 在文件末尾添加：

  - ```Bash
    export JAVA_HOME="/usr/lib/jvm/jdk-11"
    export PATH=$JAVA_HOME/bin:$PATH
    
    ```

  - 保存并关闭文件。然后执行 source ~/.bashrc 使配置在当前终端会话立即生效。

# 3. 总结

通过以上步骤，我们成功地在 Ubuntu 系统上离线安装了 OpenJDK 11，并使用 update-alternatives 工具将其设置为了默认的 JDK 版本。同时，我们也配置了 JAVA_HOME 环境变量，以确保应用程序能够正确找到 JDK。如果将来需要切换回 OpenJDK 17 或其他已安装的版本，只需再次运行 sudo update-alternatives --config java 和 sudo update-alternatives --config javac 命令，并选择相应的版本即可，非常方便。







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>