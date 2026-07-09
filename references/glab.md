# GitLab (glab) 命令参考

GitLab 的行级评论通过 MR discussions API 的 `position` 参数实现。`glab api` 的 `:id` 占位符会自动替换为当前项目。

## 前置变量：三个 SHA 必须先拿到

行级评论必须携带 `base_sha` / `head_sha` / `start_sha`，从 MR 的 `diff_refs` 获取：

```bash
MR_IID=$(glab mr view --output json | jq -r '.iid')
DIFF_REFS=$(glab api "projects/:id/merge_requests/$MR_IID" | jq '.diff_refs')
BASE_SHA=$(echo "$DIFF_REFS" | jq -r '.base_sha')
HEAD_SHA=$(echo "$DIFF_REFS" | jq -r '.head_sha')
START_SHA=$(echo "$DIFF_REFS" | jq -r '.start_sha')
```

获取 diff 本体：

```bash
glab mr diff "$MR_IID"
```

## 1. 单行评论

```bash
glab api "projects/:id/merge_requests/$MR_IID/discussions" -X POST \
  -f body='[blocker] <问题>：<证据/后果>。建议 <最小修改方向>。' \
  -f "position[position_type]=text" \
  -f "position[base_sha]=$BASE_SHA" \
  -f "position[head_sha]=$HEAD_SHA" \
  -f "position[start_sha]=$START_SHA" \
  -f "position[new_path]=src/lib/auth.ts" \
  -f "position[old_path]=src/lib/auth.ts" \
  -F "position[new_line]=42"
```

### 行号参数规则（精确，传错会 400/500）

| 目标行类型 | 传参 |
|-----------|------|
| 新增行 | 只传 `position[new_line]`（新文件行号） |
| 删除行 | 只传 `position[old_line]`（旧文件行号） |
| 未改动的上下文行 | `old_line` 和 `new_line` **都传**（各自文件中的行号，从 diff hunk 头 `@@ -a,b +c,d @@` 逐行推算） |

- `new_path` / `old_path` 都必填；仅文件重命名时两者不同。
- 行号必须落在该 MR 的 diff 范围内。

## 2. 多行区间评论

多行评论在单行参数的基础上追加 `position[line_range]`，`start` / `end` 各需要 `line_code` 和 `type`：

**`line_code` 格式**：`<文件路径的SHA1>_<旧文件行号>_<新文件行号>`（此格式来自 GitLab 内部实现，文档未显式给出，升级大版本后如遇 400 需重新验证）。

- `type`：`new` = 锚在新增/改动后一侧，`old` = 锚在删除一侧。
- 新增行的"旧文件行号"取该行插入位置上方最近一条旧文件行的行号（从 diff hunk 逐行推算新旧行号对）。

```bash
FILE="src/lib/retry.ts"
FILE_SHA=$(printf '%s' "$FILE" | shasum | awk '{print $1}')

# 区间：新文件第 10-15 行（均为新增行，插入点上方旧文件行号为 9）
glab api "projects/:id/merge_requests/$MR_IID/discussions" -X POST \
  -f body='[suggestion] <针对整段区间的意见>。' \
  -f "position[position_type]=text" \
  -f "position[base_sha]=$BASE_SHA" \
  -f "position[head_sha]=$HEAD_SHA" \
  -f "position[start_sha]=$START_SHA" \
  -f "position[new_path]=$FILE" \
  -f "position[old_path]=$FILE" \
  -F "position[new_line]=15" \
  -f "position[line_range][start][line_code]=${FILE_SHA}_9_10" \
  -f "position[line_range][start][type]=new" \
  -f "position[line_range][end][line_code]=${FILE_SHA}_9_15" \
  -f "position[line_range][end][type]=new"
```

注意：`position[new_line]`（或 `old_line`）仍需传，取区间**末行**的行号，与 `line_range][end]` 一致。

## 3. 总评（不带行位置）

```bash
glab mr note "$MR_IID" -m "总评：整体评价 + 分级统计 + 未展开的次要观察。"
```
