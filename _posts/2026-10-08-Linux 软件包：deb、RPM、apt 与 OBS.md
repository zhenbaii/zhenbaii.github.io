---
layout:       post
title:        "Linux 软件包：deb、RPM、apt 与 OBS"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 操作系统
tags:
    - "操作系统"
---
在 Linux 上安装软件时，经常会碰到 `.deb`、`.rpm`、`apt`、`dnf`、OBS 这些名字。它们分别管不同的事：**软件打成什么包、用什么工具安装、这些包是怎么做出来的。**

先看一个常见的流程：

```text
维护者准备源码和打包说明
          ↓
构建工具编译、打包，生成 .deb 或 .rpm
          ↓
把包放进软件源
          ↓
用户用 apt、dnf 或 zypper 下载并安装
```

“软件源”可以先理解成一个提供软件包下载的仓库。里面除了包文件，还有一份目录，记录有哪些软件、版本是多少、需要哪些依赖。

## 从 Ubuntu 上安装 vim 说起

在 Ubuntu 上，通常这样安装 Vim：

```bash
sudo apt update
sudo apt install vim
```

第一句更新本机保存的软件源目录，让 `apt` 知道现在有哪些包可以安装。**`apt update` 本身不会把已安装的软件升级一遍。**

第二句才是安装。`apt` 会找到 `vim` 对应的软件包，检查它还需要哪些包，把缺少的依赖一起下载下来，再交给底层工具 `dpkg` 完成安装。

比如，一个程序需要某个运行库才能工作，这个运行库就是它的“依赖”。有了 `apt`，通常不用自己逐个寻找和安装这些库。

因此，虽然命令里没有出现 `.deb`，Ubuntu 在这一步下载和安装的仍然是 `.deb` 文件。

## .deb 和 .rpm：两种软件包格式

软件包里通常装着程序、库、配置等文件，以及包名、版本、依赖关系等信息。安装工具根据这些信息，把文件放到合适的位置，并记录这个包已经安装。

`.deb` 和 `.rpm` 是两种不同的打包格式：

| 包文件格式  | 常见的使用者                   |
| ------ | ------------------------ |
| `.deb` | Debian、Ubuntu、Linux Mint |
| `.rpm` | Fedora、RHEL、openSUSE     |

它们解决的是同一类问题，只是文件格式和配套工具不同。

还要注意，后缀相同不代表一定能通用。给 Fedora 制作的 `.rpm`，不一定能直接装到 openSUSE 上；给某个 Ubuntu 版本制作的 `.deb`，也不一定适用于另一个版本。包还要匹配系统的依赖、版本和 CPU 架构。

## dpkg、rpm 和 apt、dnf、zypper 分别做什么

安装工具可以分成两类。一类负责处理具体的包文件；另一类帮你从软件源找包、处理依赖，并安排安装和升级。

| 系统 | 包文件 | 处理包文件的底层工具 | 日常常用的包管理器 |
|---|---|---|---|
| Ubuntu / Debian | `.deb` | `dpkg` | `apt` |
| Fedora | `.rpm` | `rpm` | `dnf` |
| openSUSE | `.rpm` | `rpm` | `zypper` |

以 Debian 系为例：手里已经有一个 `.deb` 文件时，`dpkg` 可以安装它，也会检查依赖是否满足。但如果少了依赖，`dpkg` 不会自己去软件源下载补齐。`apt` 会负责这部分工作。

RPM 这边也类似。`rpm` 能安装、卸载和查询 RPM 包，日常从软件源装软件则通常交给 `dnf` 或 `zypper`。它们使用 RPM 的底层机制完成安装。

所以，同样是安装 Vim，在不同系统上会写成：

```bash
# Ubuntu / Debian
sudo apt install vim

# Fedora
sudo dnf install vim

# openSUSE
sudo zypper install vim
```

这三条命令用途相同，只是对应的系统和工具不同。旧一些的 RHEL/CentOS 教程里还会看到 `yum`；`dnf` 是它的后继。

## OBS：维护者用来制作软件包的平台

用户安装之前，总得有人先把软件包做出来。

维护者需要拿到源码，准备编译环境，安装编译所需的依赖，运行构建命令，再把结果打成包。如果要为多个系统、多个 CPU 架构提供软件包，这些工作还要分别做一遍。

**OBS（Open Build Service）就是用来组织和自动执行这些工作的构建平台。** 项目可以部署自己的 OBS 服务，让构建服务器按照提交的打包说明制作软件包，并把结果发布到软件源。[OBS 官方介绍](https://openbuildservice.org/)

对于 RPM 包，这份打包说明通常是一个 `.spec` 文件。它会写清楚：

- 包的名称、版本和源码位置；
- 编译时需要什么工具和库；
- 怎样编译、整理安装文件；
- 最终把哪些文件装进包里，以及运行时需要哪些依赖。

可以把 Spec 理解为制作 RPM 包的步骤说明。OBS 准备好环境后，会调用 `rpmbuild` 等工具执行这些步骤。OBS 也支持构建 `.deb`，只是需要使用相应的打包文件。[OBS 支持的包格式与构建说明](https://openbuildservice.org/help/manuals/obs-user-guide/cha-obs-package-formats)

包格式和构建平台是分别选择的：使用 RPM，并不意味着一定使用 OBS。类似地，Ubuntu 的许多构建任务由 Launchpad 承担，但 `.deb` 文件也可以在本机或其他构建平台上生成。

对于普通用户，安装时主要关心软件源里有没有所需的包。运行 `apt install` 或 `dnf install` 时，通常是在安装已经构建好的软件，无须自己去 OBS 提交构建任务。

## 放到 openRuyi 的开发流程里看

openRuyi 的开发者文档采用 RPM Spec 打包，并通过 OBS 完成本地或服务器上的构建。它的构建服务是 [build.openruyi.cn](https://build.openruyi.cn/)。开发者会用到 `osc`，这是与 OBS 交互的命令行客户端，可以用来拉取项目、提交修改、查看构建状态，也可以配合构建工具在本机试着构建软件包。[openRuyi 打包与修包快速入门](https://www.openruyi.cn/zh-Hans/docs/guide/quick-start-for-developers/)

这里几个名字的关系是：

```text
源码 + Spec        → 要编译什么、怎么打包
osc                → 开发者用它操作 OBS 项目
OBS                → 组织构建，生成 RPM 包并提供软件源
系统的包管理器      → 用户用它下载、安装这些包
```

文档中还提到了 `qemu-user`。这是因为在其他架构的机器上构建 RISC-V 软件时，构建过程中可能需要运行 RISC-V 程序，而 `qemu-user` 可以模拟执行它们。openRuyi 的开发者指南提到 RVA23 要求，因此用于这类模拟构建的 QEMU 也需要支持相应指令。[openRuyi 构建环境说明](https://www.openruyi.cn/zh-Hans/docs/guide/quick-start-for-developers/)

至于在 openRuyi 系统里日常安装软件应该用哪条命令，要看它提供的包管理器和使用文档。仅凭“使用 RPM”和“通过 OBS 构建”，还不能判断应该敲 `dnf` 还是 `zypper`。
