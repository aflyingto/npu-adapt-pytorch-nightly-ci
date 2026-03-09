# pytorch-npu CI Skill

本 Skill 用于生成和管理 Ascend/pytorch 与 PyTorch nightly 版本的兼容性验证 CI。

## 快速生成 Workflow

当需要创建或更新 `.github/workflows/nightly-build.yml` 时，直接使用以下完整模板：

### 完整 Workflow 模板

```yaml
name: Ascend/pytorch Nightly Build Validation

on:
  schedule:
    - cron: '0 2 * * *'   # 每天 UTC 02:00（北京时间 10:00）
  workflow_dispatch:
    inputs:
      torch_nightly_date:
        description: 'PyTorch nightly 日期 (格式: YYYYMMDD，留空使用最新版)'
        required: false
        default: ''
  push:
    paths:
      - '.github/workflows/nightly-build.yml'
      - 'patches/**'
      - 'AGENTS.md'

env:
  PYTHON_VERSION: '3.11'

jobs:
  build:
    name: Build torch_npu (x86, PyTorch nightly)
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python ${{ env.PYTHON_VERSION }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install system dependencies
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y --no-install-recommends \
            cmake ninja-build \
            gcc g++ \
            git \
            patchelf \
            ccache

      - name: Restore ccache
        id: ccache-restore
        uses: actions/cache/restore@v4
        with:
          path: ~/.ccache
          key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}
          restore-keys: |
            ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-
            ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-

      - name: Install PyTorch nightly (CPU, x86_64)
        id: install_torch
        run: |
          pip install --upgrade pip
          if [ -n "${{ github.event.inputs.torch_nightly_date }}" ]; then
            DATE="${{ github.event.inputs.torch_nightly_date }}"
            pip install --pre \
              "torch==2.11.0.dev${DATE}" \
              --index-url https://download.pytorch.org/whl/nightly/cpu
          else
            pip install --pre torch \
              --index-url https://download.pytorch.org/whl/nightly/cpu
          fi
          TORCH_VER=$(python -c "import torch; print(torch.__version__)")
          echo "version=${TORCH_VER}" >> $GITHUB_OUTPUT
          echo "PyTorch nightly version: ${TORCH_VER}"

      - name: Clone Ascend/pytorch (with submodules)
        id: clone_repo
        run: |
          git clone --depth=1 --recurse-submodules \
            https://github.com/Ascend/pytorch.git ascend_pytorch
          cd ascend_pytorch
          COMMIT=$(git rev-parse HEAD)
          COMMIT_SHORT=$(git rev-parse --short HEAD)
          COMMIT_DATE=$(git log -1 --format='%ci')
          echo "commit=${COMMIT}" >> $GITHUB_OUTPUT
          echo "commit_short=${COMMIT_SHORT}" >> $GITHUB_OUTPUT
          echo "commit_date=${COMMIT_DATE}" >> $GITHUB_OUTPUT
          echo "Ascend/pytorch commit: ${COMMIT} (${COMMIT_DATE})"

      - name: Install Python build dependencies
        run: |
          cd ascend_pytorch
          pip install pyyaml setuptools auditwheel

      - name: Apply compatibility patches
        id: patch
        run: |
          PATCH_DIR="${GITHUB_WORKSPACE}/patches"
          PATCH_APPLIED=""
          PATCH_FAILED=""
          for p in "${PATCH_DIR}"/*.patch; do
            [ -f "$p" ] || continue
            echo "Applying patch: $(basename $p)"
            if git apply --directory=ascend_pytorch "$p"; then
              echo "  ✅ OK"
              PATCH_APPLIED="${PATCH_APPLIED} $(basename $p)"
            else
              echo "  ❌ FAILED (may already be merged upstream)"
              PATCH_FAILED="${PATCH_FAILED} $(basename $p)"
            fi
          done
          echo "applied=${PATCH_APPLIED}" >> $GITHUB_OUTPUT
          echo "failed=${PATCH_FAILED}"   >> $GITHUB_OUTPUT

      - name: Build torch_npu wheel
        id: build
        run: |
          cd ascend_pytorch
          mkdir -p ~/.ccache
          ccache -M 2G
          ccache -z || true
          export CC="ccache gcc"
          export CXX="ccache g++"
          export CCACHE_BASEDIR="${PWD}"
          export CCACHE_DIR="${HOME}/.ccache"
          export CCACHE_COMPRESS=1
          export CCACHE_MAXSIZE=2G
          export DISABLE_INSTALL_TORCHAIR=FALSE
          export BUILD_WITHOUT_SHA=1
          python setup.py build bdist_wheel 2>&1 | tee /tmp/build.log
          BUILD_STATUS=${PIPESTATUS[0]}
          CCACHE_HIT_RATE=$(ccache -s | awk -F'[()]' '/Hits:/ {print $2; exit}')
          [ -n "${CCACHE_HIT_RATE}" ] || CCACHE_HIT_RATE="N/A"
          echo "hit_rate=${CCACHE_HIT_RATE}" >> $GITHUB_OUTPUT
          ccache -s || true
          echo "status=${BUILD_STATUS}" >> $GITHUB_OUTPUT
          if [ ${BUILD_STATUS} -eq 0 ]; then
            WHL=$(ls dist/*.whl 2>/dev/null | head -1)
            echo "wheel=${WHL}" >> $GITHUB_OUTPUT
            echo "Build succeeded: ${WHL}"
          fi
          exit ${BUILD_STATUS}

      - name: Save ccache
        if: always()
        uses: actions/cache/save@v4
        with:
          path: ~/.ccache
          key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}

      - name: Upload build log
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: build-log-${{ github.run_number }}
          path: /tmp/build.log
          if-no-files-found: warn

      - name: Upload wheel
        if: steps.build.outputs.status == '0'
        uses: actions/upload-artifact@v4
        with:
          name: torch_npu-wheel-${{ github.run_number }}
          path: ascend_pytorch/dist/*.whl
          if-no-files-found: warn

      - name: Build summary
        if: always()
        run: |
          BUILD_STATUS="${{ steps.build.outputs.status }}"
          if [ "${BUILD_STATUS}" = "0" ]; then
            STATUS_ICON="✅ SUCCESS"
          else
            STATUS_ICON="❌ FAILED"
          fi

          cat >> $GITHUB_STEP_SUMMARY << EOF
          ## Ascend/pytorch Nightly Build Validation

          | 项目 | 详情 |
          |------|------|
          | 构建时间 | $(date -u '+%Y-%m-%d %H:%M UTC') |
          | PyTorch Nightly | \`${{ steps.install_torch.outputs.version }}\` |
          | Ascend/pytorch Commit | [\`${{ steps.clone_repo.outputs.commit_short }}\`](https://github.com/Ascend/pytorch/commit/${{ steps.clone_repo.outputs.commit }}) |
          | Commit 时间 | ${{ steps.clone_repo.outputs.commit_date }} |
          | 已应用 Patch | ${{ steps.patch.outputs.applied || '(无)' }} |
          | ccache 命中 | ${{ steps.ccache-restore.outputs.cache-hit == 'true' && '命中' || '未命中' }} |
          | ccache 命中率 | ${{ steps.build.outputs.hit_rate || 'N/A' }} |
          | 构建结果 | ${STATUS_ICON} |

          $( [ "${BUILD_STATUS}" = "0" ] && echo "> Wheel: \`${{ steps.build.outputs.wheel }}\`" || echo "> 查看 build-log artifact 获取详细错误信息" )
          EOF
```

---

## 配置项详解

### 触发条件

| 触发方式 | 说明 |
|----------|------|
| `schedule` | 每天 UTC 02:00 自动触发 |
| `workflow_dispatch` | 手动触发，可指定 PyTorch nightly 日期 |
| `push` | workflow 文件、patches 目录、AGENTS.md 变更时触发 |

### 环境变量

```yaml
env:
  PYTHON_VERSION: '3.11'  # Python 版本，需与 setup-python step 一致
```

### 系统依赖

必须安装的依赖：
- `cmake`, `ninja-build` - 构建工具
- `gcc`, `g++` - 编译器
- `git` - 版本控制
- `patchelf` - 修改 ELF 文件
- `ccache` - 编译缓存

### ccache 配置

**关键点**：
1. 使用 `cache/restore` + `cache/save` 分离模式
2. Key 格式：`ccache-{os}-py{ver}-{ref}-{run_id}`
3. `if: always()` 确保 always 保存缓存

```yaml
# 恢复缓存
- name: Restore ccache
  uses: actions/cache/restore@v4
  with:
    path: ~/.ccache
    key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}
    restore-keys: |
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-

# 保存缓存
- name: Save ccache
  if: always()
  uses: actions/cache/save@v4
  with:
    path: ~/.ccache
    key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}
```

### PyTorch 安装

```bash
# 指定日期版本
pip install --pre "torch==2.11.0.dev${DATE}" --index-url https://download.pytorch.org/whl/nightly/cpu

# 最新版本
pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cpu
```

### 克隆仓库

```bash
git clone --depth=1 --recurse-submodules https://github.com/Ascend/pytorch.git ascend_pytorch
```

**参数说明**：
- `--depth=1` - 浅克隆，节省时间
- `--recurse-submodules` - 递归克隆子模块

### Patch 应用

**正确方式**：
```bash
git apply --directory=ascend_pytorch "$patch"
```

**错误方式**：
```bash
cd ascend_pytorch && git apply "$patch"  # ❌ 路径解析错误
```

### 构建环境变量

```bash
export CC="ccache gcc"
export CXX="ccache g++"
export CCACHE_BASEDIR="${PWD}"      # 路径规范化，提高命中率
export CCACHE_DIR="${HOME}/.ccache"
export CCACHE_COMPRESS=1            # 压缩缓存
export CCACHE_MAXSIZE=2G
export DISABLE_INSTALL_TORCHAIR=FALSE  # 启用 torchair
export BUILD_WITHOUT_SHA=1              # 可复现构建
```

### GITHUB_OUTPUT 写法

```bash
# 正确：单行值
echo "version=${TORCH_VER}" >> $GITHUB_OUTPUT

# 错误：多行值会触发格式错误
echo "logs=$(cat build.log)" >> $GITHUB_OUTPUT  # ❌
```

---

## 常见问题修复

### Patch 应用失败

**错误**：`error: ascend_pytorch/torch_npu/...: No such file or directory`

**原因**：在 `ascend_pytorch` 目录内执行 `git apply`

**修复**：使用 `--directory` 参数
```bash
git apply --directory=ascend_pytorch "$patch"
```

### ccache 命中率低

**检查项**：
1. `CCACHE_BASEDIR` 是否设置
2. 缓存 key 是否正确恢复
3. `cache/save` 是否在 `always()` 条件下执行

### GITHUB_OUTPUT 格式错误

**错误**：`Unable to process file command 'output'`

**原因**：输出值包含换行符

**修复**：确保输出单行值
```bash
# 正确
echo "version=${TORCH_VER}" >> $GITHUB_OUTPUT

# 错误
echo "logs=$(cat build.log)" >> $GITHUB_OUTPUT
```

---

## 问题处理流程

### 典型工作流

```
CI 构建失败
  │
  ├─ /analyze-failure   ← 拉取日志、定位根因、对比 PyTorch 新旧 API
  │
  ├─ /report-issue      ← 按日期编号创建 issues/ 文档
  │
  └─ /gen-patch         ← 修改源码、生成 patch、提交、触发验证
```

### 常见修复模式

| 错误类型 | 修复方式 |
|----------|----------|
| 结构体成员被删除 | `.cpp` 模块级引入 `static map + mutex` 替代 |
| 成员重命名 | 重命名 Ascend 侧冲突的同名类型 |
| 基类新增纯虚函数 | 子类补充 override（无语义则返回 0） |
| 虚函数签名变更 | 更新 override 签名，原逻辑提取为私有方法 |

---

## 相关文件

- `AGENTS.md` - 项目级别的 Agent 指导文档
- `.github/workflows/nightly-build.yml` - CI workflow 定义
- `patches/` - 兼容性 patch 存放目录
- `issues/` - 问题分析记录目录
