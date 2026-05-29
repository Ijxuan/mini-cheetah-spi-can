# Codex 新建 GitHub 仓库并推送本地工程到云端

本文记录本次会话中 Codex 将本地工程 `d:\000\56747e76c9c1` 初始化为 Git 仓库、创建公开 GitHub 仓库并推送到云端的实际方法和命令。

## 目标

- 将当前本地工程纳入 Git 管理。
- 创建一个公开 GitHub 仓库。
- 将本地 `main` 分支推送到 GitHub。
- 优先使用本机已配置的认证方式，不在文档中保存任何 token、密码或私钥。

## 1. 检查本地状态

先确认当前目录还不是 Git 仓库：

```powershell
git status --short --branch
```

当时输出为：

```text
fatal: not a git repository (or any of the parent directories): .git
```

检查 GitHub CLI 是否可用：

```powershell
gh --version
gh auth status
```

结果是本机没有安装 `gh`：

```text
gh : 无法将“gh”项识别为 cmdlet、函数、脚本文件或可运行程序的名称。
```

确认工程文件：

```powershell
rg --files
```

当时工程包含：

```text
mbed-dev.lib
math_ops.h
math_ops.cpp
main.cpp
leg_message.h
```

检查 Git 是否可用，以及本机是否已经配置提交身份：

```powershell
git --version
Get-ChildItem Env: | Where-Object { $_.Name -match 'GITHUB|GH_|GIT' } | Select-Object -ExpandProperty Name
git config --global --get user.name
git config --global --get user.email
git config --global --get credential.helper
```

当时 Git 可用，且全局用户名和邮箱已经配置：

```text
git version 2.52.0.windows.1
Ijxuan
1535248756@qq.com
```

## 2. 初始化本地 Git 仓库并提交

在工程目录执行：

```powershell
git init -b main
git add .
git commit -m "Initial commit"
```

提交结果：

```text
Initialized empty Git repository in D:/000/56747e76c9c1/.git/
[main (root-commit) e09afca] Initial commit
 6 files changed, 756 insertions(+)
 create mode 100644 .hg_archival.txt
 create mode 100644 leg_message.h
 create mode 100644 main.cpp
 create mode 100644 math_ops.cpp
 create mode 100644 math_ops.h
 create mode 100644 mbed-dev.lib
```

## 3. 探测本机 GitHub 凭据

因为 `gh` 不存在，先检查 Git Credential Manager 是否已经保存了 `github.com` 的凭据。实际命令只输出是否存在用户名和密码/token，不输出凭据内容：

```powershell
@'
protocol=https
host=github.com

'@ | git credential fill | ForEach-Object {
  if ($_ -match '^username=') { 'has_username' }
  elseif ($_ -match '^password=') { 'has_password' }
}
```

输出：

```text
has_username
has_password
```

说明本机已经有可用于 GitHub API 的凭据。整个过程中没有把 token 或密码写入文件，也没有在输出中显示。

## 4. 通过 GitHub API 创建公开仓库

由于没有 `gh`，使用 PowerShell 调用 GitHub REST API 创建仓库。仓库名采用：

```text
mini-cheetah-spi-can
```

执行的 PowerShell 逻辑如下。注意：这里从 Git Credential Manager 读取凭据，只把它放入请求头使用，不打印、不保存。

```powershell
$credText = @'
protocol=https
host=github.com

'@ | git credential fill

$cred = @{}
$credText | ForEach-Object {
  if ($_ -match '^(.*?)=(.*)$') { $cred[$matches[1]] = $matches[2] }
}

if (-not $cred.password) {
  throw 'No GitHub password/token found in git credential store.'
}

$headers = @{
  Authorization = "Bearer $($cred.password)"
  Accept = 'application/vnd.github+json'
  'X-GitHub-Api-Version' = '2022-11-28'
  'User-Agent' = 'codex-git-push'
}

$user = Invoke-RestMethod -Method Get -Uri 'https://api.github.com/user' -Headers $headers
$repoName = 'mini-cheetah-spi-can'
$body = @{
  name = $repoName
  private = $false
  description = 'MIT Mini Cheetah SPI-to-CAN bridge firmware'
  auto_init = $false
} | ConvertTo-Json

try {
  $repo = Invoke-RestMethod `
    -Method Post `
    -Uri 'https://api.github.com/user/repos' `
    -Headers $headers `
    -Body $body `
    -ContentType 'application/json'
  "created=$($repo.html_url)"
} catch {
  $status = $null
  if ($_.Exception.Response) {
    $status = [int]$_.Exception.Response.StatusCode
  }
  if ($status -eq 422) {
    $repo = Invoke-RestMethod `
      -Method Get `
      -Uri "https://api.github.com/repos/$($user.login)/$repoName" `
      -Headers $headers
    "exists=$($repo.html_url)"
  } else {
    throw
  }
}

"owner=$($user.login)"
```

创建结果：

```text
created=https://github.com/Ijxuan/mini-cheetah-spi-can
owner=Ijxuan
```

最终公开仓库地址：

```text
https://github.com/Ijxuan/mini-cheetah-spi-can
```

## 5. 检查 SSH 认证

用户说明本机应该已经配置 SSH key 到 GitHub。先尝试默认 22 端口：

```powershell
ssh -T -o BatchMode=yes git@github.com
```

结果：

```text
kex_exchange_identification: Connection closed by remote host
```

这通常说明默认 SSH 端口 22 被网络环境阻断或中间设备关闭，不一定代表 SSH key 错误。

随后尝试 GitHub 支持的 443 端口 SSH：

```powershell
ssh -T -p 443 -o BatchMode=yes git@ssh.github.com
```

结果：

```text
Hi Ijxuan! You've successfully authenticated, but GitHub does not provide shell access.
```

说明 SSH key 认证成功，账号为 `Ijxuan`。

## 6. 设置远程仓库并推送

先添加普通 SSH remote：

```powershell
git remote add origin git@github.com:Ijxuan/mini-cheetah-spi-can.git
git remote -v
```

因为 22 端口不可用，随后把 remote 改成 443 端口 SSH URL：

```powershell
git remote set-url origin ssh://git@ssh.github.com:443/Ijxuan/mini-cheetah-spi-can.git
```

推送本地 `main` 分支：

```powershell
git push -u origin main
```

推送结果：

```text
branch 'main' set up to track 'origin/main'.
To ssh://ssh.github.com:443/Ijxuan/mini-cheetah-spi-can.git
 * [new branch]      main -> main
```

## 7. 最终确认

确认工作区和远程：

```powershell
git status --short --branch
git remote -v
git log --oneline --decorate -1
```

最终状态：

```text
## main...origin/main
origin  ssh://git@ssh.github.com:443/Ijxuan/mini-cheetah-spi-can.git (fetch)
origin  ssh://git@ssh.github.com:443/Ijxuan/mini-cheetah-spi-can.git (push)
e09afca (HEAD -> main, origin/main) Initial commit
```

## 结论

本次推送使用了两种认证能力：

- 创建 GitHub 仓库：通过 Git Credential Manager 中已有的 GitHub 凭据调用 GitHub REST API。
- 推送代码：通过本机已配置的 GitHub SSH key，走 `ssh.github.com:443` 通道推送。

公开仓库已经创建并推送完成：

```text
https://github.com/Ijxuan/mini-cheetah-spi-can
```
