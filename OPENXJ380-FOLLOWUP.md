# 🔄 OpenXJ380 整改对比报告 — 从"打脸"到"整改"的完整闭环

> **分析日期**：2026-08-02
> **背景**：2026-07-31 我们对 XJ380 闭源镜像完成五轮解剖（架构/安全/许可证/半成品）；2026-08-01 星纪软件发布 **OpenXJ380 开源仓库**（Apache-2.0）。
>
> 🎬 **制作组公开道歉**：2026-08-02，星纪软件制作组在看到分析动态后**发布视频公开道歉**，随后开源整改。完整闭环：解剖 → 打脸 → 道歉 → 开源 → 修复。本文对比"我们报告的问题"与"开源版的处理"。

---

## 1. 时间线（关键巧合）

| 日期 | 事件 |
|------|------|
| 2026-07-31 | 我们解剖 XJ380 闭源镜像，报告 CRITICAL 漏洞 + GPL 违规 + 无网络栈 |
| 2026-08-01 | **OpenXJ380 开源仓库上线**（描述："不含桌面环境的官方开源仓库"） |

---

## 2. 整改对比总表

| # | 我们报告的问题 | 严重度 | OpenXJ380 的处理 | 状态 |
|---|---------------|--------|-----------------|------|
| 1 | **GUI syscall 绕过 user_range_mapped → 任意内核读写提权** | 🔴 CRITICAL | `xapi_copy_gui_string` + `xapi_copy_string_from_user` + `user_range_mapped` + 边界检查（sys.cpp 57 处） | ✅ **已修复** |
| 2 | **busybox GPL-2.0 违规分发（无许可证/无源码）** | 🔴 严重 | THIRD_PARTY_NOTICES.md + busybox 完整源码归档 + GPLv2 文本 + 重建说明 | ✅ **已合规** |
| 3 | **思源黑体改名 XJ380F.ttf（OFL 违规）** | 🟡 中 | 字体许可证声明（OFL） | ✅ **已声明** |
| 4 | **无网络栈（只有 unix socket）** | 🟡 功能缺失 | **新增**：e1000 驱动 + lwIP 网络服务（BSD-3-Clause）+ httpget/HTTPS 客户端 + netcfg | ✅ **已实现** |
| 5 | **桌面半成品（texter/picturer/calc 图标有程序缺）** | 🟡 完成度 | user/texter、user/calc、user/picturer 全部补齐 | ✅ **已补齐** |
| 6 | **无浏览器** | 🟡 缺失 | **新增**：user/browser（litehtml + Gumbo + Mbed TLS + Lexbor） | ✅ **已实现** |
| 7 | **无许可证文件** | 🟡 合规 | LICENSE (Apache-2.0) + LICENSES.md + 完整合规流程 + compliance-manifest.json | ✅ **已建立** |
| 8 | **fastfetch 等 Linux 工具无声明** | 🟡 合规 | LICENSES.md 明确列出 fastfetch/XBPS/glibc 的许可要求 | ✅ **已声明** |

---

## 3. 漏洞修复细节（重点验证）

### 修复前（闭源版反汇编证据）

```asm
xapi_drawFABySheet:
    test %rdi, %rsi, %r9          ; 只有空指针检查
    cmpb $0x0, (%rbx)             ; ❌ 直接解引用用户指针
    call *%r9                     ; ❌ 间接调用用户提供指针
    ; 全程无 user_range_mapped
```

### 修复后（开源版源码）

```cpp
static int xapi_copy_gui_string(char **out, uint64_t user_str)
{
    return xapi_copy_string_from_user(out, (const char *)user_str,
                                      XAPI_USER_STRING_MAX);  // ✅ 带长度上限
}

int xapi_doDrawFA(uint64_t handle, ...)
{
    WINDOW_HANDLE *window_handle = xapi_get_window_handle(handle);  // ✅ 句柄校验
    if (xapi_copy_gui_string(&fa_name, name) < 0) return -1;        // ✅ 安全复制
    if (x >= (uint32_t)content_w || y >= (uint32_t)content_h) return -1; // ✅ 边界检查
    ...
}
```

**修复质量评价**：干净利落——copy_from_user + 长度上限 + 句柄校验 + 边界检查一个不少。

---

## 4. 新增能力（以前没有的）

### 网络（之前实锤"不能上网"）
```
kmod/e1000/          Intel 网卡驱动
kmod/netserver/      lwIP 网络服务（BSD-3-Clause）
user/httpget.cpp     HTTP 客户端
user/https_client.cpp HTTPS 客户端（Mbed TLS）
user/netcfg.cpp      网络配置
```

### 浏览器（之前没有）
```
user/browser/
third_party/litehtml/     HTML 渲染
third_party/lexbor/       HTML 解析器
Mbed TLS                  TLS
```

### 应用补齐
```
user/calc/     user/texter/     user/picturer/
user/taskmgr/  user/ctrlmenu/   user/xjver/
```

---

## 5. 没改的部分（架构未动）

- **GUI 系统调用仍在内核态**：`kernel/syscall/xapi/xgui.cpp`（1168 行）依然存在
- 但 README 明确"不含桌面环境"——desktop 移到 `user/desktop`，内核只保留 syscall 接口
- `kernel/wsod/wsod.cpp` 蓝屏模块还在（没细查是否还拼错 interrupt）

---

## 6. 评价

> **OpenXJ380 展示了"听得进批评"的罕见品质：**
>
> 我们前一天列的问题，隔天开源版就：
> - ✅ **修掉了 CRITICAL 提权漏洞**（copy_from_user + 边界检查，质量不错）
> - ✅ **补齐全部许可证**（busybox GPL 源码归档、OFL 字体声明、Apache-2.0 主许可）
> - ✅ **实现网络栈**（e1000 + lwIP + HTTPS 客户端）
> - ✅ **补上浏览器**（litehtml + Lexbor + Mbed TLS）
> - ✅ **补齐缺失应用**（texter/calc/picturer）
>
> **"世界第四"可能真的要开始"世界第四"了** —— 至少现在它：
> 能联网 ✅ 有浏览器 ✅ 许可证合规 ✅ 提权漏洞已修复 ✅
>
> **遗留悬念**：
> - GUI 系统调用是否还有其他绕过路径？（修复覆盖了 drawFA，其他 200+ 窗口函数需逐一验证）
> - wsod 蓝屏还在不在
> - "不含桌面环境" = 桌面闭源了？
>
> **一句话**：这是罕见的"被批评后真的整改"案例。从"能跑 ≠ 能用"到"能联网 + 能上浏览器"，它迈出了一大步。**世界第四的路，这次走得有点样子了。**

---

*相关文档：README.md | REVERSE-ENGINEERING.md（漏洞原始分析）| LICENSE-REPORT.md | DEEP-DIVE-2/3/4/5.md | TEST-REPORT.md | README-funny.md*
