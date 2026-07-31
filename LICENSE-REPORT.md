# ⚖️ XJ380 许可证合规分析 — "世界第四"的法律风险

> **分析日期**：2026-07-31
> **方法**：从系统镜像中提取各开源组件，核对其许可证及其使用方式

---

## 1. 缝合组件许可证总表

| 组件 | 版本 | 许可证 | 传染性 | XJ380 使用方式 | 合规风险 |
|------|------|--------|--------|----------------|----------|
| **busybox** | 1.31.1 | **GPL-2.0** | 🔴 强传染 | 静态链接完整二进制（3.1MB）+ 内核引用 | **严重违规** |
| **musl libc** | — | MIT | 🟢 宽松 | 动态库 `/lib/ld-musl-x86_64.so.1` | ✅ 合规 |
| **FatFs** | — | BSD-like（ChaN 自定义） | 🟢 宽松 | 编入内核（147 处引用） | ✅ 合规 |
| **stb_image** | — | Public Domain | 🟢 无 | 编入内核（PNG/JPEG/GIF 解码） | ✅ 合规 |
| **stb_truetype** | — | Public Domain | 🟢 无 | 编入内核（TTF 渲染） | ✅ 合规 |
| **dr_mp3** | — | Public Domain | 🟢 无 | 编入内核（MP3 解码） | ✅ 合规 |
| **zlib** | — | zlib License | 🟢 宽松 | 编入内核（stbi 依赖） | ✅ 合规 |

---

## 2. 🔴 致命问题：busybox（GPL-2.0）的传染

### 铁证

从系统镜像提取的 busybox 二进制明确包含：

```
/apps/busybox:
  "BusyBox is copyrighted by many authors between 1998-2015."
  HUSH_VERSION=1.31.1
  tar (busybox) 1.31.1
  statically linked ELF (3.1MB)
```

内核符号表中还有：

```
ffffffff8011f1e0 r _ZL21busybox_alias_applets   ← 内核引用了 busybox 实现
```

### GPL-2.0 传染逻辑

```
GPL-2.0 要求:
  1. 分发时附带 GPL 许可证文本和版权声明
  2. 提供对应源码（或书面要约）
  3. 衍生作品（改编/链接 GPL 代码）必须以 GPL 发布

XJ380 的做法:
  1. ❌ 整个系统镜像无任何 LICENSE/COPYING 文件
  2. ❌ 未提供 busybox 源码
  3. ❓ 内核含 busybox_alias_applets 等符号 → 可能构成衍生
```

---

## 3. 违规点清单

| # | 违规点 | 严重度 |
|---|--------|--------|
| 1 | 分发 GPL 软件（busybox）**未附 GPL 许可证文本** | 🔴 严重 |
| 2 | 分发 GPL 软件**未提供对应源码** | 🔴 严重 |
| 3 | 内核符号引用 busybox 逻辑，**可能构成 GPL 衍生作品**（若成立，整个自研内核都须开源） | 🔴 潜在致命 |
| 4 | 自称"自研/世界第四"，但对缝合的开源组件**零开源声明** | 🟡 诚信问题 |

---

## 4. 讽刺点

> **一边自称"世界第四大操作系统"，一边把 GPL 的 busybox 缝进去还不给许可证。**
>
> 如果严格追究：
> - busybox 版权方（BusyBox 项目 / Software Freedom Conservancy）可以要求**整个 XJ380 系统开源**
> - 自研的 C++ 微内核可能因"GPL 传染"被迫公开
>
> **"世界第四" 可能要回答的法律问题是："你的代码里，第几行是别人的 GPL？"**

---

## 5. 对比：合规的做法应该是什么

```
✅ 附带 GPL-2.0 许可证文本（COPYING 文件）
✅ 提供 busybox 源码下载链接或书面要约
✅ 若内核链接/改编了 GPL 代码 → 整个内核以 GPL 开源
✅ 在系统"关于"页面列出开源组件致谢
```

---

*相关文档：README.md（架构分析）| TEST-REPORT.md（实测）| README-funny.md（搞笑版）*
