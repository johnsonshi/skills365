# Azure Data Explorer Documentation Repository - Directory Structure

## Overview

The `dataexplorer-docs` repository is Microsoft's official documentation for Azure Data Explorer (ADX). It follows the Microsoft Docs Open Publishing System (OPS) structure with approximately **1,491 markdown files** and **1,308 image assets**.

## Directory Tree

```
dataexplorer-docs/
|
|-- .acrolinx-config.edn          # Acrolinx content quality tool config
|-- .gitattributes                # Git attributes for file handling
|-- .gitignore                    # Git ignore rules
|-- .openpublishing.publish.config.json  # Microsoft OPS publishing configuration
|-- .whatsnew.json                # What's new tracking for documentation updates
|-- CODEOWNERS                    # GitHub code ownership definitions
|-- LICENSE                       # Creative Commons license for content
|-- LICENSE-CODE                  # MIT license for code samples
|-- README.md                     # Repository overview and contribution guidelines
|-- SECURITY.md                   # Security policy and vulnerability reporting
|-- ThirdPartyNotices.md          # Third-party attributions
|
|-- .devcontainer/                # [CONFIG] VS Code dev container setup
|   `-- devcontainer.json         # Container configuration
|
|-- .docutune/                    # [CONFIG] Docutune quality tool config
|   `-- config/                   # Configuration files
|
|-- .github/                      # [CONFIG] GitHub repository configuration
|   |-- agents/                   # GitHub Copilot agent definitions
|   |-- workflows/                # GitHub Actions CI/CD workflows
|   `-- pull_request_template.md  # PR template for contributions
|
|-- .vscode/                      # [CONFIG] VS Code workspace settings
|   `-- settings.json             # Editor settings for documentation
|
`-- data-explorer/                # [MAIN CONTENT] Primary documentation directory
    |
    |-- .openpublishing.redirection.json          # URL redirect rules (active)
    |-- .openpublishing.redirection.migrated.json # Migrated redirect rules
    |
    |-- breadcrumb/               # [NAVIGATION] Breadcrumb navigation config
    |-- context/                  # [NAVIGATION] Context/metadata definitions
    |-- includes/                 # [SHARED] Reusable content snippets (~37 files)
    |   |-- cross-repo/           # Cross-repository includes
    |   `-- media/                # Media for includes
    |
    |-- media/                    # [ASSETS] Images for top-level docs (~128 subdirs)
    |   |-- adx-dashboards/       # Dashboard screenshots
    |   |-- anomaly-detection/    # ML/analytics visualizations
    |   |-- business-continuity-*/ # DR documentation images
    |   |-- create-cluster-*/     # Cluster setup screenshots
    |   |-- dashboard-*/          # Dashboard feature images
    |   `-- [many more...]        # Topic-specific image directories
    |
    |-- samples/                  # [SAMPLES] Code samples and examples
    |   `-- migrate-cluster-*/    # Migration sample files
    |
    |-- kusto-tocs/               # [NAVIGATION] Table of contents fragments
    |   |-- api/                  # API section TOC
    |   |-- management/           # Management section TOC
    |   `-- query/                # Query section TOC
    |
    |-- kusto/                    # [KUSTO LANGUAGE] Kusto Query Language docs
    |   |
    |   |-- docfx.json            # DocFX configuration
    |   |-- index.yml             # Landing page definition
    |   |-- toc.yml               # Main table of contents
    |   |-- zone-pivot-groups.yml # Zone pivot definitions
    |   |
    |   |-- access-control/       # [DOCS] Security and access control (~6 files)
    |   |   `-- media/            # Access control images
    |   |
    |   |-- api/                  # [DOCS] Client library and API documentation
    |   |   |-- connection-strings/ # Connection string docs
    |   |   |-- get-started/      # Getting started guides
    |   |   |-- golang/           # Go SDK documentation
    |   |   |-- java/             # Java SDK documentation
    |   |   |-- monaco/           # Monaco editor integration
    |   |   |-- netfx/            # .NET SDK documentation (~19 files)
    |   |   |-- node/             # Node.js SDK documentation
    |   |   |-- powershell/       # PowerShell module docs
    |   |   |-- python/           # Python SDK documentation
    |   |   |-- r/                # R SDK documentation
    |   |   |-- rest/             # REST API documentation (~12 files)
    |   |   `-- media/            # API documentation images
    |   |
    |   |-- concepts/             # [DOCS] Core concepts (~15 files)
    |   |   `-- media/            # Concept diagrams
    |   |
    |   |-- functions-library/    # [DOCS] User-defined functions (~71 files)
    |   |   `-- media/            # Function documentation images
    |   |
    |   |-- includes/             # [SHARED] Kusto-specific includes (~46 files)
    |   |   `-- applies-to-version/ # Version applicability snippets
    |   |
    |   |-- management/           # [DOCS] Control commands (~297 files)
    |   |   |-- data-export/      # Export command documentation
    |   |   |-- data-ingestion/   # Ingestion command documentation
    |   |   |-- graph/            # Graph management commands
    |   |   |-- materialized-views/ # Materialized view commands
    |   |   `-- media/            # Management command images
    |   |
    |   |-- query/                # [DOCS] Query operators/functions (~645 files)
    |   |   |-- functions/        # User-defined function docs
    |   |   |-- scalar-data-types/ # Data type documentation
    |   |   |-- schema-entities/  # Schema entity documentation
    |   |   |-- tutorials/        # Query tutorials
    |   |   `-- media/            # Query documentation images
    |   |
    |   |-- tools/                # [DOCS] Kusto tools documentation (~11 files)
    |   |   `-- media/            # Tool screenshots
    |   |
    |   |-- breadcrumb/           # [NAVIGATION] Kusto breadcrumb config
    |   `-- media/                # [ASSETS] Kusto-level images
    |       |-- debug-inline-python/ # Python debugging images
    |       |-- bypass-dns-restrictions/ # Network config images
    |       |-- continuous-export/ # Export feature images
    |       |-- set-timeouts/     # Timeout config images
    |       `-- docs-navigation/  # Navigation screenshots
    |
    `-- [200+ .md files]          # [DOCS] Top-level ADX documentation articles
        |-- data-explorer-overview.md     # Main ADX overview
        |-- create-cluster-database.md    # Cluster creation guide
        |-- azure-data-explorer-dashboards.md # Dashboards feature
        |-- business-continuity-*.md      # DR documentation
        |-- customer-managed-keys.md      # Security features
        `-- [many more...]                # Topic-specific articles
```

## Directory Summary Table

| Directory | Type | Description | File Count (approx) |
|-----------|------|-------------|---------------------|
| `/` (root) | Config | Repository root with licenses, readme, and config files | 11 files |
| `.devcontainer/` | Config | VS Code dev container configuration | 1 file |
| `.docutune/` | Config | Docutune documentation quality tool settings | 1+ files |
| `.github/` | Config | GitHub workflows, agents, and PR templates | 3+ files |
| `.vscode/` | Config | VS Code workspace settings | 1 file |
| `data-explorer/` | Documentation | Main documentation content directory | 200+ articles |
| `data-explorer/breadcrumb/` | Navigation | Breadcrumb navigation definitions | 1 file |
| `data-explorer/context/` | Navigation | Context/metadata definitions | 1 file |
| `data-explorer/includes/` | Shared | Reusable markdown snippets | ~37 files |
| `data-explorer/media/` | Assets | Images and screenshots | ~128 subdirs |
| `data-explorer/samples/` | Samples | Code samples and examples | 1 subdir |
| `data-explorer/kusto-tocs/` | Navigation | Table of contents fragments | 3 subdirs |
| `data-explorer/kusto/` | Documentation | Kusto Query Language documentation | ~1000+ files |
| `data-explorer/kusto/access-control/` | Documentation | Security and RBAC documentation | ~6 files |
| `data-explorer/kusto/api/` | Documentation | SDK and API documentation | ~50 files |
| `data-explorer/kusto/concepts/` | Documentation | Core KQL concepts | ~15 files |
| `data-explorer/kusto/functions-library/` | Documentation | User-defined functions reference | ~71 files |
| `data-explorer/kusto/includes/` | Shared | Kusto-specific includes | ~46 files |
| `data-explorer/kusto/management/` | Documentation | Control commands reference | ~297 files |
| `data-explorer/kusto/query/` | Documentation | Query operators and functions | ~645 files |
| `data-explorer/kusto/tools/` | Documentation | Kusto tooling documentation | ~11 files |

## Content Organization Patterns

### 1. Documentation Hierarchy
- **Top-level** (`data-explorer/`): Azure portal, cluster management, integration docs
- **Language-specific** (`data-explorer/kusto/`): KQL language reference and concepts

### 2. Asset Organization
- Each major topic has a corresponding `media/` subdirectory
- Images are organized by the article they support
- Shared images are in the main `media/` directories

### 3. Navigation Structure
- `toc.yml` files define table of contents
- `breadcrumb/` directories define breadcrumb navigation
- `index.yml` files define landing pages

### 4. Reusable Content
- `includes/` directories contain shared markdown snippets
- `cross-repo/` subdirectory for cross-repository content
- Version applicability snippets in `applies-to-version/`

### 5. Publishing Configuration
- `.openpublishing.*.json` files control Microsoft Docs publishing
- `docfx.json` files configure DocFX static site generation
- Redirect rules handle URL migrations

## Key Files

| File | Purpose |
|------|---------|
| `README.md` | Repository overview and contribution guidelines |
| `CODEOWNERS` | GitHub code ownership for review assignment |
| `.openpublishing.publish.config.json` | Main publishing configuration |
| `data-explorer/kusto/toc.yml` | Main Kusto documentation navigation |
| `data-explorer/kusto/index.yml` | Kusto landing page definition |

## Statistics

- **Total Markdown Files**: ~1,491
- **Total YAML Files**: ~19
- **Total Image Files**: ~1,308
- **Top-level Documentation Articles**: ~200+
- **Kusto Query Functions Documented**: ~645
- **Management Commands Documented**: ~297
