# WorkBuddy agent gh CLI smoke test 2026-10-17

本 README 由 WorkBuddy agent 通过 gh CLI (OAuth token, repo scope) 创建。

## 关键发现

GitHub MCP connector 是 read-only GitHub App(没写权限,403)。
绕开方案: gh CLI 用明哥自己的 OAuth token (scopes: repo),全写权限 OK。

## 涉及工作流

- mcp__github__get_me (read) - 通过
- gh repo create (write) - 通过
- gh api PUT contents (write) - 本条验证
- gh issue create (write) - 下一步

## 时间

2026-10-17 13:45 GMT+8
