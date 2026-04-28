---
name: UCM Wheel Build Request
about: Request a wheel build for unified-cache-management
title: '[Build] UCM wheels for '
labels: build
assignees: ''
---

## Build Configuration

```yaml
build_config:
  ucm_ref: develop
  python_version: "3.12"
  platform: simu
  upload_to_release: false
```

## Notes

<!-- Add any special requirements or notes here -->

---

**What happens after you submit:**

1. Workflow will parse the configuration above
2. Build wheels for specified platform (x86_64 for CUDA, ARM64 for Ascend)
3. Verify installation
4. Upload to Release (if enabled) or Artifacts
5. Comment results on this issue and close it

---

## 参数说明

| 参数 | 说明 | 默认值 | 可选值 |
|------|------|--------|--------|
| `ucm_ref` | UCM 分支、tag 或 commit SHA | `main` | `main`, `develop`, `v0.3.0`, `v0.5.0` 等 |
| `python_version` | Python 版本 | `3.11` | `3.11`, `3.12`, `3.13` |
| `platform` | 目标平台 | `simu` | `simu` (模拟), `cuda` (x86 GPU), `ascend` (ARM NPU) |
| `upload_to_release` | 是否上传到 Release | `true` | `true` (永久保存), `false` (30天保留) |

### Platform 说明

| Platform | 架构 | 适用场景 | 产物 |
|----------|------|----------|------|
| `simu` | x86 + ARM | 纯 CPU 模式，测试用 | 单一 wheel |
| `cuda` | x86_64 | NVIDIA GPU 环境 | wheel (含 CUDA .so) |
| `ascend` | ARM64 | 昇腾 NPU 环境 | wheel + custom_ops |

**注意**：
- CUDA 构建需要 CUDA Toolkit 环境
- Ascend 构建需要 CANN + torch_npu 环境
- 可以在 YAML 块中添加 `# 注释` 行