---
name: upload-artifact
description: GitHub Action for uploading build artifacts to BFFless
---

# BFFless Upload Artifact Action

The `bffless/upload-artifact` GitHub Action uploads build artifacts from your CI/CD pipeline to a BFFless instance. It's available on the [GitHub Marketplace](https://github.com/bffless/upload-artifact).

For full documentation, see the [README on GitHub](https://github.com/bffless/upload-artifact).

## Quick Start

Only 3 inputs are required:

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
```

## Common Workflows

### PR Preview with Comment

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # Required for PR comments
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for commit timestamp

      - name: Build
        run: npm run build

      - uses: bffless/upload-artifact@v1
        with:
          path: dist
          api-url: ${{ vars.ASSET_HOST_URL }}
          api-key: ${{ secrets.ASSET_HOST_KEY }}
          alias: preview
          description: 'PR #${{ github.event.pull_request.number }} preview'
          pr-comment: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Production Deploy

```yaml
- uses: bffless/upload-artifact@v1
  id: deploy
  with:
    path: dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
    tags: ${{ needs.release.outputs.version }}
    description: 'Release ${{ needs.release.outputs.version }}'

- run: echo "Deployed to ${{ steps.deploy.outputs.sha-url }}"
```

### Multiple Artifacts (Same Workflow)

You can upload multiple artifacts in the same workflow:

```yaml
- name: Upload frontend
  uses: bffless/upload-artifact@v1
  with:
    path: apps/frontend/dist
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
    base-path: /dist

- name: Upload skills
  uses: bffless/upload-artifact@v1
  with:
    path: .bffless
    api-url: ${{ vars.ASSET_HOST_URL }}
    api-key: ${{ secrets.ASSET_HOST_KEY }}
    alias: production
```

## All Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `path` | **yes** | -- | Build directory to upload |
| `api-url` | **yes** | -- | BFFless platform URL |
| `api-key` | **yes** | -- | API key for authentication |
| `repository` | no | `github.repository` | Repository in `owner/repo` format |
| `commit-sha` | no | auto | Git commit SHA |
| `branch` | no | auto | Branch name |
| `is-public` | no | `'true'` | Public visibility |
| `alias` | no | -- | Deployment alias (e.g., `production`) |
| `base-path` | no | `/<path>` | Path prefix in zip |
| `committed-at` | no | auto | ISO 8601 commit timestamp |
| `description` | no | -- | Human-readable description |
| `proxy-rule-set-name` | no | -- | Proxy rule set name |
| `proxy-rule-set-id` | no | -- | Proxy rule set ID |
| `tags` | no | -- | Comma-separated tags |
| `summary` | no | `'true'` | Write GitHub Step Summary |
| `summary-title` | no | `'Deployment Summary'` | Summary heading |
| `working-directory` | no | `'.'` | Working directory |
| `pr-comment` | no | `'false'` | Post comment on PR |
| `comment-header` | no | `'🚀 BFFLESS Deployment'` | PR comment header |
| `github-token` | no | `GITHUB_TOKEN` | Token for PR comments |

## Outputs

| Output | Description |
|--------|-------------|
| `deployment-url` | Primary URL (SHA-based) |
| `sha-url` | Immutable SHA-based URL |
| `alias-url` | Alias-based URL |
| `preview-url` | Preview URL (if basePath provided) |
| `branch-url` | Branch-based URL |
| `deployment-id` | API deployment ID |
| `file-count` | Number of files uploaded |
| `total-size` | Total bytes |
| `response` | Raw JSON response |

## How It Works

1. **Validates** the build directory exists
2. **Zips** the directory preserving structure
3. **Uploads** via multipart POST to `/api/deployments/zip`
4. **Sets outputs** from API response
5. **Writes Step Summary** with deployment table
6. **Posts PR comment** (if enabled)
7. **Cleans up** temporary files

## Auto-Detection

The action automatically detects:

- **Repository**: from `github.repository`
- **Commit SHA**: PR head SHA or push SHA
- **Branch**: PR head ref or push ref
- **Committed At**: via `git log` (requires `fetch-depth: 0`)
- **Base Path**: derived from `path` input as `/<path>`

## PR Comments

When `pr-comment: true`, the action posts a formatted comment:

> ## 🚀 BFFLESS Deployment
>
> **Alias:** `preview`
>
> | Property    | Value                              |
> | ----------- | ---------------------------------- |
> | **Preview** | [example.com](https://example.com) |
> | **Commit**  | `abc1234`                          |
> | **Files**   | 42                                 |
> | **Size**    | 1.2 MB                             |

Comments are automatically updated on subsequent pushes (no duplicates).

## Troubleshooting

### Missing commit timestamp

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0 # Full history needed for git log
```

### PR comment permission denied

```yaml
permissions:
  pull-requests: write # Required for comments
```

### Custom base path

If your app expects to be served from a subdirectory:

```yaml
- uses: bffless/upload-artifact@v1
  with:
    path: build
    base-path: /docs/build # Served at /docs/build/*
```
