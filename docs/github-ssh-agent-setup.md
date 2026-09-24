# 从零搭建 GitHub CLI + ssh-agent 免密推送链路

> 本文记录在一台「没有 gh CLI、git 无已存凭据」的 Windows 机器上，搭建 gh + ssh-agent 免密推送 GitHub 的完整过程，以及过程中遇到的网络与配置问题。
>
> 对应 `README.md` 第 6 节记录的历史问题：*「本机没有 `gh` CLI、git 无已存凭据，无法直接创建远程仓库」* —— 本文是它的正式解决方案。

## 一、为什么需要这套链路

向 GitHub 推送代码时，认证与传输是两件事：

| 通道 | 用途 | 本项目采用 |
|---|---|---|
| **SSH** | `git clone / fetch / push` 的实际传输 | ✅ ssh-agent 托管私钥，免密 |
| **HTTPS API** | gh 命令、仓库管理、公钥登记 | ✅ gh CLI + PAT |

把两条通道分开，排错时就能快速定位问题出在「密钥没送到」还是「GitHub 不认这把钥匙」。

## 二、环境探测

先确认起点，再决定路线：

```bash
gh --version; git --version; ssh -V; ssh-add -l
```

关键结论：

- `gh` 未安装；`git 2.55.0`、`OpenSSH 10.3p1` 已就绪；
- `~/.ssh` 目录不存在，即本机从未生成过 SSH 密钥；
- `ssh-agent` 服务未运行。

同时探测网络通道：

```bash
t() { echo "$2 -> $(curl -sS -o /dev/null -w '%{http_code}' --max-time 12 "$1")"; }
t https://github.com              "github.com (HTTPS)"
t https://api.github.com          "api.github.com"
t https://codeload.github.com     "codeload (快照下载)"
t https://github.com/login/device/code "device flow"
ssh -T git@github.com             "SSH 22 端口"
```

本机结果：`github.com` 返回 **502**（被网络策略拦截），但 `api.github.com` 返回 **200**、`codeload.github.com` 可达、SSH 22 端口连接正常且返回 `Permission denied (publickey)`（说明链路通、只是未登记密钥）。

> **判读要点**：SSH 返回 `Permission denied (publickey)` 是「好消息」，代表网络与协议都正常，只是缺公钥登记；只有超时或连接被拒才是网络问题。

## 三、安装 gh CLI

### 3.1 winget 优先（成功则跳过 3.2）

```bash
winget install --id GitHub.cli -e --accept-source-agreements --accept-package-agreements --silent
```

本机失败，报错：

```
执行此命令时发生意外错误：
InternetOpenUrl() failed.
0x80072efd : unknown error
```

`0x80072efd` 是连接失败的典型错误码（winget 的下载通道被网络策略拦截）。**不要反复重试**，直接走镜像便携安装。

### 3.2 镜像 + 便携安装

```bash
GH_VER=2.101.0
DEST="$HOME/.workbuddy/binaries/gh"; mkdir -p "$DEST"; cd "$DEST"

for M in "https://ghfast.top/https://github.com/cli/cli/releases/download/v${GH_VER}/gh_${GH_VER}_windows_amd64.zip" \
         "https://gh-proxy.com/https://github.com/cli/cli/releases/download/v${GH_VER}/gh_${GH_VER}_windows_amd64.zip" \
         "https://ghproxy.net/https://github.com/cli/cli/releases/download/v${GH_VER}/gh_${GH_VER}_windows_amd64.zip"; do
  curl -L --max-time 300 -o gh.zip "$M" && [ "$(stat -c%s gh.zip)" -gt 1000000 ] && break
done

python -c "import zipfile; zipfile.ZipFile('gh.zip').extractall('.')"
mkdir -p ~/bin && cp bin/gh.exe ~/bin/
gh --version    # gh version 2.101.0
```

三个细节值得注意：

1. **判断成功用文件体积而非 HTTP 状态码** —— 镜像失败时常返回 `200` 加一个 HTML 错误页；
2. ZIP 内是 `bin/gh.exe`，复制到 `~/bin`（已在用户 PATH）比修改 PATH 更省事；
3. 版本号可查 `https://api.github.com/repos/cli/cli/releases/latest` 获取最新值。

## 四、SSH 密钥与 ssh-agent

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "workbuddy-gh" -q

export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
ssh-add -l >/dev/null 2>&1 || eval "$(ssh-agent -a "$SSH_AUTH_SOCK" -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l
```

### 关键：为什么要 `-a` 指定固定 socket

`eval "$(ssh-agent -s)"` 会生成随机 socket 路径（形如 `/tmp/ssh-XXXX/agent.PID`）。由于**每条命令都是独立的 shell**，下一条命令就找不到 agent 了，表现为：

```
Could not open a connection to your authentication agent.
```

改用 `ssh-agent -a "$HOME/.ssh/agent.sock"` 固定路径后，任何后续命令只要先 `export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"` 就能复用同一个 agent。

**这是整套流程最容易漏的一步**：每次执行 SSH 相关命令前都必须先 export。

### ssh config 与主机指纹

```bash
cat > ~/.ssh/config <<'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config

ssh-keyscan -t rsa,ecdsa,ed25519 github.com >> ~/.ssh/known_hosts 2>/dev/null
```

`IdentitiesOnly yes` 可避免 agent 中存在多把密钥时提交错误的密钥。

## 五、公钥登记到 GitHub

### 方式 A：网页手动添加

把 `cat ~/.ssh/id_ed25519.pub` 的内容粘贴到 `https://github.com/settings/keys` 的 **New SSH key**。

> 注意区分两个页面：
> - **账号级** `github.com/settings/keys` —— 属于账号，可推送该账号下所有仓库；
> - **仓库级** 仓库 Settings → **Deploy keys** —— 只对该仓库生效，**不会出现在账号的 SSH keys 列表里**（`GET /users/{user}/keys` 查不到）。

### 方式 B：gh 自动上传（需要 PAT）

`gh auth login` 的网页设备码流程依赖 `github.com/login/device/code`，该端点在受限网络下常不可达，因此直接注入 PAT：

```bash
export GH_TOKEN="<classic PAT：repo + admin:public_key>"
echo "$GH_TOKEN" | gh auth login --with-token
gh ssh-key add ~/.ssh/id_ed25519.pub --title "workbuddy-gh"
```

### 验证（必做）

```bash
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
ssh -T -o BatchMode=yes -o ConnectTimeout=15 git@github.com
```

- 成功：`Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.`
  —— 注意可以**从这行输出直接提取 GitHub 用户名**，无需额外询问；
- 失败：`Permission denied (publickey)`。

### 区分「agent 没送到」与「GitHub 不认钥匙」

```bash
ssh -vT -o BatchMode=yes git@github.com 2>&1 | \
  grep -Ei "Offering public key|Authentications that can continue|successfully authenticated"
```

| verbose 表现 | 结论 |
|---|---|
| 出现 `Offering public key: ... explicit agent` 后被拒 | agent 与密钥正常，问题在 GitHub 端（未添加 / 加错账号 / 加成了 Deploy key） |
| 完全没有 `Offering public key` | agent 里没有密钥，需先 export socket 并 `ssh-add` |

### 旁证：用公开 API 核对账号公钥

`GET /users/{username}/keys` 免认证即可读取，适合用来判断公钥是否真的存进了**账号**：

```bash
curl -sS https://api.github.com/users/<username>/keys
# []        -> 账号下没有任何公钥
# [...]     -> 已登记，可逐条比对
```

## 六、推送

### 确认目标仓库信息（公开仓库可匿名读取）

```bash
curl -sS https://api.github.com/repos/<owner>/<repo>
# 关注字段：default_branch / size / private / visibility
```

### 推荐：clone 后追加（保留远程历史）

```bash
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
git clone git@github.com:<owner>/<repo>.git
cd <repo>
# ...新增或修改文件...
git add -A && git commit -m "docs: 补充 gh + ssh-agent 推送链路说明"
git push origin main
```

仅在目标为空仓库、且明确需要覆盖时才使用 `git init` + `git push --force`。

### 提交作者信息

推送前核对，避免占位邮箱污染历史：

```bash
git config user.name  "<name>"
git config user.email "<id>+<username>@users.noreply.github.com"
```

## 七、排错速查表

| 症状 | 原因 | 处理 |
|---|---|---|
| `gh: command not found` | 未安装或不在 PATH | 走第三章；确认 `which gh` |
| `InternetOpenUrl() failed 0x80072efd` | winget 下载通道被拦 | 走镜像便携安装 |
| `CONNECT tunnel failed, response 502` | 代理拦截目标域名 | 换镜像；SSH 走 22 端口不受影响 |
| `Could not open a connection to your authentication agent` | 未 export `SSH_AUTH_SOCK`，或用了随机 socket | 固定 socket + 每次 export |
| `Permission denied (publickey)`，verbose 已 offering | GitHub 端没有这把公钥 | 检查是否加成了 Deploy key、或加错账号 |
| `Permission denied`，verbose 无 offering | agent 中无密钥 | `ssh-add ~/.ssh/id_ed25519` |
| `git push` 要求输入密码 | remote 仍是 HTTPS | `git remote set-url origin git@github.com:<owner>/<repo>.git` |
| `sc.exe` 被安全策略阻止 | 无法用 `sc query` 查看 ssh-agent 服务 | 绕开系统服务，直接用 Git Bash 的 `ssh-agent` |

## 八、安全约定

- 私钥永不离开本机，不写入对话、文件或提交；
- PAT 用完即撤；不在命令历史、脚本或 remote URL 中留痕（推送后把 remote URL 重置为不含凭据的地址）；
- 提交前扫描提交内容，确认无密钥残留。

## 九、自检清单

- [ ] `gh --version` 正常输出
- [ ] `ssh-add -l` 能列出目标密钥
- [ ] `ssh -T git@github.com` 返回 `successfully authenticated`
- [ ] `git remote -v` 显示 `git@github.com:` 形式
- [ ] `git push` 成功，提交作者信息正确
- [ ] 推送内容中不含任何密钥或凭据

## 参考

- GitHub Docs — [Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- GitHub Docs — [Managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- GitHub CLI — [Manual installation](https://github.com/cli/cli#installation)
