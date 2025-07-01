# Python UV Repository Publish

![Latest Release](https://img.shields.io/github/v/release/p6m-actions/python-uv-repository-publish?style=flat-square&label=Latest%20Release&color=blue)

## Description

A GitHub Action that builds and publishes Python packages to PyPI-compliant repositories using UV. This action works in conjunction with the `python-uv-repository-login` action to authenticate with and publish to multiple Python package repositories including PyPI, TestPyPI, and private repositories.

### Key Features

- **Multi-project support**: Automatically discovers publishable projects or accepts specific project paths
- **Multiple repository publishing**: Publishes to one or more configured PyPI-compliant repositories
- **Smart project discovery**: Identifies publishable projects based on `pyproject.toml` configuration
- **Version management**: Supports custom package versioning or uses project-defined versions
- **UV integration**: Uses UV for fast, reliable building and publishing
- **Dry run capability**: Test publishing workflow without actually publishing packages
- **Duplicate handling**: Configurable behavior for handling existing package versions
- **Comprehensive logging**: Detailed output with timestamps and status indicators
- **Python compatibility**: Works with Python 3.7+ with automatic fallbacks

## Usage

This action should be used after setting up UV and configuring repository authentication with `python-uv-repository-login`.

```yaml
- name: Setup UV
  uses: p6m-actions/python-uv-setup@v1
  with:
    python-version: '3.11'

- name: Login to PyPI repositories
  uses: p6m-actions/python-uv-repository-login@v1
  with:
    credentials: |
      pypi=${{ secrets.PYPI_TOKEN }}
      testpypi=${{ secrets.TEST_PYPI_TOKEN }}

- name: Publish packages
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    project-path: './src'
    package-version: '1.2.3'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `project-path` | Path to the project directory containing pyproject.toml | No | `.` |
| `projects` | Specific project paths to publish (one per line). If not specified, all publishable projects will be published | No | _(empty)_ |
| `package-version` | Version to assign to the packages. If not specified, uses project version | No | _(empty)_ |
| `repositories` | Repository names to publish to (one per line). Must match names configured in python-uv-repository-login. If not specified, publishes to all configured repositories | No | _(empty)_ |
| `skip-existing` | Skip publishing if package version already exists | No | `true` |
| `dry-run` | Perform a dry run without actually publishing | No | `false` |
| `verbose` | Enable verbose output | No | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `published-packages` | List of successfully published packages with their versions |
| `published-count` | Number of packages successfully published |
| `skipped-packages` | List of packages that were skipped (e.g., due to duplicates) |

## Examples

### Basic Usage

Publish all publishable projects to all configured repositories:

```yaml
- name: Publish Python packages
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    project-path: './src'
```

### Publish Specific Projects

Publish only specific projects:

```yaml
- name: Publish specific packages
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    projects: |
      ./src/my-library-core
      ./src/my-library-client
    package-version: '2.1.0'
```

### Publish to Specific Repositories

Publish to only specific repositories (must be configured via `python-uv-repository-login`):

```yaml
- name: Publish to TestPyPI only
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    repositories: |
      testpypi
```

### Complete Workflow Example

A complete workflow showing setup, build, test, version bump, and publish:

```yaml
name: Publish Python Packages

on:
  push:
    branches: [main]

permissions:
  contents: write # Required for python-uv-cut-tag

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup UV
        uses: p6m-actions/python-uv-setup@v1
        with:
          python-version: '3.11'
          version: 'latest'

      - name: Install dependencies
        run: uv sync --all-extras --dev

      - name: Run tests
        run: uv run pytest --cov=. --cov-report=xml

      - name: Run linting
        run: uv run ruff check .

      - name: Cut new version tag
        id: cut-tag
        uses: p6m-actions/python-uv-cut-tag@v1
        with:
          version-level: 'patch'

      - name: Login to repositories
        uses: p6m-actions/python-uv-repository-login@v1
        with:
          repository-urls: |
            testpypi=https://test.pypi.org/legacy/
          credentials: |
            pypi=${{ secrets.PYPI_TOKEN }}
            testpypi=${{ secrets.TEST_PYPI_TOKEN }}

      - name: Publish packages
        uses: p6m-actions/python-uv-repository-publish@v1
        with:
          package-version: ${{ steps.cut-tag.outputs.version }}
          skip-existing: true
        id: publish

      - name: Summary
        run: |
          echo "Published ${{ steps.publish.outputs.published-count }} packages"
          echo "Packages: ${{ steps.publish.outputs.published-packages }}"
```

### Dry Run Example

Test the publishing process without actually publishing:

```yaml
- name: Test publish process
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    project-path: './src'
    dry-run: true
    verbose: true
```

### Multi-Project Monorepo Example

Publish multiple packages from a monorepo structure:

```yaml
- name: Publish monorepo packages
  uses: p6m-actions/python-uv-repository-publish@v1
  with:
    projects: |
      ./packages/core
      ./packages/cli
      ./packages/plugins
    package-version: '1.0.0'
    repositories: |
      pypi
      private-repo
```

### Release Workflow with Manual Trigger

A workflow that can be manually triggered with version level selection:

```yaml
name: Release Python Packages

on:
  workflow_dispatch:
    inputs:
      version-level:
        description: "Version level to bump"
        required: true
        default: "patch"
        type: choice
        options:
          - patch
          - minor
          - major
      target-repository:
        description: "Target repository"
        required: true
        default: "testpypi"
        type: choice
        options:
          - testpypi
          - pypi

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup UV
        uses: p6m-actions/python-uv-setup@v1
        with:
          python-version: '3.11'

      - name: Install dependencies and run tests
        run: |
          uv sync --all-extras --dev
          uv run pytest
          uv run ruff check .

      - name: Cut new version
        id: version
        uses: p6m-actions/python-uv-cut-tag@v1
        with:
          version-level: ${{ github.event.inputs.version-level }}

      - name: Login to repository
        uses: p6m-actions/python-uv-repository-login@v1
        with:
          credentials: |
            pypi=${{ secrets.PYPI_TOKEN }}
            testpypi=${{ secrets.TEST_PYPI_TOKEN }}

      - name: Publish to selected repository
        uses: p6m-actions/python-uv-repository-publish@v1
        with:
          repositories: ${{ github.event.inputs.target-repository }}
          package-version: ${{ steps.version.outputs.version }}
```

## Prerequisites

1. **Repository Authentication**: Use `python-uv-repository-login` action first to configure authentication
2. **UV Setup**: Use `python-uv-setup` action to install and configure UV with caching
3. **Project Configuration**: Projects must have valid `pyproject.toml` files with packaging metadata

## Project Configuration

For projects to be automatically discovered, they should have a `pyproject.toml` file with package metadata:

```toml
[project]
name = "my-library-core"
version = "1.0.0"
description = "My library description"
authors = [
    {name = "Your Name", email = "your.name@example.com"}
]
dependencies = [
    "requests>=2.25.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_library"]
```

### Using Hatch Backend

This action works seamlessly with Hatch as the build backend:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.version]
path = "src/my_package/__init__.py"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

### Workspace Configuration

For monorepos with multiple packages, create a workspace configuration:

```toml
# pyproject.toml (root)
[tool.uv.workspace]
members = ["packages/*"]

# packages/core/pyproject.toml
[project]
name = "my-core"
version = "1.0.0"
# ... other metadata

# packages/cli/pyproject.toml
[project]
name = "my-cli"
version = "1.0.0"
# ... other metadata
```

## Integration with Other Actions

This action is designed to work seamlessly with other UV-based actions:

- **`python-uv-setup`**: Sets up UV with caching for faster builds
- **`python-uv-cut-tag`**: Handles version bumping and Git tagging
- **`python-uv-repository-login`**: Manages authentication with package repositories

Use them together for a complete Python package development and publishing workflow.