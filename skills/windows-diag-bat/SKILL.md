---
name: windows-diag-bat
description: 生成发给 Windows 同事运行的"单文件诊断/采集 bat"。脚本跑完会把报告路径自动复制到剪贴板、用资源管理器打开并选中文件、显示醒目横幅提示同事发回来。调用场景：用户要让别的同事跑一段诊断/采集脚本（看 Windows 策略、读注册表、读事件日志、收集某目录文件清单等），然后把结果发回来分析。
---

# Windows 同事诊断 bat 生成模式

## 场景

我（用户）需要让 Windows 同事跑一段脚本，**他跑完之后把生成的报告文件发回来给我分析**。痛点：

- 同事大概率不会装 PowerShell ISE / 不熟终端，最低要求是"双击一个文件"
- 报告路径要在最后明显，不能淹没在输出里
- 脚本要能读 Windows 内部状态（事件日志、`Get-ProcessMitigation`、CIM、注册表），所以**主体必须是 PowerShell**，不能纯 bat
- 单文件分发：bat + ps1 两个文件同事容易少传、放错目录

## 解决方案：单文件 bat 内嵌 PS

bat 末尾用 `#__PS_BEGIN__` / `#__PS_END__` 双标记包围 PS 代码。bat 头部用一行 PowerShell 把自身末尾的 PS 块抽出来写到临时 ps1，再调用执行。

### 完整骨架

```bat
@echo off
setlocal
set "PS1=%TEMP%\<task-name>-%RANDOM%.ps1"

powershell -NoProfile -Command "$t = [IO.File]::ReadAllText('%~f0'); if ($t -match '(?s)#__PS_BEGIN__\r?\n(.*)\r?\n#__PS_END__') { [IO.File]::WriteAllText('%PS1%', $matches[1], (New-Object Text.UTF8Encoding $true)) }"

powershell -NoProfile -ExecutionPolicy Bypass -File "%PS1%"
del "%PS1%" >nul 2>&1
pause
exit /b 0

#__PS_BEGIN__
$ErrorActionPreference = 'Continue'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# === 任务变量 ===
$Report = Join-Path $env:TEMP "<task-name>-$(Get-Random).txt"
$Recipient = '<我的 IM/邮箱/名字>'

# === 采集逻辑 ===
$lines = @()
$lines += '=== Section 1 ==='
$lines += '...'
# ... 等等

# === 写报告 ===
$content = ($lines -join [Environment]::NewLine)
Set-Content -Path $Report -Value $content -Encoding UTF8

# === 打印预览（可选）===
Write-Host ''
Write-Host '--- 报告内容预览 ---' -ForegroundColor DarkGray
Get-Content $Report

# === 一键便利：剪贴板 + 资源管理器 ===
try { Set-Clipboard -Value $Report } catch {}
$selectArg = '/select,' + [char]34 + $Report + [char]34
try { Start-Process explorer.exe -ArgumentList $selectArg } catch {}

# === 醒目结尾横幅 ===
Write-Host ''
Write-Host ''
Write-Host '================================================================' -ForegroundColor Yellow
Write-Host "  诊断完成！请把下面这个文件发给 $Recipient" -ForegroundColor Yellow
Write-Host ''
Write-Host "    $Report" -ForegroundColor Cyan
Write-Host ''
Write-Host '  （路径已复制到剪贴板，资源管理器会自动打开并选中该文件）' -ForegroundColor DarkGray
Write-Host '================================================================' -ForegroundColor Yellow
Write-Host ''
#__PS_END__
```

## 必须遵守的几个坑（按致命程度排序）

### 1. bat 必须是 CRLF 换行 — 最致命

`Write` 工具默认写 LF。**LF 换行的 bat 在 cmd 里会把 `exit /b 0\n#__PS_BEGIN__` 之后的 PS 代码当 cmd 命令尝试执行**，疯狂报 `'xxx' 不是内部或外部命令`。

写完 bat **必须**立刻转 CRLF：

```bash
unix2dos "/path/to/script.bat"
```

验证（前 32 字节必须有 `0d 0a`）：

```bash
xxd "/path/to/script.bat" | head -3
```

### 2. PS 写出 ps1 时用 UTF-8 with BOM

Windows PowerShell 5.1（同事电脑默认那个）按 GBK 读无 BOM 的 ps1，中文会乱码导致 parse error。所以**写临时 ps1 时必须带 BOM**：

```powershell
# 正确（带 BOM）
[IO.File]::WriteAllText($path, $content, (New-Object Text.UTF8Encoding $true))

# 错误（裸 UTF8 = 无 BOM）
[IO.File]::WriteAllText($path, $content, [Text.Encoding]::UTF8)
```

骨架里那一行 `New-Object Text.UTF8Encoding $true` 的 `$true` 就是 emitBOM 参数，**别去掉**。

### 3. PS 代码里禁用嵌套引号 + 反引号

cmd 解析 batch 时会做 lookahead 扫描。哪怕 PS 代码在 `exit /b` 之后，cmd 还是会把整个文件 lex 一遍来查 label。如果 PS 里出现：

```powershell
"/select,`"$Report`""    # ← cmd 看引号匹配会乱
```

cmd 的引号配对算法可能错误地认为某段是"开放字符串"，导致执行流出问题。

替代写法：

```powershell
$selectArg = '/select,' + [char]34 + $Report + [char]34
```

或用单引号包字面，避免 PS 反引号转义。**任何需要在字符串里塞双引号的地方都用 `[char]34`**。

### 4. bat 头部保持纯 ASCII

bat 头那段 cmd 命令不要写中文。中文要全部塞进 PS 部分（PS 部分被当成 ps1 文件由 PowerShell 重新读取，编码独立处理）。

### 5. `chcp 65001` 不要加

加了 `chcp 65001` 反而让 cmd 用 UTF-8 重新解析后续字节，可能放大问题。`[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` 在 PS 内设就够了，cmd 部分不需要管编码（因为 cmd 部分是纯 ASCII）。

## 标准工作流

1. 写 bat 文件到桌面或临时目录（用户能找到的地方）
2. **立刻 `unix2dos`** 转换换行符
3. **本地双击测试一次**，确认输出末尾横幅清晰、剪贴板能复制、资源管理器能弹
4. 验证报告文件（`%TEMP%\xxx.txt`）内容正常
5. 告诉用户文件路径，让他发给同事

测试命令（git bash 环境）：

```bash
cmd.exe //c "/path/to/script.bat"
```

注意 git bash 下 cmd 输出中文可能仍乱码（git bash 终端编码问题），但同事**双击运行时**不会乱（因为 PS 内强制设了 UTF-8 控制台编码）。乱码不要紧，看输出结构对就行。

## 采集模式参考

PS 部分可以采集：

| 类型 | API |
|---|---|
| 系统信息 | `Get-CimInstance Win32_OperatingSystem` |
| 安装的杀毒 | `Get-CimInstance -Namespace root\SecurityCenter2 -ClassName AntiVirusProduct` |
| Defender 漏洞防护 | `Get-ProcessMitigation -Name <exe>` / `-System` |
| HVCI / VBS | `Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard` |
| 事件日志 | `Get-WinEvent -FilterHashtable @{LogName='...'; StartTime=$since}` |
| 文件 hash | `Get-FileHash $path -Algorithm SHA256` |
| 注册表 | `Get-ItemProperty -Path 'HKLM:\...'` |
| 直接跑某 exe | `& $exe arg1 arg2 2>&1; "ExitCode: $LASTEXITCODE"` |

封装成 `Section` 函数让输出更整齐：

```powershell
function Section($title, $block) {
    "=== $title ==="
    try { & $block 2>&1 | Out-String -Stream } catch { "EXCEPTION: $($_.Exception.Message)" }
    ""
}

$lines = @()
$lines += Section 'OS / build' { Get-CimInstance Win32_OperatingSystem | Format-List Caption,Version,BuildNumber }
$lines += Section 'Defender exploit protection' { Get-ProcessMitigation -System | Format-List ASLR,CFG,DEP }
```

## 不适用的场景

- **不要**用这个模式让同事**修改**系统状态（写注册表、改策略、kill 进程）。诊断采集 only。要修改的话，让我先看完报告再单独给修复脚本。
- 如果同事电脑没有 PowerShell（XP / 极老 Windows），改用纯 bat + WMIC，但目标只能是非常基础的信息。
