---
name: gemara-bundle
description: Use when creating or updating a bundle manifest that lists Gemara artifacts for OCI publishing. Also use when the user asks about wiring artifacts for CI, publishing to a registry, or creating a bundles/ file.
---

# Bundle Manifest Creation

Guide users through creating a bundle manifest that lists authored Gemara artifacts for publishing as a signed OCI bundle via `gemara-publish-action`.

## Overview

A bundle manifest is a plain YAML file in `bundles/` that tells the publish workflow which artifacts to package together. It is **not** a Gemara schema type — it is a CI convention consumed by the repository's discover step.

```yaml
layers:
  # Layer 1
  - governance/guidance/<guidance>.yaml
  # Layer 2 (as many as exist)
  - governance/catalogs/<threats>.yaml
  - governance/catalogs/<controls>.yaml
  - governance/catalogs/<capabilities>.yaml
  # Layer 3 (as many as exist)
  - governance/catalogs/<risks>.yaml
  - governance/catalogs/<mappings>.yaml
  - governance/policies/<policy>.yaml
```

Each entry is a plain file path relative to the repository root. A bundle includes **all** artifacts the root depends on, ordered by layer. The last entry must be the root artifact (typically a Policy). Not every layer is required — include only the artifacts that exist in the repo.

## Prerequisite Check

### Step 1: Scan the repo for Gemara artifacts

Search `governance/` (or the repository root) for `.yaml` files containing a `metadata.type` field. Build an inventory:

| Layer | Artifact Type | Found | File |
|-------|--------------|-------|------|
| 1 | GuidanceCatalog | | |
| 2 | ThreatCatalog | | |
| 2 | ControlCatalog | | |
| 2 | CapabilityCatalog | | |
| 3 | RiskCatalog | | |
| 3 | MappingDocument | | |
| 3 | Policy | | |

### Step 2: Check minimum requirements

A publishable bundle requires at minimum a root artifact (Policy or ControlCatalog). If no Layer 3 artifact exists:

> "No Policy artifact was found. A bundle typically needs a root artifact as its final layer. Would you like to:"
>
> 1. Create a Policy first (use the `gemara-policy` skill)
> 2. Use the highest-layer artifact found as the root
> 3. Cancel

### Step 3: Check for existing bundle manifests

Look for `bundles/*.yaml` files. If one already exists for the same use case, offer to update it rather than create a duplicate.

## Authoring Steps

### Step 1: Determine bundle name

The bundle file name becomes the OCI repository suffix. Suggest a name derived from the policy scope:

- `container-security` → `bundles/container-security.yaml`
- `branch-protection` → `bundles/branch-protection.yaml`

Ask the user to confirm the name.

### Step 2: Order layers

List the discovered artifacts in layer order (lowest layer first, root artifact last):

1. Layer 1 artifacts (GuidanceCatalog)
2. Layer 2 artifacts (ThreatCatalog, ControlCatalog, CapabilityCatalog)
3. Layer 3 artifacts (RiskCatalog, MappingDocument, Policy)

Include only artifacts that exist — not every layer is required. Present the proposed ordering for confirmation:

> "Here's the layer ordering for your bundle:"
>
> ```yaml
> layers:
>   - governance/guidance/essv11-guidance.yaml
>   - governance/catalogs/essv11-threats.yaml
>   - governance/catalogs/essv11-controls.yaml
>   - governance/catalogs/essv11-risks.yaml
>   - governance/policies/essv11-policy.yaml
> ```
>
> "The last entry (Policy) will be used as the root artifact for publishing. Confirm?"

### Step 3: Verify file existence

For each path in the proposed manifest, confirm the file exists on disk. If any path does not resolve, stop and ask the user to correct it before writing the file.

### Step 4: Write the bundle manifest

Write the confirmed YAML to `bundles/<name>.yaml`.

## Format Rules

- **Plain file paths only** — each entry is a string, not an object. Never use `- path: ...` or `- file: ...` or any key-value format.
- **Relative to repo root** — paths start from the repository root (e.g., `governance/...`), not from the `bundles/` directory.
- **Last entry is root** — the publish workflow's discover step extracts the last entry as the root artifact to pass to `gemara-publish-action`.
- **No extra fields** — the manifest contains only a `layers` key with a list of strings. No `metadata`, `version`, or other keys.

## After Creation

Once the bundle manifest is committed and merged:

1. The repository's publish workflow discovers it in `bundles/`
2. The last layer path is passed as the `file` input to `gemara-publish-action`
3. The action assembles the artifact, pushes to the OCI registry, signs with cosign, and optionally promotes to a second registry
4. The signed bundle is available at `ghcr.io/<org>/<repo>/<bundle-name>:latest`

If the repository does not yet have a publish workflow, direct the user to the `gemara-publish-action` README for setup.
