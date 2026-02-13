# Azure Architecture Diagram Guidelines

Best practices for creating Azure architecture diagrams. Based on [Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/design-diagrams).

## Core Principles

### Use Standard Notations
- Use official Azure icons from `icons/Azure_Public_Service_Icons/Icons/`
- Don't stretch or recolor brand icons arbitrarily
- Don't substitute marketing logos for generic conceptual elements

### Lines and Arrows
- **Always use directional arrows** - lines without arrows are ambiguous
- **Avoid bidirectional arrows** - use single-ended arrows showing flow from client to server
- For request/response, either show two separate flows (preferred) or annotate a single arrow
- Be consistent in how you represent relationships throughout

### Labels
- Label every icon, grouping container, and relationship
- Label lines when relationships aren't immediately obvious
- Use clear, accurate, meaningful labels

### Consistency
- Standardize colors, casing, icon sizes, line weights, line types
- Use same taxonomy across all diagrams in a solution set
- Apply organizational standards if they exist

### Accuracy
- Don't sacrifice accuracy for simplicity
- Don't depict a PaaS service inside a subnet if it's accessed via private endpoint
- Retire diagrams that no longer reflect reality

### Metadata
Include on every diagram:
- Title
- Description
- Last updated date
- Author
- Version

### Accessibility
- Ensure sufficient color contrast
- Don't rely solely on color to distinguish types
- Pair color with pattern consistently

### Legend
- If you use border or line semantics (solid = sync, dashed = async), include a legend

## Diagram Types

### Context Diagram
- Workload as a single black box
- Shows external personas, upstream/downstream systems
- Only high-level communication paths
- Use generic shapes for external systems
- Include explicit boundary

### High-Level System Diagram
- Decomposes workload into main components
- Shows relationships and data flow
- Arrows indicate direction of interactions
- Good for executive/stakeholder communication

### Component Diagram
- Replaces generic blocks with specific technologies
- Visual bill of materials
- Shows exact technologies and their relationships

### Deployment Diagram
- Maps software to infrastructure
- Shows scaling boundaries, deployment units
- Useful for DevOps planning

### Data-Flow Diagram (DFD)
- How data moves, transforms, is stored
- Show sources, sinks, transformation stages
- Include data classification (public, confidential, regulated)
- Indicate batch vs streaming vs real-time

### Network Diagram
- Network infrastructure perspective
- Show network segmentation
- Highlight ingress/egress points
- Consider separate east-west and north-south diagrams

### Sequence Diagram
- Temporal ordering of interactions
- Single use case or scenario
- Shows sync vs async interactions
- Valuable for API design

## Common Mistakes to Avoid

- Lines without directional arrows
- Bidirectional arrows (use two separate arrows instead)
- Unlabeled elements or relationships
- Inaccurate representations (e.g., wrong network topology)
- Missing metadata (title, date, author)
- Overloaded diagrams (layer instead)
- Low contrast or color-only distinctions
- Using unofficial or outdated icons

## Layer, Don't Overload

Provide progressive disclosure:
1. Context diagram (scope boundaries)
2. Container diagram (major components)
3. Component diagram (specific technologies)
4. Sequence diagram (specific use cases)

## See Also

- [Azure Architecture Icons](https://learn.microsoft.com/en-us/azure/architecture/icons/)
- [Microsoft 365 Icons](https://learn.microsoft.com/en-us/microsoft-365/solutions/architecture-icons-templates)
- [Microsoft Entra ID Icons](https://learn.microsoft.com/en-us/entra/architecture/architecture-icons)
- [Microsoft Fabric Icons](https://learn.microsoft.com/en-us/fabric/fundamentals/icons)
- [Microsoft Power Platform Icons](https://learn.microsoft.com/en-us/power-platform/guidance/icons)
