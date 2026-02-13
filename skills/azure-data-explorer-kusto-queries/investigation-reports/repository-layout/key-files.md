# Key Files in dataexplorer-docs Repository

## Summary

This document catalogs the key files in the dataexplorer-docs repository, organized by category. The repository follows Microsoft Docs Open Publishing System (OPS) conventions.

---

## 1. Index Files

Index files serve as landing pages and entry points for documentation sections.

### YAML Landing Pages (index.yml)

| File Path | Description |
|-----------|-------------|
| `data-explorer/index.yml` | Main landing page for Azure Data Explorer documentation with cards for getting started, training, and key features |
| `data-explorer/kusto/index.yml` | Landing page for Kusto Query Language (KQL) documentation |

### Markdown Index Pages (index.md)

| File Path | Description |
|-----------|-------------|
| `data-explorer/kusto/access-control/index.md` | Access control overview |
| `data-explorer/kusto/api/index.md` | API overview |
| `data-explorer/kusto/api/connection-strings/index.md` | Connection strings overview |
| `data-explorer/kusto/api/rest/index.md` | REST API overview |
| `data-explorer/kusto/management/index.md` | Management commands overview |
| `data-explorer/kusto/management/data-export/index.md` | Data export overview |
| `data-explorer/kusto/query/index.md` | KQL query reference main index |
| `data-explorer/kusto/query/functions/index.md` | Functions overview |
| `data-explorer/kusto/query/scalar-data-types/index.md` | Scalar data types overview |
| `data-explorer/kusto/query/schema-entities/index.md` | Schema entities overview |

---

## 2. TOC Files (Table of Contents)

TOC files define navigation structure in YAML format.

### Main TOC Files

| File Path | Description |
|-----------|-------------|
| `data-explorer/toc.yml` | Main TOC for Azure Data Explorer docs (~500+ items) |
| `data-explorer/kusto/toc.yml` | Main TOC for Kusto/KQL documentation |

### Section-Specific TOC Files

| File Path | Description |
|-----------|-------------|
| `data-explorer/kusto/query/toc.yml` | TOC for KQL query language reference |
| `data-explorer/kusto/management/toc.yml` | TOC for management commands |
| `data-explorer/kusto/api/toc.yml` | TOC for API documentation |
| `data-explorer/kusto/functions-library/toc.yml` | TOC for user-defined functions library |

### Breadcrumb TOC Files

| File Path | Description |
|-----------|-------------|
| `data-explorer/breadcrumb/toc.yml` | Breadcrumb navigation for main docs |
| `data-explorer/kusto/breadcrumb/toc.yml` | Breadcrumb navigation for Kusto docs |

### Duplicate TOC Files (kusto-tocs directory)

| File Path | Description |
|-----------|-------------|
| `data-explorer/kusto-tocs/api/toc.yml` | Alternate TOC reference for API |
| `data-explorer/kusto-tocs/management/toc.yml` | Alternate TOC reference for management |
| `data-explorer/kusto-tocs/query/toc.yml` | Alternate TOC reference for query |

---

## 3. Configuration Files

### Open Publishing System (OPS) Configuration

| File Path | Description |
|-----------|-------------|
| `.openpublishing.publish.config.json` | Master OPS configuration defining two docsets: `dataexplorer-docs` and `kusto-docs` |
| `data-explorer/.openpublishing.redirection.json` | URL redirections for moved/renamed content |
| `data-explorer/.openpublishing.redirection.migrated.json` | Migrated redirections |
| `data-explorer/kusto/.openpublishing.redirection.json` | Kusto-specific URL redirections |

### DocFX Configuration

| File Path | Description |
|-----------|-------------|
| `data-explorer/docfx.json` | DocFX build config for main Data Explorer docs - defines content inclusion, global metadata, search scope |
| `data-explorer/kusto/docfx.json` | DocFX build config for Kusto documentation |

### Development Environment

| File Path | Description |
|-----------|-------------|
| `.devcontainer/devcontainer.json` | VS Code devcontainer configuration |
| `.vscode/settings.json` | VS Code workspace settings |
| `.gitattributes` | Git line ending configuration |
| `.gitignore` | Git ignore rules |

### Quality/Tooling Configuration

| File Path | Description |
|-----------|-------------|
| `.acrolinx-config.edn` | Acrolinx writing quality checker config |
| `.docutune/config/docutune-unattended.json` | DocuTune automation configuration |
| `.whatsnew.json` | What's New content configuration |

---

## 4. Root-Level Documentation Files

| File Path | Description |
|-----------|-------------|
| `README.md` | Repository README with contribution guidelines |
| `LICENSE` | Creative Commons license for documentation content |
| `LICENSE-CODE` | MIT license for code samples |
| `CODEOWNERS` | GitHub code ownership definitions |
| `SECURITY.md` | Security reporting guidelines |
| `ThirdPartyNotices.md` | Third-party attribution notices |

---

## 5. Include Files (Reusable Content)

Include files contain reusable markdown snippets included in multiple documents.

### Main Includes Directory

**Location:** `data-explorer/includes/`

**Notable includes (35 files):**

| File | Description |
|------|-------------|
| `azure-monitor-vs-log-analytics.md` | Comparison content |
| `cloud-shell-try-it.md` | Azure Cloud Shell instructions |
| `create-free-cluster.md` | Free cluster creation steps |
| `data-connector-intro.md` | Data connector introduction |
| `data-explorer-authentication.md` | Authentication guidance |
| `data-explorer-storm-events.md` | Storm events sample data |
| `entra-service-principal.md` | Microsoft Entra service principal setup |
| `managed-identity.md` | Managed identity instructions |
| `mapping-transformations.md` | Data mapping transformations |

### Kusto-Specific Includes

**Location:** `data-explorer/kusto/includes/`

**Notable includes (50+ files):**

| File | Description |
|------|-------------|
| `agg-function-summarize-note.md` | Aggregation function notes |
| `contains-operator-comparison.md` | String operator comparisons |
| `estimation-accuracy.md` | Estimation accuracy disclaimers |
| `help-cluster-note.md` | Help cluster connection info |
| `ingestion-properties.md` | Ingestion properties reference |
| `join-parameters-attributes-hints.md` | Join operator parameters |
| `syntax-conventions-note.md` | KQL syntax conventions |
| `python-plugin-adx.md` | Python plugin instructions |

---

## 6. Media/Image Directories

Media directories contain screenshots, diagrams, and other visual assets.

### Directory Locations

| Directory Path | Content Type |
|----------------|--------------|
| `data-explorer/media/` | Main documentation images |
| `data-explorer/includes/media/` | Include-specific images |
| `data-explorer/kusto/media/` | Kusto overview images |
| `data-explorer/kusto/access-control/media/` | Access control screenshots |
| `data-explorer/kusto/api/media/` | API diagrams |
| `data-explorer/kusto/concepts/media/` | Concept diagrams |
| `data-explorer/kusto/functions-library/media/` | Function visualization examples |
| `data-explorer/kusto/management/media/` | Management UI screenshots |
| `data-explorer/kusto/query/media/` | Query result visualizations |
| `data-explorer/kusto/tools/media/` | Tool interface screenshots |

### Image Format Distribution

- PNG files: Primary format for screenshots
- SVG files: Used for scalable diagrams
- JPG/JPEG: Some photographic content
- GIF: Animated demonstrations

---

## 7. Sample/Example Files

| File Path | Description |
|-----------|-------------|
| `data-explorer/samples/migrate-cluster-to-multiple-availability-zone/configure-zones.json` | Azure zone configuration sample |

---

## 8. Notable File Organization Patterns

### Docset Structure

The repository contains **two separate docsets**:

1. **dataexplorer-docs** - Main Azure Data Explorer documentation
   - Build source: `data-explorer/`
   - Excludes: `kusto/` subdirectory

2. **kusto-docs** - Kusto Query Language reference
   - Build source: `data-explorer/kusto/`
   - Published separately at `/kusto/` URL path

### Content Organization Conventions

1. **Hierarchical TOC**: Each major section has its own `toc.yml`
2. **Index Pages**: Every major directory has an `index.md` or `index.yml`
3. **Colocated Media**: Images stored in `media/` subdirectory alongside content
4. **Include Files**: Reusable snippets in `includes/` directories
5. **Breadcrumb Navigation**: Separate breadcrumb TOC files for navigation context

### Metadata Standards

- Global metadata defined in `docfx.json` files
- Per-file metadata in YAML front matter
- Search scopes: "Azure", "Data Explorer", "Azure Data Explorer", "Kusto"
- Brand: "azure"

### URL Redirections

Extensive redirection system maintains backward compatibility:
- Multiple `.openpublishing.redirection.json` files
- Migrated redirections tracked separately

---

## File Count Summary

| Category | Count |
|----------|-------|
| Index files (index.md) | 10 |
| Index files (index.yml) | 2 |
| TOC files (toc.yml) | 11 |
| Configuration files (.json) | 11 |
| Include files | 85+ |
| Media directories | 10 |
| Root documentation files | 6 |
