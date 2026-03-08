# AGENTS.md - PyTorch NPU Nightly CI

This repository maintains API compatibility between Ascend/pytorch and PyTorch nightly builds through automated patching.

## Build Commands

### Main Build
```bash
# Build torch_npu wheel (run in ascend_pytorch directory)
python setup.py build bdist_wheel
```

### CI Workflow
```bash
# Trigger nightly build workflow
gh workflow run

# Check latest run status
gh run list

# View failed logs
gh run view --log-failed
```

### Workflow Triggers
The workflow automatically triggers on:
- **Schedule**: Daily at UTC 02:00 (Beijing 10:00)
- **Manual**: Via `gh workflow run` or GitHub Actions UI
- **Push**: When workflow file, patches, or AGENTS.md change

### Build with Ccache
```bash
# Recommended environment variables
export CC="ccache gcc"
export CXX="ccache g++"
export CCACHE_DIR=~/.ccache
export CCACHE_MAXSIZE=2G
```

## Code Style Guidelines

### Patch Creation
- Use unified diff format with `git diff > patches/NNNN-fix-<module>-<description>.patch`
- Number patches sequentially (0001, 0002, etc.)
- Prefix with `fix-` and describe the module and issue
- Apply patches with: `git apply --directory=ascend_pytorch <patch-file>`

### C++ Code Style
- **Types**: PascalCase (e.g., `ExpandableBlockPool`, `BlockPool`)
- **Methods**: camelCase (e.g., `process_events`, `do_process_npu_events`)
- **Variables**: snake_case (e.g., `blocks_pool`, `search_key`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `ACL_ERROR_NONE`)
- **Private members**: trailing underscore (e.g., `pool_`, `size_`)

### API Compatibility Patterns

| Error Type | Fix Strategy |
|------------|--------------|
| Struct member deleted | Add static map + mutex at module level |
| Member renamed (e.g., `blocks` → `blocks_`) | Rename conflicting types to avoid typedef shadowing |
| Base class new pure virtual | Add override in subclass (return 0 if no semantic equivalent) |
| Virtual function signature changed | Update override signature, extract logic to private method |

### Error Handling
- Use `TORCH_INTERNAL_ASSERT(condition, PTA_ERROR(ErrCode::VALUE))` for invariants
- Check ACL errors with `ACL_ERROR_NONE`
- Return size 0 for failure in memory allocation functions

### Comments
- Use C++ `//` for single-line comments
- Add comments explaining API compatibility workarounds
- Mark unused parameters with `/* unused */` or comment

## Workflow Patterns

### Analyzing Build Failures
1. Fetch latest failed run: `gh run list` and `gh run view --log-failed`
2. Filter logs for: `error:`, `make[2]:`, `Traceback`, `FAILED`
3. Read affected source files from Ascend/pytorch
4. Compare with PyTorch nightly headers in `/usr/local/lib/python3.12/dist-packages/torch/include/`
5. Identify API changes and root cause

### Creating Issues
- Use format: `issues/YYYY-MM-DD-NNN-<module>-<description>.md`
- Include: problem description, root cause analysis, error logs, fix strategy, patch filename
- Number issues sequentially per day

### Patch Lifecycle
```
Create patch → CI auto-applies (git apply --directory=ascend_pytorch)
            ↓
   Apply success ✅ → Build progresses, issue fixed
   Apply failure ❌ → Upstream merged fix → Delete patch and commit
```

## Important Notes

- Patches are applied to CI clone only, not Ascend/pytorch upstream
- Build duration increase indicates previous patches are working
- ccache hit rate should improve from ~0ver 16% to ~99% after first build
- Distinguish real compile errors from CI script errors (GITHUB_OUTPUT format issues)
- PyTorch version: CPU nightly from `download.pytorch.org/whl/nightly/cpu`
- CANN dependency: Uses stub libraries from `third_party/acl/libs/build/build_stub.sh`