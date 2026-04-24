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
  upload_to_release: false
```

## Notes

<!-- Add any special requirements or notes here -->

---

**What happens after you submit:**

1. Workflow will parse the configuration above
2. Build wheels for x86_64 and ARM64 platforms
3. Verify installation on both platforms
4. Upload to Release (if enabled) or Artifacts
5. Comment results on this issue and close it

---

## 参数说明

| 参数 | 说明 | 默认值 | 可选值 |
|------|------|--------|--------|
| `ucm_ref` | UCM 分支、tag 或 commit SHA | `main` | `main`, `develop`, `v0.3.0`, `v0.5.0` 等 |
| `python_version` | Python 版本 | `3.11` | `3.11`, `3.12`, `3.13` |
| `upload_to_release` | 是否上传到 Release | `true` | `true` (永久保存), `false` (30天保留) |

**注意**：可以在 YAML 块中添加 `# 注释` 行，解析时会自动跳过注释行。