# GitHub CLI (gh) 使用帮助

`gh` 是 GitHub 官方命令行工具，可以替代大量网页操作。以下是实际使用中踩过的坑。

---

## 1. 安装

通过 winget 安装（Windows）：
```powershell
winget install GitHub.cli --accept-source-agreements --accept-package-agreements
```

安装后**需要刷新 PATH**：
```powershell
$env:Path = [Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [Environment]::GetEnvironmentVariable("Path", "User")
```

验证安装：
```powershell
gh --version
```

---

## 2. 登录认证（Web Flow）

**现象**：`gh auth login` 后卡在终端等待，或提示超时。

**原因**：`gh auth login` 默认使用 web flow，会生成一个一次性代码，需要用户在浏览器中打开 `https://github.com/login/device` 输入代码完成授权。这个过程有**时间限制**（约几分钟）。

**解决步骤**：
```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"; gh auth login --git-protocol https --web
```

终端会输出类似：
```
! First copy your one-time code: XXXX-XXXX
Open this URL to continue in your web browser: https://github.com/login/device
```

**用户需要在浏览器中**：
1. 打开 `https://github.com/login/device`
2. 输入一次性代码（如 `XXXX-XXXX`）
3. 点击授权

完成后，回到终端检查状态：
```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"; gh auth status
```

---

## 3. 代理配置：每次单独注入

**现象**：`gh` 命令访问 GitHub 超时或连接重置。

**原因**：`gh` 默认不走系统代理，需要显式注入 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量。

**最佳实践**：
```powershell
# ✅ 正确：只影响当前命令
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"; gh repo list

# ❌ 避免：不要设置系统级代理变量
```

> ⚠️ **不要**使用 `git config --global http.proxy` 来配置 gh 的代理，gh 使用的是独立的环境变量机制。

---

## 4. `gh repo view --readme` 不存在

**现象**：执行 `gh repo view owner/repo --readme` 报错 `unknown flag: --readme`。

**原因**：`gh repo view` 没有 `--readme` 参数，该命令只能查看仓库概览信息。

**解决**：

**方法 1：先 clone 仓库到本地查看**（推荐，如果后续需要操作代码）
```powershell
gh repo clone owner/repo
cd repo
cat README.md
```

**方法 2：用 curl 直接获取 raw README**（适合只想快速查看内容，不想下载整个仓库）
```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"
curl.exe -sL https://raw.githubusercontent.com/owner/repo/main/README.md
```

> ⚠️ PowerShell 中必须用 `curl.exe`，因为 `curl` 是 `Invoke-WebRequest` 的别名，不支持 `-sL` 参数。

**方法 3：使用 GitHub API**
```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"
gh api repos/owner/repo/contents/README.md --jq '.content' | base64 -d
```

---

## 5. 创建仓库并推送代码

**一次性完成创建 + 推送**：
```powershell
cd D:\your-project
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"
gh repo create repo-name --public --source=. --remote=origin --push
```

参数说明：
| 参数 | 含义 |
|------|------|
| `--public` | 公开仓库 |
| `--private` | 私有仓库 |
| `--source=.` | 使用当前目录作为源码 |
| `--remote=origin` | 远程名称为 origin |
| `--push` | 自动推送当前分支 |
| `--add-readme` | 创建时初始化 README（适用于空目录）|

---

## 6. 列出仓库

```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"; $env:HTTPS_PROXY="http://127.0.0.1:7890"
gh repo list username --limit 20 --json name,description,url,primaryLanguage
```

---

## 7. 常见错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `You are not logged into any GitHub hosts` | 未登录 | 执行 `gh auth login --web` |
| `Connect Timeout Error` | 网络问题 / 未走代理 | 注入 `HTTP_PROXY`/`HTTPS_PROXY` |
| `unknown flag: --readme` | 参数不存在 | 改用 curl 或 GitHub API |
| `Repository not found` | 权限不足或仓库不存在 | 检查登录状态或仓库可见性 |

---

## 常用命令速查

```powershell
# 登录
gh auth login --web

# 查看登录状态
gh auth status

# 创建仓库（空目录）
gh repo create repo-name --public --add-readme

# 创建仓库并推送现有代码
gh repo create repo-name --public --source=. --remote=origin --push

# 查看仓库信息
gh repo view owner/repo

# 列出仓库
gh repo list owner --limit 10

# 克隆仓库
gh repo clone owner/repo

# 创建 Issue
gh issue create --title "bug" --body "描述"

# 创建 PR
gh pr create --title "feat" --body "描述"
```

---

> 💡 **提示**：所有 `gh` 命令的详细帮助都可以通过 `gh <command> --help` 查看，例如 `gh repo create --help`。
