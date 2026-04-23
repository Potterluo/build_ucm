# UCM Wheel Builder

自动化构建 [unified-cache-management](https://github.com/ModelEngine-Group/unified-cache-management.git) 的 Python wheel 包。

## 功能特性

- ✅ **Issue 触发构建** - 创建 Issue 即可请求构建
- ✅ **手动触发构建** - 通过 workflow_dispatch 手动指定参数
- ✅ **多架构支持** - 同时构建 x86_64 和 ARM64 (aarch64)
- ✅ **多 Python 版本** - 支持 Python 3.11/3.12/3.13
- ✅ **原生 ARM64 构建** - 使用 GitHub ARM64 Runner（无需 QEMU 模拟）
- ✅ **自动验证** - 构建后自动测试 wheel 安装
- ✅ **Release 归档** - 自动创建 GitHub Release 并上传 wheel
- ✅ **自动关闭 Issue** - 构建完成后自动评论结果并关闭 Issue

## 使用方法

### 方法一：Issue 触发（推荐）

1. 创建 Issue，选择 "UCM Wheel Build Request" 模板

2. 在配置块中填写参数：

```yaml
build_config:
  ucm_ref: develop        # UCM 分支、tag 或 commit SHA
  python_version: "3.12"  # Python 版本
  upload_to_release: true # 是否上传到 Release（永久保存）
```

3. 提交 Issue，workflow 自动触发构建

4. 构建完成后：
   - 自动评论构建结果到 Issue
   - 自动关闭 Issue
   - Wheel 文件上传到 Release 或 Artifacts

### 方法二：手动触发

1. 进入 Actions 页面
2. 选择 "Build UCM Wheels" workflow
3. 点击 "Run workflow"
4. 填写参数

## 配置参数

| 参数 | 说明 | 默认值 | 可选值 |
|------|------|--------|--------|
| `ucm_ref` | UCM 分支/tag/commit | `main` | `main`, `develop`, `v0.5.0` 等 |
| `python_version` | Python 版本 | `3.11` | `3.11`, `3.12`, `3.13` |
| `upload_to_release` | 上传到 Release | `true` | `true`（永久）, `false`（30天） |

## 构建产物

### GitHub Releases（永久保存）

Release tag 格式：`ucm-{ucm_ref}-py{python_version}-{timestamp}`

包含：
- `uc_manager-*-linux_x86_64.whl` - x86_64 wheel
- `uc_manager-*-linux_aarch64.whl` - ARM64 wheel

### Artifacts（30天保留）

如果 `upload_to_release: false`，wheel 仅保存到 Artifacts。

## 示例

### 构建 develop 分支 + Python 3.12

创建 Issue：

```yaml
build_config:
  ucm_ref: develop
  python_version: "3.12"
  upload_to_release: true
```

结果：
- Release: `ucm-develop-py3.12-20260423-HHMMSS`
- 包含两个 wheel 文件
- Issue 自动评论并关闭

### 构建 v0.5.0 tag + Python 3.11

```yaml
build_config:
  ucm_ref: v0.5.0
  python_version: "3.11"
  upload_to_release: false
```

结果：
- 仅保存到 Artifacts（30天）
- 不创建 Release

## 架构支持

| 架构 | Runner | 方式 | 速度 |
|------|--------|------|------|
| x86_64 | `ubuntu-latest` | 原生编译 | ~2分钟 |
| aarch64 | `ubuntu-24.04-arm` | 原生编译 | ~2分钟 |

**注意**：使用 GitHub ARM64 Runner 原生编译，无 QEMU 模拟，稳定性高。

## Workflow 流程

```
Issue 创建 → 解析配置 → 并行构建(x86+ARM) → 验证安装 → 创建Release → 评论Issue → 关闭Issue
```

## 相关链接

- [UCM 仓库](https://github.com/ModelEngine-Group/unified-cache-management)
- [构建历史](https://github.com/Potterluo/build_ucm/actions)
- [Releases](https://github.com/Potterluo/build_ucm/releases)