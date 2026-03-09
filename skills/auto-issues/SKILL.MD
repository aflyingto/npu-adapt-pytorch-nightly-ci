# skills/auto-issues/SKILL.MD

目标：仅根据本文件，自动生成可运行的 `.github/workflows/auto-issues.yml`，用于将当前项目的兼容性分析报告自动提交到 torch_npu 验证镜像仓库的 Issues。

## 1. 输出文件与触发

- 输出文件路径：`.github/workflows/auto-issues.yml`
- workflow 名称：`Auto Submit Issues to torch_npu`
- 触发方式：
  - 推送：`push` - 路径包括 `issues/**`
  - 手动：`workflow_dispatch`
- 目标仓库：`https://github.com/aflyingto/torch_npu`

## 2. Job 基础信息

- 单一 job：`submit-issues`
- job 名称：`Submit Issues to torch_npu`
- runner：`ubuntu-22.04`
- 权限：需要 `issues: write` 权限

## 3. 必须包含的步骤（顺序固定）

1. `actions/checkout@v4`
2. 遍历 `issues/` 目录下所有 `.md` 文件
3. 对每个文件：
   - 提取 issue 标题（第一行 `# [编号]` 格式）
   - 提取 issue 内容（去除标题后的全部内容）
   - 检查目标仓库是否已存在相同标题的 issue
   - 若不存在，创建新 issue
   - 若已存在，跳过并记录
4. 输出处理结果 summary

## 4. Issue 内容提取规则

### 标题提取

从文件第一行提取标题，格式为 `# [编号] 标题内容`：
```bash
TITLE=$(head -n 1 "$issue_file" | sed 's/^# \[.*\] //')
```

### 编号提取

从文件第一行提取编号，格式为 `# [YYYY-MM-DD-NNN]`：
```bash
ISSUE_ID=$(head -n 1 "$issue_file" | sed 's/^# \[\(.*\)\] .*/\1/')
```

### 内容提取

去除第一行标题后的全部内容作为 issue body：
```bash
BODY=$(tail -n +2 "$issue_file")
```

### Label 设置

根据 issue 内容自动设置 label：
- 包含 `编译失败` → `bug`
- 包含 `API` → `api-change`
- 包含 `PyTorch nightly` → `pytorch-nightly`

## 5. GitHub API 调用规则

### 检查已存在 Issue

使用 GitHub CLI 搜索：
```bash
gh issue list -R aflyingto/torch_npu --search "in:title ${TITLE}" --state all
```

### 创建 Issue

使用 GitHub CLI 创建：
```bash
gh issue create -R aflyingto/torch_npu \
  --title "[${ISSUE_ID}] ${TITLE}" \
  --body "${BODY}" \
  --label "${LABELS}"
```

### 环境变量

需要设置 GitHub Token：
```yaml
env:
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 6. 错误处理规则

1. **文件格式错误**：跳过该文件，记录警告，继续处理其他文件
2. **API 调用失败**：记录错误，继续处理其他文件
3. **Issue 已存在**：跳过，记录为 "skipped"
4. **创建成功**：记录 issue URL

## 7. Summary 规则

输出处理结果表格：

```markdown
## Issue 提交结果

| 编号 | 标题 | 状态 | Issue URL |
|------|------|------|-----------|
| 2026-03-07-001 | CachingHostAllocator 编译失败 | ✅ 已创建 | https://github.com/aflyingto/torch_npu/issues/123 |
| 2026-03-07-002 | NPUSHMEMSymmetricMemory 编译失败 | ⏭️ 已存在 | https://github.com/aflyingto/torch_npu/issues/124 |
```

## 8. 关键防坑（必须遵守）

1. **必须使用 GitHub Token**
   - 使用 `secrets.GITHUB_TOKEN` 或自定义 token
   - Token 需要有目标仓库的 `issues: write` 权限

2. **必须检查重复**
   - 创建前先搜索是否已存在相同标题的 issue
   - 避免重复提交

3. **标题格式必须统一**
   - 格式：`[YYYY-MM-DD-NNN] 标题内容`
   - 便于追踪和去重

4. **处理文件不存在的情况**
   - `issues/` 目录可能为空
   - 使用 `[ -d "issues" ]` 检查目录存在

5. **多行内容处理**
   - Issue body 可能包含多行
   - 使用 heredoc 或文件传递给 `gh issue create`

## 9. 最小验收标准

- workflow 可被 GitHub Actions 识别并触发
- 正确读取 `issues/` 目录下的 `.md` 文件
- 成功创建 issue 到目标仓库
- 不重复创建已存在的 issue
- 输出正确的 summary

## 10. 完整脚本示例

```bash
#!/bin/bash
set -e

ISSUES_DIR="issues"
TARGET_REPO="aflyingto/torch_npu"
CREATED=0
SKIPPED=0
FAILED=0

echo "## Issue 提交结果" >> $GITHUB_STEP_SUMMARY
echo "" >> $GITHUB_STEP_SUMMARY
echo "| 编号 | 标题 | 状态 | Issue URL |" >> $GITHUB_STEP_SUMMARY
echo "|------|------|------|-----------|" >> $GITHUB_STEP_SUMMARY

for issue_file in "${ISSUES_DIR}"/*.md; do
  [ -f "$issue_file" ] || continue
  
  # 提取标题和编号
  FIRST_LINE=$(head -n 1 "$issue_file")
  ISSUE_ID=$(echo "$FIRST_LINE" | sed 's/^# \[\(.*\)\] .*/\1/')
  TITLE=$(echo "$FIRST_LINE" | sed 's/^# \[.*\] //')
  
  # 提取内容（去除第一行）
  BODY=$(tail -n +2 "$issue_file")
  
  # 设置 labels
  LABELS="bug,pytorch-nightly"
  
  # 检查是否已存在
  EXISTING=$(gh issue list -R "$TARGET_REPO" --search "in:title [${ISSUE_ID}]" --state all --json number -q '.[0].number' 2>/dev/null || echo "")
  
  if [ -n "$EXISTING" ]; then
    echo "| ${ISSUE_ID} | ${TITLE} | ⏭️ 已存在 | https://github.com/${TARGET_REPO}/issues/${EXISTING} |" >> $GITHUB_STEP_SUMMARY
    SKIPPED=$((SKIPPED + 1))
    continue
  fi
  
  # 创建 issue
  ISSUE_URL=$(gh issue create -R "$TARGET_REPO" \
    --title "[${ISSUE_ID}] ${TITLE}" \
    --body-file - \
    --label "$LABELS" <<< "$BODY" 2>/dev/null) || {
    echo "| ${ISSUE_ID} | ${TITLE} | ❌ 失败 | - |" >> $GITHUB_STEP_SUMMARY
    FAILED=$((FAILED + 1))
    continue
  }
  
  echo "| ${ISSUE_ID} | ${TITLE} | ✅ 已创建 | ${ISSUE_URL} |" >> $GITHUB_STEP_SUMMARY
  CREATED=$((CREATED + 1))
done

echo "" >> $GITHUB_STEP_SUMMARY
echo "**统计**：创建 ${CREATED} 个，跳过 ${SKIPPED} 个，失败 ${FAILED} 个" >> $GITHUB_STEP_SUMMARY
```

## 11. 推荐生成提示词（给其他模型）

"请严格按 `SKILL.MD` 生成 `.github/workflows/auto-issues.yml`。要求：使用 `gh` CLI 操作 GitHub Issues，遍历 `issues/` 目录下的 `.md` 文件，提取标题和内容，检查目标仓库 `aflyingto/torch_npu` 是否已存在相同 issue，若不存在则创建，输出处理结果表格到 summary。生成后输出完整 YAML。"

---

## 相关文件

- `issues/` - 兼容性分析报告目录
- `.github/workflows/auto-issues.yml` - 自动提交 issue 的 workflow
