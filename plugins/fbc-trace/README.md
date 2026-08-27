# fbc-trace

Trace FBC (File-Based Catalog) operator releases back to their source git revisions to verify whether a specific bug fix is included in a shipped z-stream release.

## Commands

| Command | Description |
|---------|-------------|
| `/fbc-trace:trace` | Trace an FBC operator release to its source git commit and optionally verify fix inclusion |

## Use Cases

- Determine which git commit was used to build a shipped operator version
- Verify whether a specific bug fix commit is included in a z-stream release
- Search the shipped commit history for references to a Jira bug ID
- Understand the build metadata chain from git to the OCP catalog

## Quick Start

```
# Trace an operator to its source commit
/fbc-trace:trace kubernetes-nmstate-operator 4.16.3

# Check if a specific fix commit is included
/fbc-trace:trace kubernetes-nmstate-operator 4.16.3 --commit a1b2c3d

# Search for a bug ID in the shipped history
/fbc-trace:trace sriov-network-operator 4.17.2 --bug OCPBUGS-54321
```

## Prerequisites

For full functionality, the following tools should be available:

- `skopeo` -- inspect container image metadata
- `opm` -- render FBC catalog content
- `git` -- repository operations and ancestry checks
- `jq` -- parse JSON output
- `oc` -- query a running OpenShift cluster (optional)
