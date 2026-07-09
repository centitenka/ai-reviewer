# GitHub (gh) 命令参考

## 前置变量

```bash
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
PR=<PR 编号>
```

## 1. 获取 diff 与上下文

```bash
gh pr view "$PR" --json title,body,baseRefName,headRefName,headRefOid,statusCheckRollup
gh pr diff "$PR"
```

评论行号必须落在 `gh pr diff` 输出的 hunk 范围内（含上下文行），否则 API 返回 422。

## 2. 一次性发布整个 review（推荐）

把全部行级评论和总评组装成一个 review，一次调用发布——比逐条发评论少打扰订阅者，且保证原子性：

```bash
cat > "$SCRATCHPAD/review.json" <<'EOF'
{
  "event": "COMMENT",
  "body": "总评：整体评价 + 分级统计 + 未展开的次要观察。",
  "comments": [
    {
      "path": "src/lib/auth.ts",
      "line": 42,
      "side": "RIGHT",
      "body": "[blocker] 单行评论示例：token 过期未处理并发刷新，两个请求会同时触发 refresh。建议加互斥锁或复用进行中的 Promise。"
    },
    {
      "path": "src/lib/retry.ts",
      "start_line": 10,
      "start_side": "RIGHT",
      "line": 15,
      "side": "RIGHT",
      "body": "[suggestion] 多行区间评论示例：这段重试循环没有退避，失败时会打满下游。可加指数退避。"
    }
  ]
}
EOF

gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" --input "$SCRATCHPAD/review.json"
```

### 行锚定参数语义（精确规则）

| 参数 | 含义 |
|------|------|
| `line` | 评论锚定的行号；多行评论时是**区间末行** |
| `side` | `RIGHT` = 改动后的文件（新增行、未变行）；`LEFT` = 改动前的文件（删除行） |
| `start_line` | 仅多行评论需要；区间首行，必须 `< line` |
| `start_side` | 区间首行所在的 side，通常与 `side` 相同 |

- 单行评论：只传 `line` + `side`。
- 多行评论：四个参数都传。
- 评删除的行：`side=LEFT`，行号用旧文件行号。
- `event` 用 `COMMENT`；`REQUEST_CHANGES`/`APPROVE` 只在用户明确要求时使用（且对自己的 PR 调用会失败）。

## 3. 逐条发布（备选）

需要单独补一条时用 comments 端点，`commit_id` 必填（用 head SHA）：

```bash
HEAD_SHA=$(gh pr view "$PR" --json headRefOid --jq '.headRefOid')
gh api "repos/$OWNER/$REPO/pulls/$PR/comments" \
  -f commit_id="$HEAD_SHA" \
  -f path="src/lib/auth.ts" \
  -F line=42 -f side=RIGHT \
  -f body="[nit] ..."
```

多行同样加 `-F start_line=<首行> -f start_side=RIGHT`。
