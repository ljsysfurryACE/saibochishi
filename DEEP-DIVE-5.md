# 🕵️ XJ380 深度挖掘补充报告 — 第五轮发现（半成品实锤）

> **分析日期**：2026-08-01
> **方法**：桌面 ELF 引用分析 + 图标/资源目录检查 + 内核 GUI 符号确认

---

## 💥 重磅 1：桌面引用了 3 个未打包的应用

**desktop.elf 引用的应用：**

```
/apps/builtin/texter.elf      ← 文本编辑器
/apps/builtin/picturer.elf    ← 图片查看器
/apps/system/fmanager.elf     ← 文件管理器
```

**实际存在的：**

```
/apps/system/fmanager.elf     ✅ 有
/apps/builtin/texter.elf      ❌ 缺失
/apps/builtin/picturer.elf    ❌ 缺失
/apps/builtin/                📁 空目录！
```

### 图标齐全，程序缺失

```
/system/icon/texter.png      ← 文本编辑器图标（有）
/system/icon/picviewer.png   ← 图片查看器图标（有）
/system/icon/calc.png        ← 计算器图标（有）
```

**桌面上能看到 texter/picturer/calc 图标，但点了什么都不会发生——程序根本没打包进发布版。**

> **"世界第四"的桌面，是"看起来能点，点了一脸懵"的演示桌面。**

---

## 💥 重磅 2：GUI 系统调用全家桶实锤（内核态绘图是设计）

内核符号表中的 GUI 系统调用：

```
do_xapi_DrawBMP    do_xapi_DrawPNG    do_xapi_DrawLine
do_xapi_DrawRect   do_xapi_DrawText   do_xapi_DrawPoint
do_xapi_Button     do_xapi_ButtonEmp
xapi_drawFABySheet xapi_drawSvgBySheet
```

**绘图、画按钮、渲染文字全部是系统调用（ring 0）。**

> 之前说"GUI 全塞内核态"——现在确认这是**刻意设计**：
> 每个像素操作都走 syscall，等于用户程序画一条线都要陷入内核。
> **"微内核"的极致：把图形栈做成系统调用，性能与安全双输。**

---

## 🟡 发现 3：内置 fastfetch（Linux 工具）9MB

```
/system/resources/apps/fastfetch   (9MB)
```

直接把 Linux 的 fastfetch（系统信息工具）搬进来充数，里面还带着一堆其他系统的 logo：

```
YiffOS / TempleOS / SteamOS / Tuxedo OS / Serpent OS / SummitOS ...
```

> 一个"自研 OS"，系统信息工具用的是 Linux 的，还顺便认识全世界的发行版。

---

## 🟡 发现 4：busybox 完整版（2411 个 applet）

标准 Debian busybox 原封不动搬进来，带完整 applet 列表（adduser/adjtimex/acpid 等 2411 个）。

---

## 🟡 发现 5：没网却配了 DNS

```
/etc/resolv.conf:
  nameserver 223.5.5.5      ← 阿里 DNS
```

**没有网络栈，却配好了 DNS 服务器** —— 就像给没插电的电脑装好网线。

---

## 🟡 发现 6：神秘图标 guoqiyu.png

```
/system/icon/guoqiyu.png   (300x300 RGBA)
```

一个无对应程序的自定义图标，名字"guoqiyu"（国旗语？人名？），与系统图标（nextstep/button/folder）风格迥异——疑似开发者私货混入。

---

## 🎯 第五轮结论

> **"世界第四"的发布版 = 半成品：**
>
> | 声称 | 实际 |
> |------|------|
> | 完整桌面 | 图标有、程序缺（texter/picturer/calc 点了没反应） |
> | 自研工具链 | 内置 Linux 的 busybox + fastfetch 凑数 |
> | 高效架构 | 画一条线都是系统调用 |
> | 网络就绪 | DNS 配好了，网络栈不存在 |
> | 系统纯净 | guoqiyu.png 私货图标混入 |
>
> **一句话：**
> "世界第四"的桌面是**只做好了图标，没做好程序**的演示台——
> **你可以在上面"看到"计算器，但永远"用不上"它。**
>
> 这可能就是"世界第四"的真实定位：**一个 PPT 级别的操作系统。**

---

*相关文档：README.md | DEEP-DIVE-2/3/4.md | REVERSE-ENGINEERING.md | LICENSE-REPORT.md | TEST-REPORT.md | README-funny.md*
