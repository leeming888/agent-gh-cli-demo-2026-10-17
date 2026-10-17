# gh CLI Cookbook (WorkBuddy agent 实战)

收集 WorkBuddy agent 跑 GitHub 工作流时踩的坑 + 解决方案。所有命令均在明哥机器上端到端验证过（2026-10-17）。

## 核心姿势

### 1. 验证登录状态

```bash
gh auth status
```

输出关键信息：

```
github.com
  ✓ Logged in to github.com account leeming888 (keyring)
  - Active account: true
  - Git operations protocol: https
  - Token: gho_************************************
  - Token scopes: 'gist', 'read:org', 'repo'
```

**判断标准**：scopes 必须含 `repo`，否则写操作全 403。

### 2. 区分 MCP 和 gh CLI

| 工具 | 类型 | 写权限 | 推荐场景 |
|---|---|---|---|
| `mcp__github` | MCP connector（GitHub App）| ❌ 403 | 只读（搜仓、看 issue、看 commit）|
| `gh` CLI | 本地 OAuth token | ✅ repo scope | **写（建仓、改文件、提 issue、提 PR）**|

**铁律**：**写 GitHub 永远走 gh CLI**。

## 踩坑列表

### 坑 1: Git Bash 把 `/repos/...` 当绝对路径

```bash
# 错
gh api /repos/leeming888/my-repo/contents/README.md
# 报错：invalid API endpoint: "...D:/workbuddy/.../repos/.../README.md"
#      Your shell might be rewriting URL paths as filesystem paths

# 对
gh api repos/leeming888/my-repo/contents/README.md
```

### 坑 2: gh issue create --repo 必须是 OWNER/name

```bash
# 错
gh issue create --repo my-repo
# 报错：expected the "[HOST/]OWNER/REPO" format, got "my-repo"

# 对
gh issue create --repo leeming888/my-repo
```

### 坑 3: PUT contents 不带 SHA 报 422

```bash
# 错（缺 SHA）
gh api -X PUT repos/OWNER/REPO/contents/README.md \
  -f message="..." -f content="..."
# 报错："sha" wasn't supplied. (HTTP 422)

# 对
SHA=$(gh api repos/OWNER/REPO/contents/README.md --jq .sha)
gh api -X PUT repos/OWNER/REPO/contents/README.md \
  -f message="..." -f sha="$SHA" -f content="..."
```

### 坑 4: content 必须是 base64 编码

```bash
# 错
CONTENT='<raw text>'  # GitHub 会存乱码

# 对
CONTENT=$(printf '%s' 'raw text here' | base64 -w 0)
gh api ... -f content="$CONTENT"
```

### 坑 5: 读 .content 字段也要 base64 -d 解码

```bash
gh api repos/OWNER/REPO/contents/README.md --jq .content | base64 -d
```

### 坑 6: --confirm 已弃用

```bash
# 错（v2.93+）
gh repo create ... --confirm
# 警告：Flag --confirm has been deprecated, Pass any argument to skip confirmation prompt

# 对：传任意参数跳过确认
gh repo create ... --confirm yes
# 或直接不加（无 -y 时会问，但 --confirm 接受任意参数）
```

## 端到端模板（复制即可用）

### 场景 A: 建仓 + README + 提 issue

```bash
# 1. 建仓
gh repo create my-new-repo \
  --public --add-readme --description "demo" \
  --confirm yes

# 2. 改 README
SHA=$(gh api repos/leeming888/my-new-repo/contents/README.md --jq .sha)
CONTENT=$(printf '%s' '# My New Repo

这是 WorkBuddy agent 创建的 demo 仓。' | base64 -w 0)
gh api -X PUT repos/leeming888/my-new-repo/contents/README.md \
  -f message="init: WorkBuddy agent smoke test" \
  -f sha="$SHA" \
  -f content="$CONTENT"

# 3. 提 issue
gh issue create \
  --repo leeming888/my-new-repo \
  --title "[Smoke Test] gh CLI 端到端" \
  --body "..."
```

### 场景 B: 提 PR（需要先 fork + clone）

```bash
# 1. fork + clone
gh repo fork OWNER/REPO --clone=true
cd REPO

# 2. 切分支
git checkout -b my-feature

# 3. 改文件 + commit + push
echo "..." > new-file.md
git add new-file.md
git commit -m "feat: add new-file"
git push origin my-feature

# 4. 提 PR
gh pr create --title "..." --body "..."
```

### 场景 C: 改多个文件（push_files 模式）

```bash
# 注意：gh api 没有直接的 multi-file PUT，需要循环或用 mcp__github__push_files
# 但 mcp__github__push_files 在 MCP 写权限禁用状态下也会 403
# 所以实际姿势：git clone 本地改 + commit + push
```

## 调试技巧

```bash
# 看 gh 详细错误
GH_DEBUG=1 gh api repos/OWNER/REPO/contents/README.md

# 看 gh 全局配置
gh config list

# 重新登录
gh auth login --with-token < token.txt
# 或交互式
gh auth login
```

## 与 mcp__github__push_files 的关系

如果 MCP 写权限解锁（GitHub App 加 Contents:Write），`mcp__github__push_files` 可以批量推多个文件，比 gh CLI 循环更高效。

**当前状态**：MCP 写权限锁住 → 走 gh CLI。

## 参考

- [gh CLI 官方文档](https://cli.github.com/manual/)
- [GitHub REST API](https://docs.github.com/en/rest)
- 同仓 README（场景与适用矩阵）

---

更新时间：2026-10-17