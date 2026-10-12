---
title: '旧 Wacom 遇上新 macOS：CTH-690 卡在「等待同步」'
description: 'CTH-690 在 macOS 26 上一直显示等待同步。最后发现是旧驱动里的系统版本判断，改完还得重签名、给终端开 App 管理权限。'
pubDate: 2026-07-26T12:00:00+08:00
tags: ['Wacom', 'macOS', 'Apple Silicon']
---

手上有块老的 Wacom CTH-690，插到 Mac 上，Wacom Center 一直显示「等待同步」，就停在那儿不动。驱动装了，后台进程也在。

我第一反应是驱动没卸干净。这板子比较老，我特意装的是还支持它的旧版驱动，之前好像也装过新版，怀疑是两套混在一起了。

结果不是。问题出在旧驱动里写死的一段系统版本判断，在 macOS 26 上直接不往下初始化了。把这段改掉，再重签名、补权限，板子就认出来了，压感也有。

这次是让 Codex 帮着一起查的。环境是 **CTH-690、Apple Silicon Mac、macOS 26.5.2、Wacom 驱动 6.4.10-3**。要改驱动二进制和签名，Wacom 官方不支持这么干，换个版本组合不一定适用。

## 先看看卡在哪

「等待同步」这四个字基本没信息量。我先看了几样东西：USB 能认到 CTH-690；装的驱动只有 6.4.10-3 一套；Wacom 的几个后台服务都在跑。

所以板子没坏，但进程在跑也不代表驱动真的好了。

然后去翻官方发行说明，发现是个两头够不着的情况：

| 驱动版本 | 官方写的 macOS 范围 | CTH-690 |
|---|---|---|
| 6.4.10-3 | macOS 12–15 | 支持 |
| 6.4.11-1 | macOS 13–15、26 | 移除支持 |

6.4.11-1 的说明里直接把 CTH-690 列进了停止支持的设备，让你去用 6.4.10-3。旧驱动认板子但不认系统，新驱动认系统但不认板子。[6.4.10-3 发行说明](https://cdn.wacom.com/u/productsupport/drivers/mac/professional/releasenotes/Mac_6.4.10-3.html)、[6.4.11-1 发行说明](https://cdn.wacom.com/u/productsupport/drivers/mac/professional/releasenotes/Mac_6.4.11-1.html)

为了一块数位板降级系统不现实，只能看旧驱动到底卡在哪。

想自己查的话，下面几条命令都是只读的，分别看系统版本、CPU 架构、USB 设备和当前用户的服务：

```bash
sw_vers
uname -m
system_profiler SPUSBDataType
launchctl print "gui/$(id -u)"
```

后两条输出很长，搜 `Wacom` 或 `Tablet` 就行。USB 里有没有设备，和驱动服务加载到哪一步，是两回事，分开看。

## 不是混装，是版本判断

我还专门怀疑过 `Wacom_IOManager`。它名字和别的组件不太一样，位置也不在一起，看着很像卸载时漏下来的。

后来把装好的文件和官方安装包逐个比 SHA-256，下面五个可执行文件全都对得上，没有哪个是别的版本混进来的：

- Wacom Center
- TabletDriver
- WacomTabletDriver
- WacomTouchDriver
- Wacom_IOManager

继续搜的时候翻到一个开源项目 [wacom-macos27-driver-fix](https://github.com/MohamadAshrafSalama/wacom-macos27-driver-fix)。它解决的是另一个版本组合（macOS 27、驱动 6.4.13-4、CTL-4100），跟我的情况对不上，补丁不能直接拿来用。但它给了个关键思路：Wacom 驱动里有硬编码的系统版本检查，可能还没碰到设备就先拒绝了。

顺着这个思路去看 6.4.10-3 的二进制，五个组件的 arm64 部分都有这么一段：

```asm
mov w0, #1
mov w1, #16
mov w2, #10
mov w3, #0
```

这是在给版本检查准备参数，阈值是 `16.10`。注意这里是版本号的两段，不是小数 16.1。当时分析下来，这就是挡住初始化的那个上限。补丁把 `16` 改成 `26`，阈值变成 `26.10`，我的 26.5.2 就不会被拦下来了。

光看这四条指令其实说明不了全部逻辑，还得结合后面的调用和跳转一起看。写这篇的时候我又翻了一下留着的文件：官方包里五个文件各有一处 `16.10` 这段，改过的副本各有一处 `26.10`，和当时的补丁对得上。

我的是 Apple Silicon，所以只改了 arm64。x86-64 那边指令不一样，没动，Intel Mac 我也没法测。

## 改完还得重签名

二进制一改，原来的代码签名就对不上了，必须重新签。

当时的脚本先把原始文件和 entitlements 存一份，再用 `/usr/bin/codesign` 做 ad-hoc 签名。entitlements 就是程序声明自己要哪些权限和例外。重签的时候要在原来那份基础上加东西，不能只写新加的那一项把原来的整个覆盖掉。

加的是这一项：

```text
com.apple.security.cs.disable-library-validation = true
```

原因是 ad-hoc 签名没有 Wacom 的 Team ID，可驱动还要加载 Wacom 自己签名的框架。开了 Hardened Runtime 以后，默认只能加载同一个 Team ID 或者 Apple 签名的库，所以得把库校验关掉。[Apple：Disable Library Validation](https://developer.apple.com/documentation/BundleResources/Entitlements/com.apple.security.cs.disable-library-validation?changes=_7_1)

这是有代价的，改过的这几个组件加载库的时候不再校验签名。而且说到底它还是那个旧驱动，跳过了版本检查，不代表它就真的兼容新系统了。

原文件备份在：

```text
/Library/Application Support/Tablet/macos26-cth690-patch-backup-<时间戳>/
```

脚本里还写死了五个原始文件的 SHA-256，对不上就直接退出，免得把给 6.4.10-3 准备的补丁打到别的版本上。

## 加了 sudo 还是 Operation not permitted

真正跑补丁的时候又卡了一下：

```text
PermissionError: [Errno 1] Operation not permitted
```

是在往 Wacom 应用包里写文件的时候报的。已经是 sudo 了还不行，重跑几遍也一样。

最后发现是终端的问题。我用的是 Ghostty，它在「系统设置 → 隐私与安全性 → App 管理」里没打开。之前给别的应用开过各种授权，偏偏漏了真正在跑命令的这个终端。

把 Ghostty 的 App 管理打开，彻底退出再重开，补丁就能跑下去了。App 管理管的是一个应用能不能改别的应用，跟 sudo 那套权限是两码事，root 也绕不过去。[Apple：隐私与安全性设置](https://support.apple.com/guide/mac-help/change-privacy-security-settings-on-mac-mchl211c911f/mac)

这个坑我觉得最值得记。在 macOS 上看到 `Operation not permitted`，别急着加 sudo，先看是哪个操作被拒、是哪个应用发起的。

驱动自己要的辅助功能、输入监控是另外一组权限，重签以后最好再去看一眼还有没有效。给终端临时开的 App 管理，用完记得关掉。

## 怎么算修好了

补丁最后按 DataStoreMgr、IOManager、tablet driver 的顺序重新加载服务，然后检查签名和驱动通信。

当时检查的结果：五个组件签名都通过，arm64 的版本判断改好了，CTH-690、笔和触控都能识别，之前一直没有的 `WacomTabletDriverMessageServer` 也起来了。

单独检查某个组件的签名可以这样，同样是只读的：

```bash
/usr/bin/codesign --verify --strict --verbose=2 \
  "/Applications/Wacom Tablet.localized/.Tablet/WacomTabletDriver.app"
```

签名过了不代表能用，最后还是得拿笔试。在其他应用里画了一下，压感是有的。

然后又出了个小插曲：在「预览」里给 PDF 写字，线条粗细一点不变。我差点又回头去查驱动，后来想想同一支笔别的应用里压感正常，那就不是驱动的事，是预览自己对笔输入的处理。后面去找 PDF 批注软件，那是另一个话题了。

## 如果你也卡在「等待同步」

我会按这个顺序查：先对一下板子型号、系统版本和驱动支持范围，再看 USB 有没有认到、装的是哪个版本、服务起没起来。这些都指向版本判断，才值得去动二进制。

| 现象 | 先查什么 |
|---|---|
| USB 列表里没有设备 | 线、接口、设备本身 |
| 设备能看到，驱动界面卡住 | 型号和驱动对不对得上，服务有没有初始化完 |
| 改完驱动起不来 | 签名、库校验、权限 |
| 写应用文件报 `Operation not permitted` | 被拒的路径，以及终端有没有 App 管理权限 |
| 只有某个应用没压感 | 那个应用本身支不支持数位笔，工具设置 |

我只验证了修完当时能用，没长期跑，也没测之后的系统更新会不会又坏掉。

反正 CTH-690 又能用了。回头看，重装驱动帮不上什么忙。有用的是一层层分开确认：系统认没认到设备，进程有没有在跑，驱动能不能通信，应用拿不拿得到压感。
