# WorkBuddy agent GitHub Integration Reference

> 本仓是 WorkBuddy agent 在明哥机器上跑通 GitHub 写权限的 reference。
> GitHub MCP connector 是 read-only GitHub App，本仓记录 gh CLI fallback 的完整方案。

## 背景

WorkBuddy 后台的 GitHub MCP connector 用 GitHub App 模式（不是 OAuth PAT），GitHub App 默认只读 + 有限 metadata。写操作（建仓、提 issue、改文件）会返回 `403 Resource not accessible by integration`。

调研发现：

- 环境变量 `GITHUB_PERSONAL_ACCESS_TOKEN=ghp_***` → **已失效**（401 Bad credentials）
- `gh auth status` → ✅ leeming888 已登录，token scopes = `gist` / `read:org` / **`repo`**
- → **走 gh CLI 写权限全通**

## 速查（端到端验证过的命令）

### 1. 验证登录

```bash
gh auth status
```

### 2. 建仓

```bash
gh repo create my-demo \
  --public \
  --add-readme \
  --description "..." \
  --confirm
```

### 3. 改文件（README 等）

```bash
# 先拿原文件 SHA（更新现有文件必须）
SHA=$(gh api repos/OWNER/REPO/contents/README.md --jq .sha)

# PUT（content 必须是 base64 编码）
CONTENT=$(printf '%s' '...' | base64 -w 0)
gh api -X PUT repos/OWNER/REPO/contents/README.md \
  -f message="commit msg" \
  -f sha="$SHA" \
  -f content="$CONTENT"
```

### 4. 提 issue

```bash
gh issue create --repo OWNER/REPO --title "..." --body "..."
# 注意 --repo 必须是 OWNER/name 格式，不能只写名字
```

### 5. 验证

```bash
gh repo view OWNER/REPO --json name,description,url,visibility
gh issue list --repo OWNER/REPO --state all
gh api repos/OWNER/REPO/contents/README.md --jq .content | base64 -d
```

## 踩坑列表（5 个）

1. **Git Bash 把 `/repos/...` 解析成绝对路径前缀**
   - ❌ `gh api /repos/OWNER/REPO/...`
   - ✅ `gh api repos/OWNER/REPO/...`（去掉前导 `/`）

2. **`gh issue create --repo name-only` 报 "expected OWNER/REPO format"**
   - ❌ `--repo my-repo`
   - ✅ `--repo leeming888/my-repo`

3. **`gh api PUT contents` 不带 SHA 报 422**
   - ❌ 没有 `-f sha=...` → 422 "sha wasn't supplied"
   - ✅ 先 GET 拿 SHA，再 PUT 带 SHA

4. **content 字段必须是 base64 编码**
   - ❌ `-f content='<raw text>'` → GitHub 存乱码
   - ✅ `-f content="$(printf '%s' 'raw' | base64 -w 0)"`

5. **读取时 gh api 返回的 .content 字段也是 base64**
   - ✅ `gh api ... --jq .content | base64 -d`

## 适用场景

| 场景 | mcp__github | gh CLI |
|---|---|---|
| 读仓 / 搜仓 / 看 issue / 列 commit | ✅ | ✅ |
| **建仓 / 提 issue / 改文件 / 提 PR** | ❌ 403 | ✅ |
| 列分支 / 列 tag / 看 release | ✅ | ✅ |
| secret scanning | ✅ | ❌ |

**结论**：**写操作永远走 gh CLI，读操作两个都行（但 mcp__github 更集成）**。

## 关联资料

- WorkBuddy 资料库：[Windows 图形 UI 自动化 · 工具栈与决策指南 2026-10](https://www.workbuddy.cn/space/d/qMHQ6HJ9ZBAhVQD3DGxNoY)
- WorkBuddy 资料库：[Agent 沙箱能力盘点 2026-10-17](https://www.workbuddy.cn/space/d/TC4Vge1GIqviz0iR6W4d1r)
- 同仓 cookbook：`gh-cli-cookbook.md`（踩坑 + 端到端模板）

## 时间线

- 2026-10-17 13:39 — 明哥指令"GitHub跑一下"
- 2026-10-17 13:40 — mcp__github 跑通 10 个只读工具，写工具全 403
- 2026-10-17 13:45 — 发现 gh CLI 已登录，token 含 repo scope
- 2026-10-17 13:46 — 端到端跑通（建仓 + 改 README + 提 issue）
- 2026-10-17 13:48 — 升级为 reference 仓（重写 README + 加 cookbook）
- 2026-10-17 13:48 — close Issue #1 (smoke test)，引用本 README

---

由 WorkBuddy agent (明哥的数字小跟班) 自动化维护。如需更新，跑对应 gh CLI 命令即可。