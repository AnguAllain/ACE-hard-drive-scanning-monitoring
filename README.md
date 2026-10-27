# AceSentry —— ACE 反作弊磁盘活动观测器

单文件 exe，内嵌 Sysinternals Procmon，只做只读观测：记录 ACE 在运行时访问了哪些文件、频率多高。**不修改、不拦截、不绕过 ACE。**

## 文件

| 文件 | 说明 |
|---|---|
| `dist\AceSentry.exe` | 成品，2.15 MB，自带 UAC 清单（双击即请求管理员） |
| `src\AceSentry.cs` | 源码（C#，.NET Framework 4.x） |
| `src\app.manifest` | 清单：`requireAdministrator` |
| `tools\Procmon64.exe` | 编译时内嵌的 Procmon 4.12 |

## 用法

直接双击，或命令行：

```powershell
AceSentry.exe                          # 等 ACE 出现，抓 120 秒，最多守 8 小时
AceSentry.exe /capture 300 /watch 120  # 抓 300 秒，守 2 小时
AceSentry.exe /now /capture 15         # 立即抓（自测用，不等 ACE）
AceSentry.exe /out D:\logs             # 指定输出目录
AceSentry.exe /?                       # 帮助
```

推荐流程：先启动它 → 再启动三角洲 → ACE 一加载就开始记录。

## 输出

- `ace_scan_report.txt` —— 每次会话追加一段报告
- `ace_io_samples.csv` —— 每秒的进程 I/O 计数器采样

报告包含：

- 会话时长、抓取窗口、PML 体积
- 文件操作事件总数、**峰值/平均 次每秒**
- 按操作类型分布（CreateFile / ReadFile / QueryDirectory / QueryOpen …）
- 按分钟忙闲、**最繁忙的 10 秒**
- 访问频次最高的 50 个路径、按目录汇总前 20

## 工作原理

1. 每 2 秒枚举进程，匹配 `ACE-Service64`、`ACE-Tray`、`SGuard64`、`SGuardSvc64`、`SGuardUpdate64`
2. 命中后把内嵌的 Procmon 释放到 `%TEMP%\AceSentry\`，以 `/BackingFile` 开始内核级文件 I/O 捕获
3. 抓取窗口到时（或 PML 超过 3 GB、或 ACE 退出）→ 优雅 `/Terminate` → `/OpenLog` 导出 CSV
4. 流式解析 CSV，只保留 ACE 进程名的行，聚合统计
5. 删除 PML 与 CSV，仅保留报告和采样

PML 约 30 MB/秒，120 秒的窗口约 3–4 GB，分析完立即删除。

## 已知限制

- **只覆盖启动后的一段时间**。默认 120 秒，覆盖 ACE 初始化时的扫描高峰；长时间游玩中的周期性扫描需要把 `/capture` 调大
- 必须管理员运行（内核 I/O 事件需要 Procmon 驱动）
- 抓取的是**文件系统事件**，不是"读了多少扇区"；目录枚举（`QueryDirectory`）在 ACE 扫盘里占比很高，报告里单独列出
- 运行 Procmon 期间系统会有轻微 I/O 开销（约 5–10%）

## 重新编译

```powershell
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /nologo /target:exe /platform:x64 `
  /optimize+ /codepage:65001 /out:dist\AceSentry.exe /win32manifest:src\app.manifest `
  /resource:tools\Procmon64.exe,Procmon64.exe `
  /r:System.dll /r:System.Core.dll /r:System.Management.dll src\AceSentry.cs
```

## 实测（模拟负载）

用改名成 `ACE-Service64.exe` 的 `findstr` 扫描 `C:\Windows` 做验证：

| 抓取 | 事件数 | 峰值 | 平均 |
|---|---|---|---|
| 13 秒 | 94,636 | 14,739 次/秒 | 5,915 次/秒 |
| 21 秒 | 121,126 | 15,062 次/秒 | 5,047 次/秒 |

---

Procmon 版权归 Microsoft Sysinternals，随附 `tools\Eula.txt`。
