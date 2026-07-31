# 🕵️ XJ380 深度挖掘补充报告 — 第三轮发现（开发环境 / 字体 / 协议）

> **分析日期**：2026-07-31
> **方法**：ELF 字符串提取 + 字体元数据分析 + 源码路径还原

---

## 💥 重磅 1：整个系统在 WSL 里开发（终极打脸）

**所有应用 ELF 都嵌着 Windows WSL 的源码路径：**

```
/mnt/c/XJ380/XJ380          ← WSL 挂载的 Windows C 盘路径
```

### 完整工具链画像

| 组件 | 工具链 | 平台 |
|------|--------|------|
| 内核 (kernel.krl) | Ubuntu GCC 13.3.0 + clang 18.1.3 混编 | WSL / Ubuntu 24.04 |
| EFI 引导器 (BOOTX64.efi) | **GCC 13-win32** | **Windows 原生工具链** |
| 应用 (.elf) | GCC 13.3.0 + clang 18.1.3 | WSL |
| busybox | glibc + Debian 包构建 (`./DEBIAN`) | Debian/Ubuntu |
| 图片素材 | Adobe Photoshop 24.0 (Windows) | Windows |
| 源码路径 | `/mnt/c/XJ380/XJ380` | **WSL** |

> **"世界第四大操作系统"的诞生地：微软 WSL 里的 Ubuntu 容器。**
> 自研 OS 的源码存在 Windows 的 C 盘里——这可能是"世界第四"里"Windows 血统"的真相。

---

## 💥 重磅 2：字体是 Adobe 思源黑体改名

**XJ380F.ttf（10.8MB）实锤为 Adobe 思源黑体（Source Han Sans CN）：**

```
Font Name: Source Han Sans CN
Version: 1.004
Copyright 2014, 2015 Adobe Systems Incorporated
Reserved Font Name 'Source'
SIL Open Font License, Version 1.1
http://scripts.sil.org/OFL
```

- 直接**重命名成 XJ380F.ttf** 冒充自研字体
- 协议是 **SIL OFL 1.1**（又一条许可证要求：保留版权声明）
- 系统里同样没有 OFL 许可证文本

> 字体抄 Adobe 的、图标用 Photoshop 画的、错误屏抄 Windows 的——
> **"世界第四"的原创性，主要体现在改文件名上。**

---

## 💥 重磅 3：内核源码文件完整暴露（106 个 C++ 文件）

内核未 strip，完整源码路径一览无余：

```
kernel/main.cpp, scheduler.cpp, pcb.cpp, ipc.cpp, mutex.cpp
kernel/memory/{buddy,frame,heap,page,vma,lazyalloc}.cpp
kernel/syscall/{syscall,sys,proc,message,signal}.cpp
kernel/syscall/xapi/{xgui,xfile,xinstaller,xtui}.cpp   ← GUI 系统调用!
kernel/wsod/{wsod,backtrace}.cpp                        ← 仿 Windows 蓝屏
kernel/user/{user,runfile,ulog,x3tp}.cpp
driver/ahci/{ahci,ata,atapi,utils}.cpp
driver/fs/fatfs/{ff,diskio,ffsystem,ffunicode}.cpp       ← ChaN FatFs 移植
driver/fs/vfs/{vfs,dev,procfs,pipefs,pty,socketfs,tmpfs,unixsock,dnsfs,nmfs}.cpp
driver/{pci,ide,nvme,hda,pcspk,vsound,sb16,ps2,rtc,serial,dma,fbdev,power}.cpp
```

**关键实锤：**
- `kernel/syscall/xapi/xgui.cpp` → **GUI 是系统调用级设计**，不是架构失误
- `kernel/wsod/wsod.cpp` → **仿 Windows 蓝屏**，函数名 `wsod_unexpect_intrrupt` 还把 interrupt 拼错了

---

## 🟡 发现 4：x3tp 是自研"三体协议"？

```
kernel/user/x3tp.cpp / x3tp.h
```

X3TP（猜测：XJ380 Three-Terminal Protocol？）——自研的用户态协议，但内核里**没有任何网络传输实现**，所以 x3tp 大概率只是本地 IPC 的封装名。

---

## 🟡 发现 5：音频测试用主板蜂鸣器

```
PC Speaker: Test completed
hda test: start 440Hz square wave
```

"世界第四"的音频测试：**主板蜂鸣器 + 440Hz 方波**。连 44100Hz 采样率都没到。

---

## 🎯 第三轮结论

> **XJ380 的"原创性"盘点：**
>
> | 声称 | 实际 |
> |------|------|
> | 自研字体 | **Adobe 思源黑体改名**（OFL 协议违规） |
> | 自研系统 | **WSL 里写**，源码在 Windows C 盘 |
> | 自研引导器 | **GCC 13-win32 编译**（Windows 工具链） |
> | 独特错误屏 | **模仿 Windows 蓝屏**（单词还拼错） |
> | 独特 GUI | GUI 系统调用（xgui.cpp）——确实"独特" |
> | 原创素材 | Photoshop 24.0 画的 |
>
> **结论一句话：**
> "世界第四"的原创含量 ≈ **内核骨架**（自研）+ 全世界的开源零件 + Adobe 的字体 + Windows 的蓝屏 + 微软 WSL 的开发环境。
>
> **它可能是世界上第一个"用 Windows 开发、模仿 Windows、却宣称替代 Windows"的操作系统。**

---

*相关文档：README.md | TEST-REPORT.md | LICENSE-REPORT.md | DEEP-DIVE-2.md | README-funny.md*
