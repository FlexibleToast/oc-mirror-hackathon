# OpenShift Image Deletion Guide for Disconnected Environments

A step-by-step guide for safely removing old OpenShift images from mirror registries using oc-mirror v2 deletion capabilities.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Pre-Deletion Planning](#pre-deletion-planning)
4. [Step 1: Generate Deletion Plan](#step-1-generate-deletion-plan)
5. [Step 2: Review Deletion Plan](#step-2-review-deletion-plan)
6. [Step 3: Execute Deletion](#step-3-execute-deletion)
7. [Step 4: Post-Deletion Verification](#step-4-post-deletion-verification)
8. [Troubleshooting](#troubleshooting)
9. [References](#references)

## Overview

This guide walks you through safely removing old OpenShift images (**4.19.2 → 4.19.6**) from your mirror registry after upgrading your cluster to **4.19.7**, using our standardized `oc-mirror` deletion workflow.

### What You'll Accomplish
- 🔍 **Pre-deletion validation** of current registry and cluster state
- 📋 **Generate deletion plan** using our standardized script (safe preview)
- 👀 **Review generated plan** to verify what will be deleted
- 🗑️ **Execute controlled deletion** of old OpenShift versions
- ✅ **Post-deletion verification** of registry and cluster health

### Safety First
- ✅ **Two-phase process**: Generate plan → Review → Execute
- ✅ **No accidental deletions**: Must explicitly review and approve
- ✅ **Preserves current versions**: Only removes specified old versions
- ✅ **Rollback ready**: Generated plans serve as audit trail
- ❗ **Workspace Critical**: Must use original mirror workspace (`content/`) - contains essential Cincinnati graph data

## Prerequisites

### Required Access
- Administrative access to the mirror registry (`$(hostname):8443`)
- Push/delete permissions on the target registry
- Valid authentication for registry operations
- Access to our standardized deletion scripts

### Required Tools
- `oc-mirror` v2 (4.19.0 or later)
- `oc` CLI tool (for verification)
- Valid `auth.json` with registry credentials
- Our deletion scripts: `oc-mirror-delete-generate.sh`

### Technical Requirements
- Registry must contain the target images for deletion
- **CRITICAL**: Must use original mirror workspace (`content/`) - contains essential Cincinnati graph data
- Network connectivity to mirror registry
- **Important**: Consistent cache directory usage (same host recommended)

### Cluster State Requirements
- Cluster should be **upgraded past** the versions you plan to delete
- No running clusters should depend on the versions being deleted
- Verify upgrade history to avoid deleting required intermediate versions

## Pre-Deletion Planning

### 1. Verify Current Cluster Version

First, confirm your cluster is running a version **newer** than what you plan to delete:

```bash
# Check current cluster version
oc get clusterversion

# Expected output: 4.19.7 (or later)
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.19.7    True        False         24h     Cluster version is 4.19.7
```

### 2. Inventory Registry Content

Verify the versions you plan to delete actually exist in your registry:

```bash
# Check versions you plan to delete (adjust versions as needed)
oc adm release info $(hostname):8443/openshift/release-images:4.19.2-x86_64 2>/dev/null && echo "✅ 4.19.2 present" || echo "❌ 4.19.2 not found"
oc adm release info $(hostname):8443/openshift/release-images:4.19.3-x86_64 2>/dev/null && echo "✅ 4.19.3 present" || echo "❌ 4.19.3 not found"
oc adm release info $(hostname):8443/openshift/release-images:4.19.6-x86_64 2>/dev/null && echo "✅ 4.19.6 present" || echo "❌ 4.19.6 not found"

# Use this command template to check any version:
# oc adm release info $(hostname):8443/openshift/release-images:VERSION-x86_64
```

**💡 Tip:** Only verify the versions you plan to delete. No need to check your current cluster version - if your cluster is running, those images are obviously accessible. You can also use your web browser to review the inventory.


### 3. Review Deletion Configuration

Verify your deletion configuration targets the correct versions:

```bash
# Review the deletion configuration
cat oc-mirror-master/imageset-delete.yaml
```

Expected configuration:
```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: DeleteImageSetConfiguration
delete:
  platform:
    channels:
    - name: stable-4.19
      minVersion: 4.19.2
      maxVersion: 4.19.6  # Only delete versions older than current 4.19.7
    graph: true
```

## Step 1: Generate Deletion Plan

### 1.1 Navigate to Working Directory

```bash
# Navigate to the oc-mirror working directory
cd oc-mirror-master/
```

### 1.2 Execute Deletion Plan Generation

Run our standardized deletion plan generation script:

```bash
# Generate deletion plan (SAFE - no actual deletions occur)
./oc-mirror-delete-generate.sh
```

**Expected Output:**
```
🗑️ Generating deletion plan for old images...
🎯 Target registry: bastion.sandbox762.opentlc.com:8443
📋 Config: imageset-delete.yaml
📁 Workspace: file://content (original mirror workspace)
⚠️  SAFE MODE: No deletions will be executed

[INFO] 👋 Hello, welcome to oc-mirror
[INFO] ⚙️  setting up the environment for you...
[INFO] 🔀 workflow mode: diskToMirror / delete
[INFO] 🕵  going to discover the necessary images...
[INFO] 📄 Generating delete file...
[INFO] content/working-dir/delete file created
[INFO] 👋 Goodbye, thank you for using oc-mirror

✅ Deletion plan generated successfully!
📄 Plan saved to: content/working-dir/delete/delete-images.yaml
🔍 IMPORTANT: Review the deletion plan before executing!
```

### 1.3 Verify Generation Success

Check that the deletion plan was created in your original workspace:

```bash
# Verify deletion plan was generated
ls -la content/working-dir/delete/
```

You should see:
```
content/working-dir/delete/
├── delete-images.yaml        # Main deletion plan (200KB+ file)
└── delete-imageset-config.yaml  # Configuration used
```

## Step 2: Review Deletion Plan

### 2.1 Examine Generated Deletion Plan

**CRITICAL SAFETY STEP:** Review the generated deletion plan before executing:

```bash
# Review the deletion plan
cat content/working-dir/delete/delete-images.yaml
```

### 2.2 Understand the Deletion Plan Format

The generated plan will contain:
- **Manifests to delete**: Specific image manifests with SHA256 digests
- **Registry locations**: Exact paths in your registry
- **Release versions**: Confirm it targets 4.19.2-4.19.6 only

### 2.3 Verify Target Versions

Look for entries like:
```yaml
# Expected entries in deletion plan
- image: bastion.sandbox762.opentlc.com:8443/openshift/release-images@sha256:...
  # Should target versions 4.19.2, 4.19.3, 4.19.4, 4.19.5, 4.19.6
```

### 2.4 Confirm Preservation of Current Version

**Verify that 4.19.7 (your current cluster version) is NOT listed for deletion:**

```bash
# This should return NO results (4.19.7 should be preserved)
grep -i "4.19.7\|4.19.8\|4.19.9\|4.19.10" content/working-dir/delete/delete-images.yaml || echo "✅ Current versions are preserved"
```

### 2.5 Estimate Deletion Impact

Count the number of images to be deleted:

```bash
# Count images in deletion plan
grep -c "image:" content/working-dir/delete/delete-images.yaml
echo "images will be deleted from the registry"
```

## Step 3: Execute Deletion

### 3.1 Final Pre-Execution Checklist

Before executing the deletion, confirm:

- [ ] **Reviewed deletion plan thoroughly**
- [ ] **Verified current cluster version (4.19.7+)**
- [ ] **Confirmed no running clusters need deleted versions**
- [ ] **Registry backup available (if required by policy)**
- [ ] **Coordinated with relevant teams**

### 3.2 Execute Deletion

After thorough review, execute the deletion using our standardized deletion script:

```bash
# Execute deletion using our safe, guided script
./oc-mirror-delete-execute.sh
```

**Interactive Execution Process:**

The script provides comprehensive safety checks and guidance:

```
🚨 DANGER: About to execute image deletion!
🎯 Target registry: bastion.sandbox762.opentlc.com:8443
📄 Deletion plan: content/working-dir/delete/delete-images.yaml
⚠️  WARNING: This will PERMANENTLY DELETE images from registry!

🔍 Deletion plan found - showing summary:
📊 Images to be deleted: 847
💾 Plan size: 221K

⏰ FINAL CONFIRMATION REQUIRED
This operation will:
  • Delete OpenShift versions 4.19.2 through 4.19.6 from registry
  • Permanently remove image manifests and layers  
  • Keep local cache (119GB) for performance - will NOT be automatically cleaned
  • Free up registry storage space (requires registry GC afterward)
  • Preserve current version 4.19.7 and later

🛑 Press Ctrl+C now to abort, or Enter to proceed with deletion...
```

**What the Script Does:**
- ✅ **Verifies deletion plan exists** before proceeding
- ✅ **Shows deletion summary** (image count, plan size)
- ✅ **Requires explicit confirmation** from user
- ✅ **Explains cache behavior** (preserved for performance)
- ✅ **Provides post-deletion guidance** for next steps

### 3.3 Monitor Deletion Progress

Once you confirm deletion, the script executes the actual `oc mirror delete` command:

**Expected Output:**
```
🗑️ Executing deletion plan...
📊 This may take several minutes depending on registry size

[INFO] 👋 Hello, welcome to oc-mirror
[INFO] ⚙️  setting up the environment for you...
[INFO] 🔀 workflow mode: diskToMirror / delete  
[INFO] 🗑️ deleting images from registry...
[INFO] 📄 Processing deletion plan...
[INFO] 🧹 Removing image manifests...
[INFO] ✅ Deletion completed successfully
[INFO] 👋 Goodbye, thank you for using oc-mirror

✅ Deletion execution completed!
🧹 IMPORTANT: Run registry garbage collection to reclaim storage:
   • For Quay: Log into registry and run GC from admin panel
   • For mirror-registry: sudo podman exec -it quay-app /bin/bash -c 'registry-garbage-collect'
```

### 3.4 Post-Deletion Guidance

The script automatically provides comprehensive next steps and options:

```
💡 Next steps:
   • Verify deleted versions are gone: oc adm release info $(hostname):8443/openshift/release-images:4.19.2-x86_64
   • Check current version still works: oc adm release info $(hostname):8443/openshift/release-images:4.19.7-x86_64  
   • Monitor registry storage usage: df -h /opt/quay/

🗂️  Cache Management Options:
   • Cache size: 119G
   • Keep cache for future operations (recommended for frequent mirroring)
   • Manual cleanup if space needed: rm -rf .cache/
   • Or add --force-cache-delete to this script for automatic cleanup
```

**Key Benefits of Using the Standardized Script:**
- ✅ **Built-in safety checks** prevent accidental execution
- ✅ **Clear progress indication** during deletion process
- ✅ **Automatic guidance** for post-deletion steps
- ✅ **Cache management options** clearly explained
- ✅ **Registry GC reminders** for storage reclamation

## Step 4: Post-Deletion Verification

### 4.1 Verify Deleted Versions Are Gone

Test that the deleted versions are no longer accessible:

```bash
# These should now fail (versions deleted)
oc adm release info $(hostname):8443/openshift/release-images:4.19.2-x86_64 2>&1 | grep -q "deleted or has expired" && echo "✅ 4.19.2 successfully deleted" || echo "❌ 4.19.2 still present"
oc adm release info $(hostname):8443/openshift/release-images:4.19.6-x86_64 2>&1 | grep -q "deleted or has expired" && echo "✅ 4.19.6 successfully deleted" || echo "❌ 4.19.6 still present"

# Alternative: Check all deleted versions at once
echo "🔍 Checking deleted versions:"
for version in 4.19.2 4.19.3 4.19.4 4.19.5 4.19.6; do
    if oc adm release info $(hostname):8443/openshift/release-images:${version}-x86_64 2>&1 | grep -q "deleted or has expired"; then
        echo "✅ ${version} successfully deleted"
    else
        echo "❌ ${version} still present"
    fi
done
```

### 4.2 Verify Current Version Is Preserved

Test that your current cluster version is still available:

```bash
# This should still work (current version preserved)
oc adm release info $(hostname):8443/openshift/release-images:4.19.7-x86_64
```

**Expected Output:**
```
Name:      4.19.7
Digest:    sha256:...
Created:   ...
OS/Arch:   linux/amd64
Manifests: ...
```

### 4.3 Verify Cluster Health

Ensure your running cluster is unaffected:

```bash
# Check cluster operators
oc get co

# All operators should show AVAILABLE=True, PROGRESSING=False, DEGRADED=False
```

### 4.4 Test Registry Functionality

Verify the registry is still functioning properly:

```bash
# Test registry connectivity
curl -k https://$(hostname):8443/v2/

# Should return: {}%
```

### 4.5 Check Storage Reclamation

Optionally check storage space reclaimed:

```bash
# Check available space (should show reclaimed storage)
df -h /opt/quay/
```

## Troubleshooting

### Common Issues

#### 1. NoGraphData Error During Plan Generation
**Error:** `NoGraphData: No graph data found on disk`

**Root Cause:** Delete operations require Cincinnati graph data from the original mirror workspace to understand OpenShift version relationships.

**Solution:**
```bash
# CRITICAL: Always use the original mirror workspace
# ❌ WRONG - separate workspace lacks graph data
oc mirror delete --workspace file://delete-workspace ...

# ✅ CORRECT - use original workspace with metadata
oc mirror delete --workspace file://content ...
```

**Why This Happens:**
- Delete operations need Cincinnati graph data to understand version relationships
- Graph data is stored in `content/working-dir/` from original mirroring
- Separate workspaces lack this essential metadata
- Our standardized script uses the correct workspace automatically

#### 2. Permission Denied During Deletion
**Error:** `403 Forbidden` or permission denied errors

**Solution:**
```bash
# Verify registry authentication
podman login $(hostname):8443

# Check auth file
cat ~/.config/containers/auth.json

# Ensure your account has delete permissions
```

#### 3. Cache Directory Issues
**Error:** Cache-related errors during deletion

**Solution:**
```bash
# Use consistent cache directory
oc mirror delete --cache-dir .cache --delete-yaml-file content/working-dir/delete/delete-images.yaml docker://$(hostname):8443 --v2

# If cache is corrupted, remove and recreate
rm -rf .cache/
```

#### 4. Registry Connectivity Problems
**Error:** Cannot connect to registry

**Solution:**
```bash
# Test registry connectivity
curl -k https://$(hostname):8443/v2/

# Check firewall/network settings
# Verify registry service is running
```

#### 5. Generated Plan Is Empty
**Error:** No images found for deletion

**Possible Causes:**
- Target versions don't exist in registry
- Configuration file has incorrect version ranges
- Registry path issues

**Solution:**
```bash
# Verify images exist before deletion
oc adm release info $(hostname):8443/openshift/release-images:4.19.2-x86_64

# Check configuration file
cat imageset-delete.yaml
```

### Diagnostic Commands

```bash
# Check oc-mirror version
oc-mirror --v2 version

# List deletion plan contents
find content/working-dir/delete/ -name "*.yaml" -exec ls -la {} \;

# Verify registry content
podman search $(hostname):8443/ 2>/dev/null | head -10

# Test registry authentication
podman login --get-login $(hostname):8443

# Check workspace has graph data (critical for delete operations)
ls -la content/working-dir/hold-release/
```

### Recovery Procedures

#### If Deletion Goes Wrong

1. **Stop immediately** if errors occur
2. **Check cluster health**: `oc get co`
3. **Verify current version availability**: `oc adm release info`
4. **Re-mirror if needed**: Use your mirroring scripts to restore content

#### Emergency Recovery

If critical versions were accidentally deleted:

```bash
# Re-mirror required content immediately
cd oc-mirror-master/
./oc-mirror-to-disk.sh      # Mirror to disk
./oc-mirror-from-disk-to-registry.sh  # Upload to registry
```

## References

### Related Documentation
- [Comprehensive Image Deletion Guide](../reference/image-deletion.md) - Detailed technical reference
- [Cluster Upgrade Guide](cluster-upgrade.md) - For upgrading before cleanup
- [oc-mirror Workflow Guide](../setup/oc-mirror-workflow.md) - Standard mirroring procedures

### Command Reference

```bash
# Generate deletion plan (safe preview)
oc mirror delete -c imageset-delete.yaml --generate --workspace file://content docker://$(hostname):8443 --v2 --cache-dir .cache

# Execute deletion using generated plan
oc mirror delete --delete-yaml-file content/working-dir/delete/delete-images.yaml docker://$(hostname):8443 --v2 --cache-dir .cache

# Force cache deletion (if needed)
oc mirror delete --delete-yaml-file content/working-dir/delete/delete-images.yaml --force-cache-delete docker://$(hostname):8443 --v2
```

---

## Quick Start Example

Ready to clean up old images? Here's the complete workflow:

```bash
# 1. Navigate to working directory
cd oc-mirror-master/

# 2. Generate deletion plan (SAFE - no actual deletions)
./oc-mirror-delete-generate.sh

# 3. Review generated plan (CRITICAL SAFETY STEP)
cat content/working-dir/delete/delete-images.yaml

# 4. Execute deletion (only after thorough review)  
./oc-mirror-delete-execute.sh
# The script will:
# - Verify deletion plan exists
# - Show summary (image count, file size)  
# - Require explicit confirmation
# - Execute deletion with progress monitoring
# - Provide post-deletion guidance and next steps

# 5. Follow script's post-deletion guidance automatically provided
# Including verification commands and cache management options
```

**Why Use the Standardized Scripts?**
- ✅ **Built-in safety checks** prevent common mistakes
- ✅ **Interactive confirmations** for dangerous operations  
- ✅ **Comprehensive guidance** throughout the process
- ✅ **Automatic verification** suggestions post-execution
- ✅ **Cache management** options clearly explained

```bash
echo "✅ Image deletion completed safely with standardized scripts!"
```

---

**⚠️ Remember**: Always test deletion operations in non-production environments first and ensure you have proper backups and rollback procedures in place.

**✅ Safety by Design**: This two-phase deletion process provides excellent safety through mandatory review steps.
