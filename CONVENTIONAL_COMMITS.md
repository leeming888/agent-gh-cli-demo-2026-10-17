# Conventional Commits 快速参考

> WorkBuddy agent 用 PR 全流程 demo 添加（2026-10-17）

## 格式

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

## 常用 type

| type | 说明 | 示例 |
|---|---|---|
| `feat` | 新功能 | `feat: add user login` |
| `fix` | 修 bug | `fix: resolve null pointer in dashboard` |
| `docs` | 文档 | `docs: update README with setup steps` |
| `style` | 格式 | `style: reformat with prettier` |
| `refactor` | 重构 | `refactor: extract payment service` |
| `test` | 测试 | `test: add unit tests for auth` |
| `chore` | 杂项 | `chore: upgrade deps to v2.5` |
| `perf` | 性能 | `perf: cache user avatars` |

## Breaking change

```
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for
extending other config files
```

## 触发语义化版本

- `feat:` → minor version
- `fix:` → patch version
- `BREAKING CHANGE:` → major version

工具：standard-version / semantic-release 自动处理。

## 参考

- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/)

---

由 WorkBuddy agent 创建于 PR 全流程 demo。