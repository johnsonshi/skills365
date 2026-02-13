# skills365

A collection of productivity skills for AI coding agents. These skills provide specialized knowledge and workflows for common enterprise tasks.

## Available Skills

### 1. azure-data-explorer-kusto-queries

Comprehensive guide for Azure Data Explorer (ADX) and Kusto Query Language (KQL). Covers 645+ query functions, 297+ management commands, data ingestion, visualization, time series/ML, and cluster management.

**Install:**
```bash
npx skills add johnsonshi/skills365 --skill azure-data-explorer-kusto-queries
```

**Use when:**
- Writing or optimizing KQL queries
- Setting up data ingestion pipelines
- Creating dashboards and visualizations
- Implementing time series analysis or anomaly detection
- Configuring management commands, policies, or security

### 2. pptx-azure

PowerPoint generation with Azure-themed assets and templates.

**Install:**
```bash
npx skills add johnsonshi/skills365 --skill pptx-azure
```

**Use when:**
- Creating Azure architecture presentations
- Generating slide decks with Azure branding
- Working with PowerPoint files programmatically

### 3. deep-knowledge-skill-creator

Multi-phase investigation workflow that converts documentation repositories into comprehensive skills. Orchestrates systematic analysis through 5 phases: submodule setup, repository layout analysis, feature discovery, in-depth research, and skill generation.

**Install:**
```bash
npx skills add johnsonshi/skills365 --skill deep-knowledge-skill-creator
```

**Use when:**
- Creating skills from documentation repos
- Encoding organizational knowledge into reusable skills
- Systematic analysis of large codebases or documentation sets

## Repository Structure

```
skills365/
├── README.md                              # This file
├── AGENTS.md                              # Agent instructions
├── skills/
│   ├── azure-data-explorer-kusto-queries/ # ADX/KQL skill
│   │   ├── SKILL.md
│   │   ├── feature-area-skill-resources/
│   │   ├── investigation-reports/
│   │   └── dataexplorer-docs/             # Git submodule
│   ├── pptx-azure/                        # PowerPoint skill
│   │   └── SKILL.md
│   └── deep-knowledge-skill-creator/      # Meta-skill for creating skills
│       └── SKILL.md
```

## Known Limitations

### azure-data-explorer-kusto-queries

The following areas have partial coverage and may need enhancement:

- **Geospatial Functions**: 50+ geo functions (geo_distance, geo_point_in_polygon, H3/S2 cells) are documented in the source but not fully encoded in skill resources
- **Graph Query Language**: Graph operators (make-graph, graph-match, graph-shortest-paths) have limited coverage
- **Plugins**: Python, R, and SQL plugins have basic coverage but could be expanded
- **Real-Time Intelligence (Fabric)**: Microsoft Fabric-specific features are not covered

### deep-knowledge-skill-creator

- Designed for documentation repositories; may need adaptation for code-heavy repos
- Agent team coordination requires sufficient API capacity

## Contributing

Skills follow a standard structure with a `SKILL.md` file containing frontmatter (name, description) and content organized by feature area. See `deep-knowledge-skill-creator` for guidance on creating new skills from documentation sources.

## License

MIT
