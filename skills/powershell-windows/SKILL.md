# PowerShell Windows 使用帮助

Windows 环境下的 PowerShell 与 Linux Shell 有很大差异。以下是实际开发中踩过的坑和对应的解决方案。

---

## 1. 刚安装的程序找不到（PATH 未刷新）

**现象**：用 `winget` 安装了 Node.js / GitHub CLI 后，执行 `node -v` 提示找不到命令。

**原因**：PowerShell 不会自动继承新安装程序的环境变量，需要手动刷新当前会话的 PATH。

**解决**：
```powershell
$env:Path = [Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [Environment]::GetEnvironmentVariable("Path", "User")
```

或者更简洁地重载：
```powershell
$env:Path = "C:\Program Files\nodejs;" + $env:Path
```

---

## 2. `curl` 是 `Invoke-WebRequest` 的别名

**现象**：执行 `curl -sL https://...` 报错，提示参数不匹配。

**原因**：PowerShell 中的 `curl` 实际上是 `Invoke-WebRequest` 的别名，参数完全不同。

**解决**：使用 `curl.exe` 调用真正的 curl：
```powershell
curl.exe -sL https://raw.githubusercontent.com/.../README.md
```

或者直接用 PowerShell 原生的方式：
```powershell
Invoke-WebRequest -Uri "https://..." -OutFile "file.md"
```

---

## 3. `npm` / `npx` 有时找不到

**现象**：`npm` 命令报错 "无法将 npm 识别为 cmdlet"，但 `where.exe npm` 能找到。

**原因**：
- npm 安装后生成的是 `.ps1` 脚本（`npm.ps1`、`npx.ps1`）
- PowerShell 执行策略可能阻止运行脚本
- 或者当前会话 PATH 未包含 npm 所在目录

**解决**：
```powershell
# 确保 PATH 包含 nodejs 目录
$env:Path = "C:\Program Files\nodejs;" + $env:Path

# 如果仍有问题，直接用 node 调用 npm 内部脚本
& "C:\Program Files\nodejs\node.exe" "C:\Program Files\nodejs\node_modules\npm\bin\npx-cli.js" <package>
```

---

## 4. 环境变量只对当前会话有效

**现象**：`$env:HTTP_PROXY = "http://127.0.0.1:7890"` 后，新开 PowerShell 窗口代理失效。

**原因**：`$env:VAR = "value"` 只修改当前进程的环境变量，不会写入系统或用户级别的配置。

**解决**：如果需要持久化，使用 `[Environment]::SetEnvironmentVariable`：
```powershell
# 用户级别（推荐）
[Environment]::SetEnvironmentVariable("VAR_NAME", "value", "User")

# 系统级别（需要管理员权限）
[Environment]::SetEnvironmentVariable("VAR_NAME", "value", "Machine")
```

> ⚠️ **特别提醒**：代理环境变量建议**不要**持久化到系统，而是每次对需要代理的命令单独注入。详见下方第 6 条。

---

## 5. 命令分隔与条件执行

**现象**：用 `&&` 或 `||` 连接命令报错（旧版 PowerShell）。

**原因**：PowerShell 5.x 不支持 `&&` 和 `||`，需要使用分号 `;` 或嵌套 `if`。PowerShell 7+ 已支持 `&&` 和 `||`。

**解决**：
```powershell
# PowerShell 5.x（Windows 默认）
cmd1; cmd2; cmd3

# PowerShell 7+
cmd1 && cmd2   # cmd1 成功才执行 cmd2
cmd1 || cmd2   # cmd1 失败才执行 cmd2
```

---

## 6. 代理环境变量：每次单独注入

**现象**：某些工具（如 `npm`、`gh`）在 PowerShell 中不走系统代理设置。

**最佳实践**：**不要**设置系统级别的 HTTP_PROXY/HTTPS_PROXY，而是每次对需要代理的命令单独注入：

```powershell
# ✅ 正确做法：只影响当前命令
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"; gh auth status

# ❌ 避免：不要写入系统环境变量
# [Environment]::SetEnvironmentVariable("HTTP_PROXY", "...", "User")
```

**原因**：
- 持久化代理会影响所有程序，可能导致某些工具行为异常
- Windows 不同工具读取代理的方式不一致（环境变量、WinHTTP、IE 设置等）
- 单独注入可以精确控制哪些命令走代理

---

## 7. `where` vs `where.exe`

**现象**：`where node` 输出奇怪的结果。

**原因**：PowerShell 的 `where` 是 `Where-Object` 的别名，用于管道过滤，不是查找程序路径的 `where.exe`。

**解决**：
```powershell
# ✅ 查找程序路径
where.exe node

# ❌ 这会调用 Where-Object，结果完全不同
where node
```

---

## 8. 管道和 stderr 重定向差异

**现象**：`command 2>&1 | head` 的行为和 Bash 不同。

**原因**：PowerShell 的管道传递的是对象，不是纯文本。`2>&1` 会将 stderr 合并到 stdout，但类型仍然是 ErrorRecord 对象。

**解决**：
```powershell
# 将错误流转为字符串再过滤
command 2>&1 | Where-Object { $_ -is [System.Management.Automation.ErrorRecord] }

# 或者直接重定向到文件
command > output.txt 2> error.txt
```

---

## 快速检查清单

| 问题 | 检查命令 |
|------|----------|
| PATH 是否包含某目录 | `$env:Path -split ';' \| Where-Object { $_ -like '*nodejs*' }` |
| 当前 PowerShell 版本 | `$PSVersionTable.PSVersion` |
| 程序真实路径 | `Get-Command npm` 或 `where.exe npm` |
| 环境变量是否生效 | `$env:VAR_NAME` |

---

> 💡 **建议**：如果经常遇到 Shell 兼容性问题，可以考虑安装 [PowerShell 7](https://github.com/PowerShell/PowerShell)（跨平台、功能更完善），或直接使用 Git Bash。
