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
  # UCM branch, tag or commit SHA
  # Examples: main, develop, v0.5.0, feature-xxx
  ucm_ref: develop

  # Python version: 3.11, 3.12, or 3.13
  python_version: "3.12"

  # Upload wheels to GitHub Release (permanent storage)
  # true = Release (永久保存)
  # false = Artifacts only (30天保留)
  upload_to_release: true
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