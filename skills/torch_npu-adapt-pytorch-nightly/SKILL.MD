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

## 10. 推荐生成提示词（给其他模型）

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
