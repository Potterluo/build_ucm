---
name: cross-compile-wheel
description: 跨平台编译 Python wheel 包的完整指南。涵盖 Docker QEMU 模拟、GitHub Actions CI/CD、Issue 触发自动化构建、GPU/NPU 平台（CUDA/Ascend）构建等多种方案。适用于需要构建多架构（x86_64/ARM64）或多 Python 版本 wheel 包的场景。当用户提及：构建 ARM64 wheel、跨平台编译 Python 包、为不同架构打包 Python 项目、编译 pybind11/CMake 扩展、多架构 wheel 构建、Python wheel 交叉编译、GitHub Actions 自动构建 wheel、Issue 触发构建、CUDA wheel 构建、Ascend NPU 构建、GPU 依赖编译时自动触发此 skill。
---

# Python Wheel 跨平台编译指南

## 适用场景

- 在 x86_64 主机上构建 ARM64 (aarch64) wheel 包
- 为多个 Python 版本（3.11/3.12/3.13）构建 wheel
- 包含 C++/CMake/pybind11 扩展的 Python 项目
- **GPU/NPU 平台构建**：CUDA (NVIDIA)、Ascend (华为昇腾)
- Docker 容器化构建流程
- GitHub Actions CI/CD 自动构建
- Issue 触发的自动化构建流程

---

## 方案选择

| 方案 | 适用场景 | 速度 | 稳定性 | 自动化 |
|------|----------|------|--------|--------|
| **Docker QEMU 模拟** | 本地测试、单次构建 | ★★☆☆☆ | ★★★☆☆ | 手动 |
| **GitHub Actions Runner** | CI/CD 简单构建 | ★★★★★ | ★★★★★ | 半自动 |
| **Issue 触发完整流程** | 团队协作、自动化归档 | ★★★★★ | ★★★★★ | 全自动 |
| **CUDA 容器构建** | NVIDIA GPU 项目 | ★★★★★ | ★★★★★ | 半自动 |
| **Ascend ARM64 构建** | 华为昇腾 NPU 项目 | ★★★★☆ | ★★★★☆ | 半自动 |
| **云服务器原生构建** | 大规模生产构建 | ★★★★★ | ★★★★★ | 手动 |

---

## 方案一：Docker QEMU 模拟构建

### 基本原理

Docker 的多架构支持允许在 x86 主机上通过 QEMU 用户态模拟运行 ARM64 容器，实现原生编译。

### 快速启动

```bash
# 1. 启用 Docker 多架构支持
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# 2. 运行 ARM64 构建
docker run --rm --platform linux/arm64 \
    -v $(pwd):/workspace \
    ubuntu:22.04 \
    bash -c "apt update && apt install -y python3.11-dev build-essential && pip install build && cd /workspace && python3.11 -m build --wheel"
```

### 构建脚本模板

创建 `build_arm64.sh`：

```bash
#!/bin/bash
# 通用 ARM64 构建脚本
# 变量：PYTHON_VERSION (如 3.11, 3.12)

set -e

PYTHON_VERSION="${PYTHON_VERSION:-3.11}"

echo "[1] Installing system dependencies..."
apt update
DEBIAN_FRONTEND=noninteractive apt install -y \
    software-properties-common curl git \
    gcc g++ make cmake ninja-build

echo "[2] Adding deadsnakes PPA for Python ${PYTHON_VERSION}..."
add-apt-repository -y ppa:deadsnakes/ppa
apt update

echo "[3] Installing Python ${PYTHON_VERSION}..."
DEBIAN_FRONTEND=noninteractive apt install -y \
    python${PYTHON_VERSION} python${PYTHON_VERSION}-dev python${PYTHON_VERSION}-venv

echo "[4] Installing pip..."
curl -sS https://bootstrap.pypa.io/get-pip.py | python${PYTHON_VERSION}

echo "[5] Installing build tools..."
python${PYTHON_VERSION} -m pip install --break-system-packages \
    build wheel setuptools cmake ninja

echo "[6] Building wheel..."
cd /workspace

# QEMU 稳定性优化：禁用 LTO
export CFLAGS="-O1 -fno-lto"
export CXXFLAGS="-O1 -fno-lto"
export LDFLAGS="-fno-lto"

# 清理并构建
rm -rf build dist/*.whl *.egg-info
python${PYTHON_VERSION} -m build --wheel --no-isolation

echo "Build complete!"
ls -la dist/*.whl
```

### QEMU 稳定性关键优化

| 问题 | 解决方案 | 说明 |
|------|----------|------|
| **并行编译崩溃** | `-j2` 或 `-j1` | QEMU 线程调度不稳定 |
| **LTO 导致崩溃** | `-fno-lto` | LTO 增加内存和复杂度 |
| **编译超时** | 延长 timeout，`--no-isolation` | QEMU 比原生慢 10-50x |
| **内存不足** | 限制 Docker 内存 | ARM64 模拟内存开销大 |

**修改项目并行编译参数**：

```bash
# setup.py 或 CMakeLists.txt 中
sed -i 's/-j8/-j2/g' setup.py        # Python 项目
sed -i 's/-j$(nproc)/-j2/g' build.sh # Shell 脚本
```

### 执行构建

```bash
# Windows PowerShell / Git Bash (需要排除路径转换)
MSYS2_ARG_CONV_EXCL="*" docker run --rm --platform linux/arm64 \
    -v "D:/project:/workspace" \
    -e PYTHON_VERSION=3.11 \
    ubuntu:22.04 \
    bash /workspace/build_arm64.sh

# Linux/Mac
docker run --rm --platform linux/arm64 \
    -v $(pwd):/workspace \
    -e PYTHON_VERSION=3.12 \
    ubuntu:22.04 \
    bash /workspace/build_arm64.sh
```

---

## 方案二：GitHub Actions 简单矩阵构建

### Workflow 配置

```yaml
# .github/workflows/build_wheels.yml
name: Build Wheels

on:
  workflow_dispatch:
    inputs:
      python-version:
        description: 'Python version (e.g., 3.11, 3.12)'
        required: true
        default: '3.11'

jobs:
  build:
    runs-on: ${{ matrix.runner }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - runner: ubuntu-latest
            arch: x86_64
          - runner: ubuntu-24.04-arm
            arch: aarch64

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}

      - name: Install build dependencies
        run: |
          python -m pip install --upgrade pip
          pip install build wheel setuptools cmake ninja

      - name: Build wheel
        run: |
          rm -rf build dist *.egg-info
          python -m build --wheel

      - name: Upload wheel
        uses: actions/upload-artifact@v4
        with:
          name: wheel-${{ matrix.arch }}-py${{ inputs.python-version }}
          path: dist/*.whl
          retention-days: 30
```

### 架构支持

| 架构 | Runner | 方式 | 速度 |
|------|--------|------|------|
| x86_64 | `ubuntu-latest` | 原生编译 | ~2分钟 |
| aarch64 | `ubuntu-24.04-arm` | 原生编译 | ~2分钟 |

**注意**：GitHub ARM64 Runner 原生编译，无 QEMU 模拟，稳定性高。

---

## 方案三：Issue 触发完整自动化流程

### 适用场景

- 团队协作构建，非技术人员也能触发
- 自动归档到 GitHub Release
- 构建结果自动通知（Issue 评论 + 关闭）
- 支持指定目标仓库分支/tag

### Issue 模板

创建 `.github/ISSUE_TEMPLATE/build_request.md`：

```markdown
---
name: Wheel Build Request
about: Request a wheel build for your project
title: '[Build] Wheels for '
labels: build
---

## Build Configuration

```yaml
build_config:
  # Target branch, tag or commit SHA
  target_ref: develop

  # Python version: 3.11, 3.12, or 3.13
  python_version: "3.12"

  # Upload to GitHub Release (permanent)
  # true = Release (永久保存)
  # false = Artifacts only (30天保留)
  upload_to_release: true
```

## Notes

<!-- Add any special requirements here -->

---

**What happens after submit:**
1. Workflow parses configuration
2. Builds wheels for x86_64 and ARM64
3. Verifies installation on both platforms
4. Uploads to Release or Artifacts
5. Comments result and closes this issue
```

### 完整 Workflow

```yaml
# .github/workflows/build_wheels.yml
name: Build Wheels

on:
  issues:
    types: [opened, edited]

  workflow_dispatch:
    inputs:
      target_ref:
        description: 'Branch, tag or commit SHA'
        required: true
        default: 'main'
        type: string
      python_version:
        description: 'Python version'
        required: true
        default: '3.11'
        type: choice
        options: ['3.11', '3.12', '3.13']
      upload_to_release:
        description: 'Upload to Release'
        required: true
        default: true
        type: boolean

permissions:
  contents: write
  issues: write
  actions: read

jobs:
  parse-config:
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' || contains(github.event.issue.body, 'build_config:')
    outputs:
      target_ref: ${{ steps.parse.outputs.target_ref }}
      python_version: ${{ steps.parse.outputs.python_version }}
      upload_to_release: ${{ steps.parse.outputs.upload_to_release }}
      issue_number: ${{ steps.parse.outputs.issue_number }}

    steps:
      - name: Parse configuration
        id: parse
        run: |
          python3 << 'PYEOF'
          import os, re, json

          event_name = os.environ.get('GITHUB_EVENT_NAME', '')

          if event_name == 'workflow_dispatch':
              target_ref = os.environ.get('INPUT_TARGET_REF', 'main')
              python_version = os.environ.get('INPUT_PYTHON_VERSION', '3.11')
              upload_to_release = os.environ.get('INPUT_UPLOAD_TO_RELEASE', 'true')
              issue_number = '0'
          else:
              with open(os.environ.get('GITHUB_EVENT_PATH')) as f:
                  event = json.load(f)
              issue_body = event.get('issue', {}).get('body', '')
              issue_number = str(event.get('issue', {}).get('number', '0'))

              target_ref = 'main'
              python_version = '3.11'
              upload_to_release = 'true'

              config_match = re.search(
                  r'build_config:\s*\n\s*target_ref:\s*(\S+)\s*\n\s*python_version:\s*["\']?(\d+\.\d+)["\']?\s*\n\s*upload_to_release:\s*(\w+)',
                  issue_body, re.MULTILINE)

              if config_match:
                  target_ref = config_match.group(1).strip()
                  python_version = config_match.group(2).strip()
                  upload_to_release = config_match.group(3).strip().lower()

          with open(os.environ.get('GITHUB_OUTPUT', '/dev/null'), 'a') as f:
              f.write(f"target_ref={target_ref}\n")
              f.write(f"python_version={python_version}\n")
              f.write(f"upload_to_release={upload_to_release}\n")
              f.write(f"issue_number={issue_number}\n")
          PYEOF
        env:
          INPUT_TARGET_REF: ${{ inputs.target_ref }}
          INPUT_PYTHON_VERSION: ${{ inputs.python_version }}
          INPUT_UPLOAD_TO_RELEASE: ${{ inputs.upload_to_release }}

  build-wheels:
    needs: parse-config
    runs-on: ${{ matrix.runner }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - runner: ubuntu-latest
            arch: x86_64
          - runner: ubuntu-24.04-arm
            arch: aarch64

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ needs.parse-config.outputs.python_version }}

      - name: Install build tools
        run: pip install build wheel setuptools cmake ninja

      - name: Clone target repository
        run: |
          git clone --depth 1 --branch ${{ needs.parse-config.outputs.target_ref }} \
              $TARGET_REPO target_code
        env:
          TARGET_REPO: ${{ vars.TARGET_REPO_URL }}

      - name: Build wheel
        run: |
          cd target_code
          rm -rf build dist *.egg-info
          python -m build --wheel --no-isolation

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: wheel-${{ matrix.arch }}-py${{ needs.parse-config.outputs.python_version }}
          path: target_code/dist/*.whl
          retention-days: 30

  verify-wheels:
    needs: [parse-config, build-wheels]
    runs-on: ${{ matrix.runner }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - runner: ubuntu-latest
            arch: x86_64
          - runner: ubuntu-24.04-arm
            arch: aarch64

    steps:
      - uses: actions/download-artifact@v4
        with:
          name: wheel-${{ matrix.arch }}-py${{ needs.parse-config.outputs.python_version }}
          path: wheels

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ needs.parse-config.outputs.python_version }}

      - name: Install and verify
        run: |
          pip install wheels/*.whl
          python -c "import your_package; print('✅ Import OK')"

  create-release:
    needs: [parse-config, build-wheels, verify-wheels]
    runs-on: ubuntu-latest
    if: needs.parse-config.outputs.upload_to_release == 'true'

    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: wheel-*-py${{ needs.parse-config.outputs.python_version }}
          path: wheels
          merge-multiple: true

      - name: Generate release tag
        id: tag
        run: |
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          echo "release_tag=wheels-${{ needs.parse-config.outputs.target_ref }}-py${{ needs.parse-config.outputs.python_version }}-${TIMESTAMP}" >> $GITHUB_OUTPUT

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.tag.outputs.release_tag }}
          name: Wheels - ${{ needs.parse-config.outputs.target_ref }} (Python ${{ needs.parse-config.outputs.python_version }})
          body: |
            ## Wheel Build
            **Target Ref:** ${{ needs.parse-config.outputs.target_ref }}
            **Python Version:** ${{ needs.parse-config.outputs.python_version }}
            All wheels verified to install correctly.
          files: wheels/*.whl
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  report-result:
    needs: [parse-config, build-wheels, verify-wheels, create-release]
    runs-on: ubuntu-latest
    if: needs.parse-config.outputs.issue_number != '0'

    steps:
      - name: Comment and Close Issue
        uses: actions/github-script@v7
        with:
          script: |
            const issueNumber = parseInt('${{ needs.parse-config.outputs.issue_number }}');
            const releaseTag = '${{ needs.create-release.outputs.release_tag }}';

            let body = `## ✅ Wheel Build Completed\n\n`;
            body += `| Parameter | Value |\n|-----------|-------|\n`;
            body += `| Target Ref | ${{ needs.parse-config.outputs.target_ref }} |\n`;
            body += `| Python Version | ${{ needs.parse-config.outputs.python_version }} |\n`;
            body += `| x86_64 | ✅ Built & Verified |\n`;
            body += `| ARM64 | ✅ Built & Verified |\n\n`;

            if (releaseTag) {
              body += `### 📦 Download\n`;
              body += `[${releaseTag}](https://github.com/${context.repo.owner}/${context.repo.repo}/releases/tag/${releaseTag})\n`;
            }

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issueNumber,
              body: body
            });

            await github.rest.issues.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issueNumber,
              state: 'closed'
            });
```

### Workflow 流程图

```
Issue 创建 → 解析配置 → 并行构建(x86+ARM) → 验证安装 → 创建Release → 评论Issue → 关闭Issue
```

### 配置参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `target_ref` | 目标分支/tag/commit | `main` |
| `python_version` | Python 版本 | `3.11` |
| `upload_to_release` | 上传到 Release | `true` |

---

## 方案四：GPU/NPU 平台构建

### 平台差异说明

| Platform | 架构 | 运行时依赖 | 适用场景 |
|----------|------|-----------|----------|
| `simu` | x86 + ARM | 无 | 纯 CPU 模式，测试用 |
| `cuda` | x86_64 | CUDA Toolkit 12.x | NVIDIA GPU 环境 |
| `ascend` | ARM64 | CANN + torch_npu | 华为昇腾 NPU 环境 |

### CUDA 构建（NVIDIA GPU）

#### 本地 Docker 构建

```bash
# 使用 NVIDIA CUDA 容器
docker run --rm --gpus all \
    -v $(pwd):/workspace \
    nvidia/cuda:12.4-devel-ubuntu22.04 \
    bash -c "
        apt update && apt install -y python3.12-dev python3-pip
        pip install build wheel setuptools cmake ninja pybind11
        cd /workspace
        export PLATFORM=cuda
        export CUDA_HOME=/usr/local/cuda
        python3.12 -m build --wheel --no-isolation
    "
```

#### GitHub Actions CUDA 构建

```yaml
build-cuda:
  runs-on: ubuntu-latest
  container:
    image: nvidia/cuda:12.4-devel-ubuntu22.04
  steps:
    - uses: actions/checkout@v4

    - name: Install Python and build tools
      run: |
        apt-get update
        apt-get install -y python3.12 python3.12-dev python3-pip
        python3.12 -m pip install build wheel setuptools cmake ninja

    - name: Build wheel
      env:
        PLATFORM: cuda
        CUDA_HOME: /usr/local/cuda
      run: |
        python3.12 -m build --wheel --no-isolation

    - name: Upload wheel
      uses: actions/upload-artifact@v4
      with:
        name: wheel-cuda-x86_64
        path: dist/*.whl
```

### Ascend 构建（华为昇腾 NPU）

#### 环境要求

- **架构**：ARM64 (aarch64)
- **CANN Toolkit**：华为 Ascend Compute Architecture
- **torch_npu**：PyTorch NPU 扩展

#### 本地构建（需预配置环境）

```bash
# 在 ARM64 + CANN 环境中
export PLATFORM=ascend
export ASCEND_ROOT=/usr/local/Ascend/ascend-toolkit/latest

# 构建 wheel
python3.12 -m build --wheel --no-isolation

# 构建产物
# - uc_manager-*.whl (主 wheel)
# - UCM-custom_ops-*.run (自定义算子安装器)
# - ucm_custom_ops-*.whl (自定义算子 wheel)
```

#### GitHub Actions Ascend 构建

```yaml
build-ascend:
  runs-on: ubuntu-24.04-arm  # ARM64 runner
  steps:
    - uses: actions/checkout@v4

    - name: Check CANN environment
      run: |
        if [ -d "/usr/local/Ascend/ascend-toolkit" ]; then
          echo "CANN found"
        else
          echo "⚠️ Fallback to simu mode"
        fi

    - name: Build wheel
      env:
        PLATFORM: ascend
      run: |
        if [ -d "/usr/local/Ascend/ascend-toolkit" ]; then
          export PLATFORM=ascend
          export ASCEND_ROOT=/usr/local/Ascend/ascend-toolkit/latest
        else
          export PLATFORM=simu  # Fallback
        fi
        python -m build --wheel --no-isolation
```

### 平台环境变量对照表

| PLATFORM | CMake RUNTIME_ENVIRONMENT | 编译产物 |
|----------|--------------------------|---------|
| `cuda` | `-DRUNTIME_ENVIRONMENT=cuda` | CUDA .so 内入 wheel |
| `ascend` | `-DRUNTIME_ENVIRONMENT=ascend` | wheel + custom_ops |
| `ascend-a3` | `-DRUNTIME_ENVIRONMENT=ascend` | A3 特定优化 |
| `musa` | `-DRUNTIME_ENVIRONMENT=musa` | MooreThreads GPU |
| `maca` | `-DRUNTIME_ENVIRONMENT=maca` | Cambricon MLU |
| `simu` (默认) | `-DRUNTIME_ENVIRONMENT=simu` | 无 GPU/NPU 内核 |

---

## 验证测试流程

### 1. 检查 wheel 架构

```bash
# 查看 wheel 内的 .so 文件
unzip -l dist/*.whl | grep ".so"

# 验证架构标识
# ARM64: *.cpython-311-aarch64-linux-gnu.so
# x86_64: *.cpython-311-x86_64-linux-gnu.so
```

### 2. 安装验证测试

```bash
# ARM64 测试
MSYS2_ARG_CONV_EXCL="*" docker run --rm --platform linux/arm64 \
    -v "D:/project/dist:/dist" \
    python:3.11-slim \
    bash -c "pip install /dist/*.whl && python -c 'import pkg; print(\"OK\")'"

# x86_64 测试
docker run --rm --platform linux/amd64 \
    -v $(pwd)/dist:/dist \
    python:3.11-slim \
    bash -c "pip install /dist/*.whl && python -c 'import pkg; print(\"OK\")'"
```

### 3. 完整验证脚本

```python
#!/usr/bin/env python
"""Wheel 安装验证脚本"""
import sys, platform, os

def test_wheel(package_name):
    print("=" * 50)
    print("Wheel Installation Validation")
    print("=" * 50)
    print(f"Platform: {platform.machine()}")
    print(f"Python: {platform.python_version()}")

    try:
        mod = __import__(package_name)
        print(f"✅ {package_name} imported OK")
        print(f"Location: {mod.__file__}")
    except ImportError as e:
        print(f"❌ FAILED: {e}")
        sys.exit(1)

    mod_path = os.path.dirname(mod.__file__)
    so_count = sum(1 for r,d,f in os.walk(mod_path) for file in f if file.endswith(".so"))
    print(f"Native libraries: {so_count} .so files")
    print("=" * 50)
    print("VALIDATION PASSED!")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python test_wheel.py <package_name>")
        sys.exit(1)
    test_wheel(sys.argv[1])
```

---

## 常见问题排查

### 1. QEMU Segmentation Fault

```
qemu-aarch64: Segmentation fault
```

**解决方案**：
- 减少并行度 `-j1`（最稳定）
- 禁用 LTO：`-fno-lto`
- 减小优化级别 `-O1`

### 2. Python 版本不可用

```bash
add-apt-repository -y ppa:deadsnakes/ppa
apt update
apt install -y python3.12 python3.12-dev
```

### 3. pip externally-managed-environment

```bash
python3.12 -m pip install --break-system-packages build wheel
```

### 4. CMake 找不到 Python

```bash
apt install -y python3.11-dev
cmake -DPython_INCLUDE_DIRS=/usr/include/python3.11 ..
```

### 5. Release 创建权限错误

```
Resource not accessible by integration
```

**解决方案**：添加 permissions 配置：
```yaml
permissions:
  contents: write
  issues: write
```

### 6. YAML 解析失败（Shell 执行 YAML）

**问题**：Shell 脚本将 YAML 内容当作命令执行。

**解决方案**：使用 Python 解析而非 Shell：
```yaml
run: |
  python3 << 'PYEOF'
  import os, re, json
  # 解析逻辑...
  PYEOF
```

---

## 最佳实践总结

1. **优先使用原生环境**（GitHub Actions、云服务器）
2. **QEMU 仅用于本地测试**，生产构建避免使用
3. **Issue 触发流程适合团队协作**
4. **始终验证安装**：构建成功不代表能正确运行
5. **使用 Python 解析 YAML**：避免 Shell 解析问题
6. **添加权限配置**：Release 和 Issue 操作需要显式权限

---

## 快速命令参考

```bash
# 启用 QEMU 多架构
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# ARM64 快速构建
docker run --rm --platform linux/arm64 -v $(pwd):/workspace ubuntu:22.04 \
    bash -c "apt update && apt install -y python3.11-dev gcc g++ && \
    pip install --break-system-packages build && \
    cd /workspace && python3.11 -m build --wheel"

# 验证架构
docker run --rm --platform linux/arm64 python:3.11-slim uname -m

# 安装测试
docker run --rm --platform linux/arm64 -v $(pwd)/dist:/dist python:3.11-slim \
    bash -c "pip install /dist/*.whl && python -c 'import pkg; print(\"OK\")'"
```