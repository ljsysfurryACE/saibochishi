# 🧵 saibochishi — XJ380 "世界第四" 操作系统技术分析报告

> **标题灵感**：XJ380 开发方自称"世界第四大操作系统"。本仓库收录对该系统的完整逆向技术分析。
>
> **分析对象**：XJ380 Singularity 1.0.0（星纪软件 / xingjisoft.com）
> **样本来源**：3GB VMware 虚拟磁盘镜像（GPT 单分区 EFI，实际占用 87MB）
> **分析方式**：静态逆向（符号表 / 反汇编 / 字符串特征提取）

---

## 1. 系统总览

| 项目 | 值 |
|------|-----|
| 系统名称 | XJ380 Singularity 1.0.0 |
| 厂商 | 星纪软件（xingjisoft.com） |
| 磁盘 | 3GiB 虚拟盘（稀疏，实际 87.4 MiB） |
| 分区 | GPT 单分区（EFI System，3G） |
| 内核 | `system/kernel.krl` — 5.2MB 静态 ELF64 |
| 引导 | `EFI/BOOT/BOOTX64.efi` |
| 符号 | 2529 个（未 strip，带 debug_info） |
| 语言 | C++（1968 个 mangled 符号 `_Z*`） |

### 内建应用（apps/）
```
desktop.elf   桌面环境
dock.elf      任务栏
login.elf     登录界面
fmanager.elf  文件管理器
filedlg.elf   文件对话框
elfrun.elf    ELF 运行器
dyn-hello     动态链接测试
sigtest.elf   信号测试
```

---

## 2. 架构定性：不是微内核，是"巨型缝合单体"

核心结论：**自研 C++ 微内核骨架 + Linux syscall 兼容层 + 开源零件全塞内核态**。

```
┌─────────────────────────────────────────────────────┐
│                   内核态 (ring 0)                    │
│                                                     │
│  ┌─────────┐ ┌─────────┐ ┌──────────────────────┐  │
│  │ 自研     │ │ Linux   │ │  GUI 全栈 (XWM)      │  │
│  │ 微内核   │ │ syscall │ │  desktop/dock/login  │  │
│  │ 调度/内存│ │ 兼容层  │ │  draw_mouse/draw_pt  │  │
│  │ 进程/IPC │ │         │ │  207 个窗口绘图符号  │  │
│  └─────────┘ └─────────┘ └──────────────────────┘  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │ 缝合的开源零件                                  │  │
│  │  FatFs / musl / busybox / stb / dr_mp3 / zlib │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │ 调试器全开: [DEBUG-grep-syscall] 等            │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 2.1 内核本体（自研部分，确实有东西）

- **任务/调度**：43 个符号，`scheduler_init_task`、`smp_scheduler_lock`、`scheduler_is_ready` 等
- **内存管理**：184 个符号，buddy 分配器（`buddy_init`）+ heap（`heap_extend/heap_onerror`）+ 页表管理
- **进程**：`sys_fork/sys_clone/sys_vfork/sys_clone3/sys_execve` 全套
- **IPC**：`do_message`、`xbps`（自研进程通信，调试日志可见 `[DEBUG-xbps-wait]`）
- **文件系统**：284 个符号，devfs / procfs / pipefs / socketfs / FatFs
- **同步**：134 个锁/信号量相关符号

### 2.2 Linux 兼容层（长得像，但不是）

- 完整 Linux syscall 命名：`sys_openat / sys_clone3 / sys_execve / sys_mremap / sys_capget / sys_capset / sys_fsopen...`
- 但 `nm` 中 **0 个 Linux 内核典型符号**（无 start_kernel / init_task / do_syscall_64）
- banner 字符串：`Linux version 6.6.30 (xj380@localhost) (x86_64-xj380-gcc) #1 SMP PREEMPT_DYNAMIC`
  → **仅抄版本号，内核不是 Linux**（无任何 Linux 内核实现符号）

---

## 3. 开源缝合成分表

| 组件 | 证据 | 用途 |
|------|------|------|
| **FatFs** | 147 处引用，`fatfs_init/mkdir/mkfile`，`fatfs_time_to_ns` | FAT32 文件系统（ChaN 开源库） |
| **musl** | `/lib/ld-musl-x86_64.so.1`、`/lib/libc.musl-x86_64.so.1` | 用户态 C 库 |
| **busybox** | 23 处，`busybox_alias_applets`、`/apps/busybox`、`[busybox-debug]` 系列 | 命令行工具集 |
| **stb_image** | `stbi__jpeg/png/gif/bmp/tga` 全系列符号 | 图片解码（stb 单文件库） |
| **stb_truetype** | `stbtt_InitFont_internal`、`stbtt_FindMatchingFont_internal` | TTF 字体渲染 |
| **dr_mp3** | `drmp3_L3_decode`、`drmp3_L3_read_side_info`、源码名 `dr_mp3.h` | MP3 解码（David Reid 单文件库） |
| **zlib** | 21 处，`stbi__parse_zlib` | 压缩（stbi 内部依赖） |
| **XULS/XWM** | `desktop: init_xuls`、`XWM Initializing Success` | 自研窗口系统 |
| **xapi_json** | `xapi_json_impl.inc`、`XAPI_JSON_VALUE_*` | 自研 JSON 解析 |
| **NeXTSTEP 皮肤** | `/system/icon/nextstep.png` | GUI 风格借鉴 |

---

## 4. 🔴 严重问题清单

### 4.1 GUI 全栈在内核态（最致命）
```
207 个窗口/绘图符号:
  WINDOWLS / SHEET_INFO / SHEET_BUFFER 结构
  draw_mouse / draw_point / desktop_flusher / flush_task_dock
  create_window_dock / focus_window_dock / register_task_dock
```
→ 任何用户态图形 bug 都可能导致整个内核崩溃。

### 4.2 零 KASLR
- `grep kaslr` → **0 个符号**
- 内核固定加载在 `0xffffffff80000000`
- 攻击者无需绕过地址随机化即可进行内核提权

### 4.3 权限隔离 ≈ 0
- 安全相关符号仅 **23 个**
- 用户态与内核态边界模糊，无能力模型/权限检查痕迹

### 4.4 调试日志全开（生产级风险）
```
[DEBUG-grep-syscall] enter task=%s nr=%llu(%s) args=...
[DEBUG-xbps-syscall] task=%s nr=%llu(%s) ...
[busybox-debug] access path=%s ret=%lld
[DEBUG-xbps-wait] waiting parent=%s ...
```
→ 每个 syscall 的参数/返回值都打日志，性能与安全双重问题。

### 4.5 网络栈名不副实
- 有 socket API（bind/connect/listen/accept/getsockopt...）
- 但几乎无 TCP/IP 协议处理符号（无 tcp_/udp_/arp_ 实现）
- 网络能力大概率仅限简单/本地通信

### 4.6 版本字符串抄袭嫌疑
- `Linux version 6.6.30` banner 但非 Linux 实现
- 部分代码由 `Ubuntu clang 18.1.3` 编译混入（gcc+clang 混编）

---

## 5. 综合评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 自研程度 | ⭐⭐⭐⭐ | 调度/内存/进程/窗口确实自研 |
| 架构设计 | ⭐⭐ | 微内核之名，单体缝合之实 |
| 安全性 | ⭐ | 无 KASLR / 无隔离 / 调试全开 |
| 工程规范 | ⭐⭐ | 调试日志未清理，混编工具链 |
| 完成度 | ⭐⭐⭐ | 能启动到图形界面，有完整应用集 |

### 一句话结论

> **能独立写出调度器、内存管理、syscall 兼容层和窗口系统，水平超过 99% 的开发者；但把 GUI、busybox、调试器全部塞进内核态、放弃 KASLR 与权限隔离，自称"世界第四大操作系统"——是把"能跑"当成了"能打"。**
>
> 作为学习/演示项目：优秀。
> 作为生产 OS：距离"世界第四"还差一个"安全重写"。

---

## 6. 附录：关键命令备忘

```bash
# 分析工具链
file kernel.krl
nm kernel.krl | wc -l
nm kernel.krl | grep "_Z" | wc -l        # C++ 符号数
nm kernel.krl | grep -i kaslr | wc -l     # KASLR 检查
strings kernel.krl | grep -iE "FatFs|stbi|drmp3|busybox|musl"   # 组件特征

# 磁盘分析
qemu-img info xj380.vmdk
qemu-img convert -O raw xj380.vmdk disk.img
losetup /dev/loop0 disk.img && partprobe /dev/loop0
mount /dev/loop0p1 /mnt
```

---

*报告生成日期：2026-07-31*
*分析方法：静态逆向（未执行样本）*

---

## 📎 相关文档

- **[实测报告](./TEST-REPORT.md)** — QEMU 实测："世界第四"真的能启动，还进了桌面
- **[搞笑版](./README-funny.md)** — 《世界第四？先问问你家承重墙》
- 截图：[OOBE 首次配置向导](./screenshot-oobe.png) | [桌面](./screenshot-desktop.png)

- **[许可证合规分析](./LICENSE-REPORT.md)** — GPL 传染：busybox 的雷，可能让"世界第四"被迫开源

- **[深度挖掘补充报告](./DEEP-DIVE-2.md)** — 没有网络栈 + Photoshop 元数据 + 编译残留

- **[深度挖掘补充报告（第三轮）](./DEEP-DIVE-3.md)** — WSL 开发 + 思源黑体改名 + 源码全暴露

- **[深度挖掘补充报告（第四轮）](./DEEP-DIVE-4.md)** — 安全防护归零 + 调试信息全保留 + TEMP测试函数

- **[反汇编深度分析](./REVERSE-ENGINEERING.md)** — GUI syscall 绕过用户内存检查 → 任意内核读写提权漏洞

- **[深度挖掘补充报告（第五轮）](./DEEP-DIVE-5.md)** — 半成品实锤：图标有程序缺 + GUI 系统调用全家桶

- **[OpenXJ380 整改对比](./OPENXJ380-FOLLOWUP.md)** — 打脸→整改闭环：漏洞已修 + 许可证已补 + 网络/浏览器已实现
