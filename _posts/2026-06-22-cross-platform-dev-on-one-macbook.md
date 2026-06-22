---
title: "一台 MacBook Pro 跑通 Linux/Windows 开发与测试"
date: "2026-06-22 21:00:00 +0800"
categories: [Tooling, Environment]
tags: [macOS, Apple Silicon, Linux, Windows, OrbStack, VMware Fusion, UTM, Visual Studio, ARM64, SSH, Blade]
layout: post
published: true
---

> 一台 Apple Silicon 的 MacBook Pro，怎么同时当 Linux 和 Windows 的开发测试机？Linux 用 OrbStack 跑容器和虚拟机，Windows 用 VMware Fusion / UTM 装 ARM 版虚拟机、配 Visual Studio 2022 + 2026 Build Tools（含 ARM 编译器）和 OpenSSH Server。把三套目标平台收进一台机器的实战记录。

## 起因：一个构建系统逼出来的三平台需求

最近一段时间我在给 [blade](https://github.com/chen3feng/blade-build) 补 Windows 支持，深到 MSVC 工具链的各种细节：DLL 导出（见 [《Windows DLL 导出之痛》]({% post_url 2026-06-07-windows-dll-export-pain %}) 和 [《符号导出三国志》]({% post_url 2026-06-07-symbol-export-windows-linux-macos %})）、调试信息、覆盖率、PGO、clang-cl 路由……为了不闭门造车，我还把 blade 直接拿去构建真实的开源项目 dogfooding——Notepad++、7-Zip、PuTTY 这些。

这一下问题就来了：blade 是个跨 Linux / macOS / Windows 的构建系统，改一处 Windows 逻辑，得确认**三个平台都没退化**；dogfooding 又要求有**真实的 MSVC 环境**能跑能链能测。而我手头主力是一台 **Apple Silicon 的 MacBook Pro**。

于是目标很明确：**一台 Mac，同时是 macOS、Linux、Windows 三套目标平台的开发测试机**，而且因为是 ARM 芯片，还得顺带覆盖 ARM/x86 两种架构。

下面是我最后落地的方案。

## 总体思路

| 目标平台 | 方案 | 架构覆盖 |
|---|---|---|
| macOS | 原生 | ARM64（必要时 Rosetta 测 x86_64） |
| Linux | **OrbStack** 容器 + 轻量虚拟机 | ARM64 原生 + x86_64 模拟 |
| Windows | **VMware Fusion / UTM** 装虚拟机 | ARM64 原生 + x86/x64 交叉与模拟 |

核心取舍：macOS 原生不用说；Linux 不开重虚拟机，用 OrbStack 这种贴着原生跑的方案把开销压到最低；Windows 因为要完整的 MSVC 工具链和真实系统行为，老老实实开一台虚拟机，但用 SSH 把它变成「终端里的一台远程机」，免得老去点 GUI。

## Linux：OrbStack 跑容器和虚拟机

[OrbStack](https://orbstack.dev/) 是 macOS 上的 Docker 容器 + Linux 虚拟机方案，定位类似 Docker Desktop + multipass，但**快得多、省电得多**。对我这种「随时起一个干净 Linux 验证一下」的用法非常合适。

几个我看重的点：

- **启动快、占用低**。起一台 Linux machine 是秒级的，内存按需占用、空闲几乎不耗电——比常驻的重虚拟机友好太多，对笔记本续航很关键。
- **ARM 原生 + x86 模拟一把抓**。M 系列上它默认跑 ARM64，但能借 Rosetta 跑 `amd64` 容器/二进制。对 blade 来说，这意味着**同一台机器就能测 Linux 的两种架构**：ARM64 验证原生行为，x86_64 模拟跑一遍确认没有架构假设。
- **零配置的网络与文件共享**，外加自动的 SSH 和 `machine.orb.local` 域名，`orb` / `orbctl` 命令行直接进。Mac 和 Linux 之间互访文件基本无感。

典型用法：

```bash
# 装好后，创建不同发行版的机器
orb create ubuntu          # 一台 Ubuntu（ARM64）
orb create -a amd64 ubuntu jammy-x86   # 一台 x86_64 的，用来测架构差异

# 进去
orb -m ubuntu              # 或 ssh ubuntu@orb

# 在 Mac 侧直接对某台机器跑命令
orb run -m jammy-x86 uname -m   # x86_64
```

把 blade 的源码挂进去，Linux 侧的构建/测试就跑起来了。需要测「干净环境」时，删掉重建一台也只是几秒钟的事。

> 容器和虚拟机的取舍：纯构建/测试用容器就够，且更轻；需要 systemd、内核模块、或更接近完整系统的行为时用它的 Linux machine（轻量 VM）。我两者都用——CI 风格的验证走容器，需要折腾系统级行为的走 machine。

## Windows：虚拟机 + MSVC 工具链 + SSH

Windows 这边没法走「贴着原生跑」的捷径，得开一台真正的虚拟机。Apple Silicon 上能装的是 **Windows 11 on ARM**（微软已提供 ARM64 ISO），x86/x64 程序由系统的模拟层跑。

### 选 hypervisor：VMware Fusion 还是 UTM

- **VMware Fusion**：个人使用现在免费了，对 Apple Silicon 上的 Windows 11 ARM 支持成熟，网络/共享/快照都顺手。默认 NAT 网段是 `172.16.x.x`，宿主 Mac 能直接访问 guest（后面 SSH 要用到）。
- **UTM**：开源、基于 QEMU。既能虚拟化 ARM Windows，也能**纯模拟 x86 Windows**（慢，但偶尔需要验证 x86 原生系统时有用）。

我主力用 Fusion 跑 ARM64 的 Windows 11，UTM 留作需要 x86 系统时的备选。

### 装 Visual Studio 2022 + 2026 Build Tools（含 ARM 编译器）

不需要完整 IDE，装 **Build Tools** 就够——它只给 MSVC 工具链、SDK 和链接器，没有 devenv 那套 GUI，体积小、适合当构建机。

我同时装了 **2022 和 2026 两套 Build Tools**，目的是**测 blade 对不同 MSVC 版本的兼容性**——工具链跨大版本时，默认开关、调试信息格式、链接器行为都可能有差异，dogfooding 时多版本并存能尽早暴露问题。VS 的 Build Tools 支持多版本并排安装，互不干扰。

装的时候要勾上 **ARM 相关的编译器组件**，这是 ARM 虚拟机的关键：

- **ARM64 native**：在 ARM64 Windows 上原生跑的 `cl.exe`，直接编 ARM64 目标，不经模拟，速度最好。
- **ARM64/ARM64EC + x64/x86 交叉编译器**：一台机器上把 blade 要覆盖的 target 架构都配齐。

> 小心 host/target 的组合：ARM64 上既有「ARM64 host → ARM64 target」的原生工具链，也有「x64 host（模拟）→ ARM64 target」的交叉工具链。能用原生就别用模拟 host，构建快很多。`vcvarsall.bat` / `vcvarsarm64.bat` 选对那套环境。

### 装 OpenSSH Server：把 Windows 变成「终端里的远程机」

最后一步、也是体验提升最大的一步：给 Windows 装上 **OpenSSH Server**。这样我能从 Mac 终端 `ssh` 进去跑 blade 构建、把 Windows 纳入脚本流水线，而不用一直切到 GUI 里点。

Windows 10/11 自带 OpenSSH 服务端（可选功能），装好启用、放行 22 端口即可。我把这套（安装服务端 + 授权公钥）写进了我的 [devenv](https://github.com/chen3feng/devenv) 仓库，管理员 PowerShell 里两条命令搞定：

```powershell
# 1) 安装并启用 sshd（开机自启 + 放行防火墙 22）
.\windows\Install-SshServer.ps1 -DefaultShell 'C:\Program Files\PowerShell\7\pwsh.exe'

# 2) 授权 Mac 的公钥
.\windows\Authorize-SshKey.ps1 'ssh-ed25519 AAAA... me@mac'
```

然后从 Mac：

```bash
ssh cf@172.16.86.128      # Fusion 的 NAT 地址
```

这里有**两个真实的坑**，写脚本时专门处理了：

1. **没设密码的账户连不上**。Windows 默认禁止「空密码账户」走网络登录（安全策略 `LimitBlankPasswordUse`），所以密码认证这条路在没设密码时根本走不通——**密钥认证是唯一的路**。这反而更安全，正好顺势上密钥。
2. **管理员账户的 authorized_keys 位置不一样**。普通用户放 `~\.ssh\authorized_keys`，但**管理员账户** sshd 只认 `C:\ProgramData\ssh\administrators_authorized_keys`，且该文件 ACL 必须只给 Administrators + SYSTEM，否则被忽略。更隐蔽的是：判断「是不是管理员」要看**组成员身份**，不能看「当前会话是否提权」——UAC 过滤后的非提权 token 会把 Administrators 组摘掉，照它判断会把公钥写错地方，导致登录一直失败。脚本里用 `Get-LocalGroupMember` 直接查成员身份，绕开了这个陷阱。

## 串起来：一台机器的日常

配齐之后，日常长这样：

- 代码在 Mac 上编辑（或各平台各自 checkout）。
- Linux 验证：`orb run -m ubuntu blade test ...`，要测架构差异就切到 x86_64 那台。
- Windows 验证：`ssh` 进 Fusion 的 Windows VM，`blade build` / `blade test` 跑 MSVC，多版本 Build Tools 之间切换确认兼容。
- macOS 原生直接本机跑。

改一处 blade 的 Windows 逻辑，三平台的回归在同一台笔记本上、基本都在终端里就能转一圈；dogfooding 那几个真实项目（Notepad++ / 7-Zip / PuTTY）也是在这套 Windows VM 里构建的。

## 一些经验

- **能原生别模拟**：OrbStack 的 ARM64 容器、Windows 的 ARM64-host 工具链，都比走模拟快得多；模拟（x86 容器、x64-host 交叉、UTM 纯模拟）留给「确实要验证那个架构」的场景。
- **快照是命根子**：折腾工具链、改系统配置前先打快照。Build Tools 多版本并存、注册表改 sshd 默认 shell 这类操作，出问题能秒回滚。
- **网络早确认**：连不上 Windows VM 多半是 VM 网络模式问题。Fusion 的 NAT 一般宿主可直连 `172.16.x.x`；要从局域网别的机器访问就得桥接或端口转发。
- **时间/时区**：虚拟机和容器的时钟漂移会让构建时间戳、证书校验、甚至「未来日期」类逻辑出怪事（这个我在博客部署上吃过亏），值得顺手校一下。

## 结语

「一台 MacBook Pro 同时开发测试 Linux 和 Windows」听起来折腾，落地下来其实就三块拼图：**OrbStack 把 Linux 做成秒起秒删的轻量资源，VM + Build Tools 给出真实的 MSVC（含 ARM）环境，SSH 把 Windows 收进终端**。对我这种要在三平台之间反复横跳验证构建系统的人，这套组合把「切平台」的成本压到了很低——大部分时候，不用离开终端。
