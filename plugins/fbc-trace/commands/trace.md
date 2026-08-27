---
description: "Trace an FBC operator release back to its source git revision to verify whether a specific bug fix is included in a shipped z-stream"
argument-hint: "<operator-name> <ocp-version> [--commit <sha>] [--bug <id>]"
example: "/fbc-trace:trace kubernetes-nmstate-operator 4.16.3 --commit abc1234"
---

## Name
fbc-trace:trace

## Synopsis
```
/fbc-trace:trace <operator-name> <ocp-version> [--commit <sha>] [--bug <id>]
```

## Description
The `fbc-trace:trace` command traces an FBC (File-Based Catalog) operator release back to its source git revision. This is essential for answering the question: "Is my bug fix included in the shipped z-stream release?"

FBC operators in OpenShift are built via Konflux pipelines. Each shipped operator image carries metadata (labels and annotations) that record the exact git commit used to build it. This command walks the chain from the shipped catalog entry back to the source repository commit, enabling you to verify whether a specific fix is present.

## Implementation

### Step 1: Identify the FBC fragment image for the operator

Determine the FBC fragment image reference for the given operator and OCP version. The approach depends on what tooling is available:

**Option A -- Using a running cluster with the catalog installed:**

```bash
# Get the catalog source image for the OCP version
oc get catalogsource redhat-operators -n openshift-marketplace -o jsonpath='{.spec.image}'

# Render the catalog and find the operator's bundle image
opm render <catalog-image> | jq -r 'select(.schema == "olm.bundle") | select(.name | startswith("<operator-name>")) | .image'
```

**Option B -- Using `oc-mirror` or `skopeo` against the registry directly:**

```bash
# Inspect the redhat-operator-index for the target OCP version
skopeo inspect --config docker://registry.redhat.io/redhat/redhat-operator-index:v<ocp-major.minor>

# Or use opm to render from the index image
opm render registry.redhat.io/redhat/redhat-operator-index:v<ocp-major.minor> | \
  jq -r 'select(.schema == "olm.bundle") | select(.name | startswith("<operator-name>"))'
```

**Option C -- Using Konflux/RHEC catalog metadata:**

Search the operator's repository for FBC fragment definitions, typically under the `catalog/` or `fbc/` directory, which reference specific bundle image digests.

### Step 2: Inspect the bundle image for git commit metadata

Once you have the bundle image reference, inspect it for build metadata:

```bash
# Inspect the image labels (Konflux sets these during build)
skopeo inspect docker://<bundle-image> | jq '.Labels'

# Key labels to look for:
# - io.openshift.build.commit.id       -- the source git commit SHA
# - io.openshift.build.commit.url      -- link to the commit on GitHub
# - io.openshift.build.source-location -- the source git repository URL
# - vcs-ref                            -- alternative label for git SHA
# - com.redhat.component               -- the component/operator name

# For Konflux-built images, also check annotations:
skopeo inspect docker://<bundle-image> | jq '.Annotations'
# - org.opencontainers.image.revision  -- git commit SHA
# - org.opencontainers.image.source    -- source repository URL
```

### Step 3: Map the commit to the source repository

Using the commit SHA and repository URL from the image labels:

```bash
# Clone the operator's source repository (if not already available)
git clone <source-repo-url> /tmp/<operator-name>
cd /tmp/<operator-name>

# Verify the commit exists and inspect it
git log --oneline -1 <commit-sha>

# Show the full commit details
git show <commit-sha> --stat
```

### Step 4: Determine if a specific fix is included

If the user provided a `--commit` SHA for a bug fix:

```bash
# Check if the fix commit is an ancestor of the shipped commit
git merge-base --is-ancestor <fix-commit-sha> <shipped-commit-sha>
echo $?  # 0 = fix IS included, 1 = fix is NOT included

# If not a direct ancestor, check if the fix was cherry-picked
# by searching for the commit message or Change-Id
git log --oneline <shipped-commit-sha> --grep="<fix-commit-subject>"

# Show the relationship between the two commits
git log --oneline --ancestry-path <fix-commit-sha>..<shipped-commit-sha> 2>/dev/null || \
  echo "No direct ancestry path -- the fix may not be included in this release"
```

If the user provided a `--bug` ID:

```bash
# Search for commits that reference the bug ID in their message
git log --oneline <shipped-commit-sha> --grep="<bug-id>"

# Also search for the bug ID in Jira-style references (e.g., OCPBUGS-12345)
git log --oneline <shipped-commit-sha> --grep="OCPBUGS-<bug-id>"
```

### Step 5: Compile and present the trace report

Present the findings in a structured report:

```
FBC Trace Report
================

Operator:        <operator-name>
OCP Version:     <ocp-version>
Catalog Image:   <catalog-image-ref>
Bundle Image:    <bundle-image-ref>

Source Repository: <git-repo-url>
Shipped Commit:    <commit-sha>
Commit Date:       <date>
Commit Message:    <subject>

Fix Verification:
  Fix Commit:    <fix-commit-sha>
  Included:      YES / NO
  Evidence:      <ancestry check result or cherry-pick match>

If NOT included:
  The fix commit <sha> is NOT an ancestor of the shipped commit.
  Commits after the shipped revision:
    <list of commits between shipped and fix, if fix is newer>

  Next z-stream that may include the fix:
    Check the next minor release or upcoming errata.
```

## Return Value
- A structured trace report showing the chain from the shipped catalog entry to the source git commit
- A clear YES/NO answer on whether a specific fix commit or bug ID is included
- Actionable guidance if the fix is not yet shipped

## Examples

1. **Trace an operator release to its git revision:**
   ```
   /fbc-trace:trace kubernetes-nmstate-operator 4.16.3
   ```
   Returns the source git commit that was used to build the shipped version.

2. **Check if a specific commit is included in a z-stream:**
   ```
   /fbc-trace:trace kubernetes-nmstate-operator 4.16.3 --commit a1b2c3d
   ```
   Returns whether commit `a1b2c3d` is an ancestor of the shipped commit.

3. **Check if a bug fix is included by bug ID:**
   ```
   /fbc-trace:trace sriov-network-operator 4.17.2 --bug OCPBUGS-54321
   ```
   Searches the shipped commit history for references to the bug.

4. **Trace a community operator:**
   ```
   /fbc-trace:trace local-storage-operator 4.18.1
   ```

## Arguments
- **operator-name** (required): The name of the operator to trace (e.g., `kubernetes-nmstate-operator`)
- **ocp-version** (required): The OCP z-stream version to check (e.g., `4.16.3`)
- **--commit** (optional): A specific git commit SHA to check for inclusion
- **--bug** (optional): A Jira bug ID to search for in the commit history

## Prerequisites

The following tools should be available for full functionality:
- `skopeo` -- for inspecting container image metadata
- `opm` -- for rendering FBC catalog content
- `git` -- for repository operations and ancestry checks
- `jq` -- for parsing JSON output
- `oc` -- for querying a running OpenShift cluster (optional)

If some tools are unavailable, the command will adapt its approach and note any limitations in the output.

## Background

### What is FBC?
File-Based Catalog (FBC) is the format used by OLM (Operator Lifecycle Manager) to define operator catalogs in OpenShift 4.11+. Each operator maintains its own catalog fragment that is compiled into the overall catalog index image.

### Why trace to git revisions?
When a bug fix is merged into an operator's repository, it does not immediately appear in a shipped z-stream. The fix must go through the build pipeline (Konflux or CPaaS), be included in a bundle image, and then be incorporated into the catalog index for a specific OCP release. Tracing the chain from the catalog back to the git commit is the definitive way to confirm a fix is shipped.

### Build metadata chain
```
Git Commit --> Konflux Build --> Bundle Image --> FBC Fragment --> Catalog Index --> OCP Release
     ^                              |
     |                              |
     +--- Labels/Annotations ------+
          (commit SHA, repo URL)
```
