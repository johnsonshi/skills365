---
name: deep-knowledge-skill-creator
description: "Multi-phase workflow for converting documentation repositories into comprehensive skills; use when creating or refreshing skills from docs/codebases, investigating large repos systematically, or turning submodule knowledge into investigation reports plus generated skill content."
---

# Deep Knowledge Skill Creator

Convert documentation repositories into comprehensive, well-structured skills through systematic multi-phase investigation using agent teams.

## When to Use

- Creating skills from documentation repos (e.g., Microsoft docs, API references)
- Encoding organizational knowledge into reusable skills
- Systematic analysis of large codebases or documentation sets
- Refreshing/updating existing knowledge-based skills

## Output Structure

```
skills/<skill-name>/
├── SKILL.md                              # Parent skill (generated in Phase 4)
├── investigation-reports/                # Phases 1-3 output
│   ├── repository-layout/                # Phase 1: Repo structure analysis
│   ├── feature-overview/                 # Phase 2: Feature discovery
│   └── feature-in-depth/                 # Phase 3: Comprehensive research
├── feature-area-skill-resources/         # Phase 4: Skill content by feature
│   ├── <feature-area-1>/
│   │   └── *.md
│   ├── <feature-area-2>/
│   │   └── *.md
│   └── ...
├── submodules/<submodule-1>/             # Git submodule repo 1 containing helpful information
└── submodules/<submodule-1>/             # Git submodule repo 2 containing helpful information
```

## Workflow Overview

```
Phase 0: Submodule Setup
    │
    ▼
Phase 1: Repository Layout Analysis
    │   (Agent team crawls submodule, documents structure)
    ▼
Phase 2: Feature Overview Discovery
    │   (Agent team identifies ALL features/sub-features)
    ▼
Phase 3: Feature In-Depth Research  ◄──────────────────┐
    │   (Agent team creates comprehensive docs          │
    │    per feature area - ITERATES until ALL          │
    │    feature areas from Phase 2 are covered)        │
    │                                                   │
    ├───► Check: All features covered? ─── No ─────────┘
    │                  │
    │                 Yes
    ▼                  │
Phase 4: Skill Generation ◄────────────────────────────┘
        (Generate SKILL.md + feature-area-skill-resources/)
```

**IMPORTANT:** Phase 3 is **iterative**. It must loop until ALL feature areas identified in Phase 2's `feature-taxonomy.md` have corresponding in-depth research. Do not proceed to Phase 4 until every feature area has been investigated.

All phases run **strictly sequentially**. Each phase depends on the previous phase's output.

## Phase 0: Submodule Setup

### Option A: Add New Submodule

```bash
cd skills/<skill-name>
git submodule add <repo-url> <submodule-name>
```

Example:
```bash
cd skills/azure-data-explorer-kusto-queries
git submodule add https://github.com/MicrosoftDocs/dataexplorer-docs.git dataexplorer-docs
```

### Option B: Discover Existing Submodules

```bash
git submodule status
```

If submodules exist, proceed to Phase 1.

### Ask User

If no submodule exists and none was specified:
- Ask user for the documentation repository URL
- Ask user for the target skill name
- Add the submodule

## Phase 1: Repository Layout Analysis

**Goal:** Understand the submodule's structure and produce reference documentation.

**Output:** `investigation-reports/repository-layout/*.md`

### Agent Team Setup

Create a team for Phase 1:

```
TeamCreate: "<skill-name>-phase1-repo-layout"
```

### Tasks to Create

1. **Directory Structure Analysis**
   - Map top-level directories and their purposes
   - Identify documentation vs. code vs. config vs. assets
   - Note any README files and their locations

2. **Content Organization Pattern**
   - How is content organized? (by feature, by audience, by task?)
   - Identify the navigation/TOC structure
   - Map relationships between sections

3. **Key Files Identification**
   - Find index files, TOCs, sitemaps
   - Identify schema files, config files
   - Note any templates or examples

### Output Format

Create markdown files in `investigation-reports/repository-layout/`:

- `directory-structure.md` - Full directory tree with annotations
- `content-organization.md` - How content is organized, navigation patterns
- `key-files.md` - Important files and their purposes

## Phase 2: Feature Overview Discovery

**Goal:** Identify all features, sub-features, and feature areas from the documentation.

**Output:** `investigation-reports/feature-overview/*.md`

**Input:** Phase 1 repository layout reports

### Agent Team Setup

Create a team for Phase 2:

```
TeamCreate: "<skill-name>-phase2-feature-discovery"
```

### Tasks to Create

1. **Feature Area Identification**
   - Read Phase 1 reports to understand repo structure
   - Identify major feature areas/categories
   - Create hierarchical feature taxonomy

2. **Sub-Feature Mapping**
   - For each feature area, identify sub-features
   - Note feature relationships and dependencies
   - Identify which docs cover which features

3. **Feature Prioritization**
   - Identify core vs. advanced features
   - Note commonly referenced features
   - Flag features that need deep investigation

### Output Format

Create markdown files in `investigation-reports/feature-overview/`:

- `feature-taxonomy.md` - Hierarchical list of all features
- `feature-to-docs-mapping.md` - Which docs cover which features
- `priority-features.md` - Core features for Phase 3 deep dive

## Phase 3: Feature In-Depth Research

**Goal:** Create comprehensive documentation for **EVERY** feature area identified in Phase 2.

**Output:** `investigation-reports/feature-in-depth/<feature-area>/*.md`

**Input:** Phase 1 + Phase 2 reports

### CRITICAL: Iterate Until Complete

Phase 3 must **loop until all feature areas are covered**:

```
1. Read feature-taxonomy.md from Phase 2
2. List ALL feature areas identified
3. Check which feature areas already have in-depth research
4. For each MISSING feature area:
   - Spawn agent team
   - Create reference.md, best-practices.md, examples.md
   - Verify output files exist
5. Repeat steps 3-4 until ALL feature areas have coverage
6. Only then proceed to Phase 4
```

**Do NOT proceed to Phase 4 if any feature areas are missing investigation reports.**

### Agent Team Setup

Create a team for Phase 3 (may need multiple iterations):

```
TeamCreate: "<skill-name>-phase3-in-depth"
# or for subsequent iterations:
TeamCreate: "<skill-name>-phase3-in-depth-batch2"
```

Spawn **one agent per feature area** identified in Phase 2.

### Per-Feature Tasks

For each feature area, create tasks for:

1. **Reference Documentation**
   - Syntax, parameters, options
   - API signatures, return types
   - Configuration options

2. **Best Practices**
   - When to use this feature
   - Performance considerations
   - Common patterns and anti-patterns

3. **Examples**
   - Code samples from the docs
   - Real-world usage scenarios
   - Edge cases and gotchas

### Output Format

Create subdirectories in `investigation-reports/feature-in-depth/`:

```
feature-in-depth/
├── <feature-area-1>/
│   ├── reference.md
│   ├── best-practices.md
│   └── examples.md
├── <feature-area-2>/
│   └── ...
```

## Phase 4: Skill Generation

**Goal:** Generate the final skill structure from investigation reports.

**Output:** `SKILL.md` + `feature-area-skill-resources/`

**Input:** All Phase 1-3 reports

### Generate Parent SKILL.md

The parent SKILL.md should include:

1. **Frontmatter**
   - `name`: The skill name
   - `description`: Single-line quoted string with trigger phrases

Use this exact safe pattern:

```yaml
---
name: <skill-name>
description: "Single-line description with trigger phrases and use conditions."
---
```

Frontmatter safety rules:
- Always keep `description` on a single line (no `|` block scalar, no folded multiline values)
- Always quote `description` so YAML-special characters (for example `:`, `#`, `[`, `]`) are treated as plain text
- Pick a quote style that minimizes escaping:
  - Use double quotes if the text contains apostrophes (`'`)
  - Use single quotes if the text contains many double quotes
- If using double quotes, escape inner double quotes as `\"`
- If using single quotes, escape apostrophes by doubling them (`''`)

2. **Overview**
   - What this skill covers (the product/technology)
   - High-level capabilities

3. **Quick Start**
   - Most common use cases
   - Basic examples

4. **Feature Area Navigation**
   - Links to each feature area's resources
   - When to use each feature area

### Generate Feature Area Resources

For each feature area, create a subdirectory in `feature-area-skill-resources/`:

```
feature-area-skill-resources/
├── <feature-area-1>/
│   ├── overview.md          # From feature-in-depth reference + best practices
│   ├── examples.md          # From feature-in-depth examples
│   └── <sub-feature>.md     # Additional files as needed
```

### Quality Checklist

Before completing Phase 4:

- [ ] SKILL.md has proper frontmatter with name and description
- [ ] SKILL.md `description` is a single-line quoted YAML string
- [ ] SKILL.md description includes trigger phrases
- [ ] All feature areas have corresponding resources
- [ ] Cross-references between SKILL.md and feature resources work
- [ ] No duplicate content between SKILL.md and resources

## Agent Team Coordination

### Creating Teams

```python
# Use TeamCreate for each phase
TeamCreate(team_name="<skill-name>-phase<N>-<description>")
```

### Creating Tasks

```python
# Use TaskCreate for work items
TaskCreate(
    subject="<action title>",
    description="<detailed description>",
    activeForm="<present continuous form>"
)
```

### Assigning Work

```python
# Spawn agents and assign to tasks
Task(
    subagent_type="general-purpose",
    team_name="<team-name>",
    name="<agent-name>",
    prompt="<task description>"
)

# Assign task to agent
TaskUpdate(taskId="<id>", owner="<agent-name>")
```

### Phase Transitions

After all tasks in a phase complete:

1. Verify all expected output files exist
2. Send shutdown requests to all phase agents
3. Delete the phase team
4. **For Phase 3 ONLY:** Check if ALL feature areas from Phase 2 taxonomy are covered
   - If not, create another Phase 3 team and continue
   - Repeat until 100% coverage
5. Create the next phase's team

### Phase 3 Completion Checklist

Before proceeding to Phase 4, verify:

```bash
# List all feature areas from taxonomy
grep -E "^### [0-9]+\." investigation-reports/feature-overview/feature-taxonomy.md

# List all completed in-depth research
ls investigation-reports/feature-in-depth/

# These two lists must match!
```

If any feature area is missing from `feature-in-depth/`, spawn another Phase 3 batch.

## Example: ADX Kusto Skill

### Phase 0

```bash
git submodule add https://github.com/MicrosoftDocs/dataexplorer-docs.git \
    skills/azure-data-explorer-kusto-queries/dataexplorer-docs
```

### Phase 1 Output

```
investigation-reports/repository-layout/
├── directory-structure.md    # docs/kusto/, docs/azure-data-explorer/, etc.
├── content-organization.md   # Organized by query language, management, ingestion
└── key-files.md              # TOC.yml, index.md locations
```

### Phase 2 Output

```
investigation-reports/feature-overview/
├── feature-taxonomy.md       # Query language, Operators, Functions, Management...
├── feature-to-docs-mapping.md
└── priority-features.md      # Query language, Operators, Data ingestion
```

### Phase 3 Output (Multiple Iterations)

Phase 3 iterates until ALL 11 feature areas from taxonomy are covered:

**Batch 1 (CORE features):**
```
investigation-reports/feature-in-depth/
├── kql-query-language/
├── data-ingestion/
├── visualization-dashboards/
├── time-series-ml/
└── management-commands/
```

**Batch 2 (Remaining features):**
```
investigation-reports/feature-in-depth/
├── ... (batch 1)
├── api-sdk-integration/
├── security-access-control/
├── cluster-management/
├── business-continuity/
├── integration-services/
├── udf-functions-library/
└── tools-clients/
```

**Verification before Phase 4:**
- Taxonomy lists 11 feature areas
- feature-in-depth/ has 11 subdirectories
- ✅ 100% coverage achieved → proceed to Phase 4

### Phase 4 Output

```
skills/azure-data-explorer-kusto-queries/
├── SKILL.md                  # Parent skill for ADX Kusto
├── feature-area-skill-resources/
│   ├── kql-query-language/
│   ├── data-ingestion/
│   ├── visualization-dashboards/
│   ├── time-series-ml/
│   ├── management-commands/
│   ├── api-sdk-integration/
│   ├── security-access-control/
│   ├── cluster-management/
│   ├── business-continuity/
│   ├── integration-services/
│   ├── udf-functions-library/
│   └── tools-clients/        # ALL 11+ feature areas covered
├── investigation-reports/    # Preserved for reference
└── dataexplorer-docs/        # Submodule
```

## See Also

- [skill-creator skill](https://github.com/anthropics/skills) - For general skill creation guidance
- TeamCreate, TaskCreate, TaskUpdate, SendMessage - Agent coordination tools
