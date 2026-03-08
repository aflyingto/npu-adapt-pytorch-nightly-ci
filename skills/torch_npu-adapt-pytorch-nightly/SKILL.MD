# pytorch-npu CI

每日自动验证 [Ascend/pytorch](https://github.com/Ascend/pytorch) 在 PyTorch 主干最新 nightly 版本下的编译兼容性。

## 工作原理

- **触发方式**：每天 UTC 02:00（北京时间 10:00）自动触发，或手动触发
- **运行环境**：GitHub Actions 免费 x86 runner（`ubuntu-22.04`）
- **Python 版本**：3.11（通过 `actions/setup-python@v5` 安装）
- **PyTorch 版本**：PyTorch 主干每日 nightly 构建（CPU 版）
- **CANN 依赖**：无需安装 CANN，使用仓库内置的桩库（`third_party/acl/libs/build_stub.sh`）

## 构建流程

1. 安装系统依赖（cmake, ninja-build, gcc, g++, git, patchelf, ccache）
2. 安装 Python 3.11
3. 恢复 ccache 缓存
4. 安装 PyTorch nightly（从 `download.pytorch.org/whl/nightly/cpu`）
5. 克隆 `Ascend/pytorch`（`--depth=1 --recurse-submodules`）
6. 安装 Python 构建依赖（pyyaml, setuptools, auditwheel）
7. 按序应用 `patches/` 目录下所有兼容性 patch
8. 执行 `python setup.py build bdist_wheel`
9. 保存 ccache 缓存
10. 上传构建日志和生成的 wheel 包

## 当前 CI 策略

- 默认启用 torchair：`DISABLE_INSTALL_TORCHAIR=FALSE`
- 不再强制禁用 RPC：不设置 `DISABLE_RPC_FRAMEWORK=TRUE`
- 启用 `ccache`：构建前 `cache/restore`，构建后 `cache/save`
- ccache key 格式：`ccache-{os}-py{python_version}-{ref_name}-{run_id}`
- Step Summary 输出 ccache 命中状态与命中率
- Patch 应用失败不中断流程，记录到 `failed` 输出

## 查看结果

- [Actions 页面](../../actions/workflows/nightly-build.yml) 查看每次构建状态
- 每次运行的 Step Summary 包含：
  - 构建时间
  - PyTorch Nightly 版本
  - Ascend/pytorch commit（带链接）
  - 已应用 patch 列表
  - ccache 命中状态与命中率
  - 构建结果
- 构建失败时可下载 `build-log-{run_number}` artifact 查看详细编译错误

### 结果判读要点

- `Build torch_npu wheel` 成功且出现 `Build succeeded: dist/*.whl`，说明核心编译成功。
- 如果同一 step 最后报 `Unable to process file command 'output'` / `Invalid format`，通常是 workflow 输出写法错误（`$GITHUB_OUTPUT` 格式）而非编译错误。
- `ccache` 首次冷启动命中率低属正常；第二次开始命中率应显著提升。

## 手动触发

在 Actions 页面点击 **Run workflow**，可选填 PyTorch nightly 日期（格式 `YYYYMMDD`），留空使用最新版。

---

## 问题处理（Claude Code Skills）

本仓库内置三个 Claude Code slash commands，覆盖从检测到修复的完整闭环。在项目目录下启动 Claude Code 后即可使用。

### 典型工作流

```
CI 构建失败
  │
  ├─ /analyze-failure   ← 拉取日志、定位根因、对比 PyTorch 新旧 API
  │
  ├─ /report-issue      ← 按日期编号创建 issues/ 文档
  │
  └─ /gen-patch         ← 修改源码、生成 patch、提交、触发验证
                               │
                        下次构建推进更远？
                          ├─ 是 → 继续 /analyze-failure 处理下一个问题
                          └─ 否 → 检查 patch 是否正确应用
```

---

### `/analyze-failure` — 分析 CI 失败原因

CI 变红后的第一步。执行内容：

1. `gh run list` 找到最新失败 Run
2. `gh run view --log-failed` 拉取失败日志
3. 过滤 `error:`、`make[2]`、`Traceback` 等关键行
4. 读取受影响的 Ascend/pytorch 源文件
5. 在 PyTorch nightly 头文件（`/usr/local/lib/python3.11/dist-packages/torch/include/`）中查找变更后的 API 定义
6. 输出结构化报告：已生效 patch、错误摘要、根本原因、受影响范围、建议修复方向

```
/analyze-failure
```

**关键观察**：构建运行时长比上次更长 → 前序 patch 已生效，本次暴露新问题。

**新增观察（重要）**：先区分"真实编译失败"与"CI 脚本失败"：
- 编译失败：`error:` / `make[2]: ***` / `FAILED` 等 C/C++ 错误
- 脚本失败：`Unable to process file command 'output'`、`Invalid format`、`set-output`/`GITHUB_OUTPUT` 相关报错

---

### `/report-issue` — 创建 issue 记录

在 `/analyze-failure` 完成后使用。按 `YYYY-MM-DD-NNN` 规则编号，在 `issues/` 下创建 Markdown 文档，记录：问题描述、根本原因分析（含错误日志）、修复方案、对应 patch 文件名、前置 patch（如有）。

```
/report-issue
```

示例：`issues/2026-03-07-001-CachingHostAllocator-HostBlockPool-api-break.md`

---

### `/gen-patch` — 生成 patch 并触发验证

执行内容：

1. 确认本地克隆（`/tmp/ascend_pytorch` 或类似路径）可用
2. 对比新旧 API，按最小改动原则修改受影响源文件
3. `git diff <文件> > patches/NNNN-fix-<模块>-<描述>.patch` 生成 patch
4. 提交 patch + issue 文档并 push
5. `gh workflow run` 触发新一轮构建
6. 观察构建进度判断 patch 是否生效

```
/gen-patch
```

示例：`patches/0002-fix-NPUSHMEMSymmetricMemory-GroupInfo-api-compat.patch`

**常见修复模式：**

| 错误类型 | 修复方式 |
|----------|----------|
| 结构体成员被删除（原用于缓存） | `.cpp` 模块级引入 `static map + mutex` 替代 |
| 成员重命名（如 `blocks` → `blocks_`） | 重命名 Ascend 侧冲突的同名类型，消除 typedef 遮蔽 |
| 基类新增纯虚函数 | 子类 `.hpp` 中补充 override（无对应语义则返回 0） |
| 虚函数签名新增参数 | 更新 override 签名，原逻辑提取为私有方法 |

---

### Patch 生命周期

```
新增 patch → CI 自动应用（git apply --directory=ascend_pytorch）
                │
     apply 成功 ✅ → 构建推进，问题修复
     apply 失败 ❌ → 上游已合入该修复 → 删除 patch 文件并提交
```

> Patch 打在 CI 临时克隆的副本上，**不修改 Ascend/pytorch 官方仓库**。

---

## Workflow 最佳实践

### ccache 配置

```bash
export CC="ccache gcc"
export CXX="ccache g++"
export CCACHE_BASEDIR="${PWD}"
export CCACHE_DIR="${HOME}/.ccache"
export CCACHE_COMPRESS=1
export CCACHE_MAXSIZE=2G
```

### Patch 应用方式

```bash
# 正确方式：使用 --directory 参数
git apply --directory=ascend_pytorch "$patch"

# 错误方式：cd 后再 apply（会导致路径解析错误）
cd ascend_pytorch && git apply "$patch"  # ❌
```

### GITHUB_OUTPUT 写法

```bash
# 正确：单行值
echo "version=${TORCH_VER}" >> $GITHUB_OUTPUT

# 错误：多行值会触发格式错误
echo "logs=$(cat build.log)" >> $GITHUB_OUTPUT  # ❌
```

### 缓存 Key 策略

```yaml
key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-${{ github.run_id }}
restore-keys: |
  ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-
  ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-
```

使用 `cache/restore` + `cache/save` 分离模式，确保 `always()` 时也能保存缓存。

---

## 相关文件

- `AGENTS.md` - 项目级别的 Agent 指导文档
- `.github/workflows/nightly-build.yml` - CI workflow 定义
- `patches/` - 兼容性 patch 存放目录
- `issues/` - 问题分析记录目录
