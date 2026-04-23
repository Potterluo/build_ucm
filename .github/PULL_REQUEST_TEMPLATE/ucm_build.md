---
name: UCM Build Request
about: Request a UCM wheel build
---

## UCM Wheel Build Request

### Build Configuration

```yaml
build_config:
  # UCM branch, tag or commit SHA
  # Examples: main, v0.5.0, feature-branch, commit-sha
  ucm_ref: main
  
  # Python version (3.11, 3.12, or 3.13)
  python_version: 3.11
  
  # Upload wheels to GitHub Release?
  # true = permanent storage (Release)
  # false = temporary storage (Artifacts, 30 days)
  upload_to_release: true
```

### Build Details

**UCM Repository:** https://github.com/ModelEngine-Group/unified-cache-management

**Target UCM Ref:** <!-- Specify the branch/tag -->

**Python Version:** <!-- Specify Python version -->

**Additional Notes:** <!-- Any special requirements or notes -->

---

### What will happen?

1. ✅ Workflow will parse this configuration
2. ✅ Build wheels for x86_64 and ARM64
3. ✅ Verify installation on both platforms
4. ✅ Upload to Artifacts (30 days) or Release (permanent)

### After Build Completes

- Check the workflow run for build logs
- Download wheels from Artifacts or Releases
- A comment will be added to this PR with build results