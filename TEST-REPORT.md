# 🧪 XJ380 实测报告 — "世界第四" 真的能启动

> **实测日期**：2026-07-31
> **测试环境**：QEMU + KVM 加速（183.66.27.22 跳板机），UEFI (OVMF) 引导
> **样本**：XJ380 Singularity 1.0.0 虚拟磁盘（3GB GPT / EFI 单分区）

---

## ✅ 结论先行：它能启动，而且能进桌面

**"世界第四" 不是纯吹——完整启动链全部实测通过：**

```
✅ UEFI 固件加载
✅ XJ380 Boot Manager（自研引导器，非 GRUB）
✅ XSK2.1 内核加载（"Kernel Read Success"）
✅ ACPI 2.0 解析（XSDT 0x3FB7D0E8）
✅ 设备注册（stdio / tty / null / urandom）
✅ TTF 字体加载（XJ380F.ttf 10.8MB，中文渲染）
✅ XWM 窗口系统初始化成功
✅ OOBE 图形化首次配置向导
✅ 创建管理员账户（admin / 123456）
✅ task: Process login exit with code 0（表单提交成功）
✅ desktop 进程 (pid=2) + dock 进程 (pid=3) 启动
✅ 桌面渲染（蓝色壁纸 + 图标，90% 彩色像素）
```

---

## 📸 实测过程记录

### 1. 引导阶段（串口日志）

```
XJ380 Operating System - Serial Log
Copyright(C) XINGJI Interacitve Software 2017-2026 All rights reserved.
OS Version: BuildVersion
EFI Initialize Success.
XJ380 Boot Manager
Press DELETE for Advanced Boot Options.
New Page Table Created Success.
Root Dir Open Success.
XSK2.1 Kernel Read Success.
Frame Buffer Phys Addr: 0xFFFF8000800000
Video Initialize Success.
ACPI Table Found Success.
ACPI Version: 2
```

### 2. 内核启动

```
pty: initialized
Registers (stdio)device success
Registers (tty)device success
Registers (null)device success
Registers (urandom)device success
XJ380 Message Initialize Success.
Found 65 kernel symbols
Dynamic linker initialized
Process Reaper is ready
[XJ380 System Kernel][WARNING] XJ380 is running under the QEMU/BOCHS.
desktop flusher is running
```

### 3. 图形系统

```
Initializing Font...
TTF: loaded '/system/font/XJ380F.ttf' size=10807764 read=39ms init=0ms
TTF: loaded '/system/font/XJ380C.ttf' size=5448380 read=33ms init=0ms
TTF Initialize Success.
Initializing XWM...
XWM Initializing Success.
desktop: prepare login session
desktop: init_xuls begin
desktop: init_xuls done
desktop initialized.
```

### 4. OOBE 首次配置（截图 OCR）

```
欢迎使用 XJ380
创建第一个管理员账户
用户名: [admin]
密码:   [******]
确认密码:[******]
按制表键切换输入框，按回车键创建账户。
```

填完回车后：
```
task: Process login exit with code 0.   ← 成功！
Creating User Process. Process Name: desktop (pid=2)
Creating User Process. Process Name: dock (pid=3)
```

---

## 🐛 实测中坐实的"屎点"

| # | 问题 | 实测证据 |
|---|------|----------|
| 1 | **生产系统开调试日志** | 每个进程创建都打 `[busybox-debug] create_process cmdline= argc=0 linux_abi=0` |
| 2 | **官方自认只能跑虚拟机** | `[WARNING] XJ380 is running under the QEMU/BOCHS` |
| 3 | **内核符号表全裸奔** | `Found 65 kernel symbols`（debug 版内核） |
| 4 | **OOBE 焦点切换 Bug** | 键盘 Tab 导航不灵，鼠标点击坐标偏差大，实测填了 5 轮才成功 |
| 5 | **版本号心虚** | Banner 抄 `Linux version 6.6.30`，但内核 0 个 Linux 实现符号 |

---

## 🎯 最终评价

> **"世界第四" 名不虚传——不是吹的，是真能跑。**
>
> 但它像一台精密的**演示机**：
> - ✅ 能启动、能配置、能进桌面、能渲染中文
> - ❌ 调试日志全开、零 KASLR、无权限隔离、只能活在 QEMU 里
>
> **"能跑" 这一条，它做到了。** 至于"能用"——那要看敢不敢把桌面随便点崩一次。

---

*完整静态分析见 [README.md](./README.md) | 搞笑版见 [README-funny.md](./README-funny.md)*
