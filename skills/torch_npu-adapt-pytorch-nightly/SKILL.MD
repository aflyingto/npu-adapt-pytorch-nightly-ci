# skills/torch_npu-adapt-pytorch-nightly/SKILL.MD

目标：仅根据本文件，自动生成可运行的 `.github/workflows/nightly-build.yml`，用于验证 `Ascend/pytorch` 在 PyTorch nightly 版本下的编译兼容性。

## 1. 输出文件与触发

- 输出文件路径：`.github/workflows/nightly-build.yml`
- workflow 名称：`Ascend/pytorch Nightly Build Validation`
- 触发方式：
  - 定时：`schedule` - 每天 UTC 02:00（北京时间 10:00）
  - 手动：`workflow_dispatch` - 可选参数 `torch_nightly_date`（格式：YYYYMMDD）
  - 推送：`push` - 路径包括 `.github/workflows/nightly-build.yml`、`patches/**`、`AGENTS.md`
- 全局环境变量：`PYTHON_VERSION: '3.11'`

## 2. Job 基础信息

- 单一 job：`build`
- job 名称：`Build torch_npu (x86, PyTorch nightly)`
- runner：`ubuntu-22.04`

## 3. 必须包含的步骤（顺序固定）

1. `actions/checkout@v4`
2. `actions/setup-python@v5`，版本使用 `${{ env.PYTHON_VERSION }}`
3. 安装系统依赖：`cmake ninja-build gcc g++ git patchelf ccache`
4. 恢复 ccache：`actions/cache/restore@v4`
5. 安装 PyTorch nightly：
   - 升级 pip
   - 若指定日期：安装 `torch==2.11.0.dev${DATE}`（从 nightly/cpu 索引）
   - 否则：安装最新 `torch --pre`（从 nightly/cpu 索引）
   - 输出版本到 `$GITHUB_OUTPUT`
6. 克隆 `Ascend/pytorch`：
   - 参数：`--depth=1 --recurse-submodules`
   - 目标目录：`ascend_pytorch`
   - 记录 commit 信息（完整 hash、短 hash、日期）
7. 安装 Python 构建依赖：`pyyaml setuptools auditwheel`
8. 应用兼容性 patches：
   - 遍历 `patches/` 目录下所有 `.patch` 文件
   - 使用 `git apply --directory=ascend_pytorch` 应用
   - 记录成功/失败列表到 `$GITHUB_OUTPUT`
9. 配置 ccache 环境变量
10. 构建 wheel：`python setup.py build bdist_wheel`
11. 保存 ccache：`actions/cache/save@v4`（`if: always()`）
12. 上传构建日志 artifact（`if: always()`）
13. 上传 wheel artifact（仅成功时）
14. 输出构建 summary

## 4. ccache 规则

- 使用 `cache/restore` + `cache/save` 分离模式
- 缓存路径：`~/.ccache`
- 缓存 key：`ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}`
- `restore-keys`：
  - `ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-`
  - `ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-`
- 配置步骤：
  - `ccache -M 2G`
  - `ccache -z`（重置统计）
  - 设置环境变量：
    - `CC=ccache gcc`
    - `CXX=ccache g++`
    - `CCACHE_BASEDIR=${PWD}`
    - `CCACHE_DIR=$HOME/.ccache`
    - `CCACHE_COMPRESS=1`
    - `CCACHE_MAXSIZE=2G`
- 构建后：
  - 提取命中率：`ccache -s | awk -F'[()]' '/Hits:/ {print $2; exit}'`
  - 输出到 `$GITHUB_OUTPUT`

## 5. 构建策略

- 启用 torchair：`export DISABLE_INSTALL_TORCHAIR=FALSE`
- 不禁用 RPC：不设置 `DISABLE_RPC_FRAMEWORK`
- 保留：`export BUILD_WITHOUT_SHA=1`
- 构建命令：
  - 在 `ascend_pytorch` 目录下执行
  - 日志输出：`python setup.py build bdist_wheel 2>&1 | tee /tmp/build.log`
  - 使用 `${PIPESTATUS[0]}` 获取构建状态
- 构建成功时：
  - 列出生成的 wheel 文件
  - 输出到 `$GITHUB_OUTPUT`

## 6. Patch 应用规则

- 遍历方式：`for p in "${PATCH_DIR}"/*.patch; do [ -f "$p" ] || continue; ... done`
- 应用命令：`git apply --directory=ascend_pytorch "$p"`
- **禁止**：`cd ascend_pytorch && git apply`（会导致路径解析错误）
- 记录结果：
  - 成功：追加到 `PATCH_APPLIED`
  - 失败：追加到 `PATCH_FAILED`，不中断流程

## 7. Summary 规则

- 使用表格格式输出构建信息：
  - 表头：`| 项目 | 详情 |`
  - 包含：构建时间、PyTorch Nightly 版本、Ascend/pytorch Commit（带链接）、Commit 时间、已应用 Patch、ccache 命中、ccache 命中率、构建结果
- 构建结果使用图标：
  - 成功：`✅ SUCCESS`
  - 失败：`❌ FAILED`
- 成功时列出 wheel 文件
- 失败时提示查看 build-log artifact

## 8. 关键防坑（必须遵守）

1. 日志 artifact 必须 `if: always()`
   - 保证失败时也能下载日志

2. wheel 上传必须仅在成功时执行
   - 条件：`if: steps.build.outputs.status == '0'`

3. ccache 保存必须 `if: always()`
   - 即使构建失败也要保存缓存

4. Patch 应用必须使用 `--directory` 参数
   - 错误：`cd ascend_pytorch && git apply "$patch"`
   - 正确：`git apply --directory=ascend_pytorch "$patch"`

5. `$GITHUB_OUTPUT` 必须是单行值
   - 禁止输出多行内容

6. 构建状态使用 `${PIPESTATUS[0]}` 而非 `$?`
   - 因为使用了 `tee` 管道

## 9. 最小验收标准

- workflow 可被 GitHub Actions 识别并触发
- 克隆 Ascend/pytorch（master 分支）
- 正确应用 patches 目录下的 patch 文件
- 构建失败时可下载 `build-log-{run_number}`
- 构建成功时可下载 `dist/*.whl`
- ccache 在第二次运行时命中率显著提升

## 10. 完整 Workflow 模板

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

## 11. 推荐生成提示词（给其他模型）

"请严格按 `build_skills.md` 生成 `.github/workflows/nightly-build.yml`。要求：按文档顺序组织 step，使用 `cache/restore` + `cache/save` 分离模式，克隆 Ascend/pytorch master 分支，应用 `patches/` 目录下的兼容性 patch，设置 `DISABLE_INSTALL_TORCHAIR=FALSE`，使用表格格式输出 step summary。生成后输出完整 YAML。"

---

## 附录：常见问题修复模式

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
