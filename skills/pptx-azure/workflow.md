# Presentation Workflow

This document describes the recommended workflow for creating Azure presentations. The process is collaborative and iterative.

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 1: Context Gathering (Doc Co-authoring)                      │
│  ─────────────────────────────────────────────────────────────────  │
│  Use /doc-coauthoring skill to create markdown spec                 │
│  Template: templates/presentation-spec.md                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 2: Validation                                                │
│  ─────────────────────────────────────────────────────────────────  │
│  Review markdown spec against slides/ guides                        │
│  Ensure each slide follows best practices                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 3: Slide-by-Slide Creation                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  Build PPTX incrementally, one slide at a time                      │
│  User reviews and may edit manually between iterations              │
└─────────────────────────────────────────────────────────────────────┘
```

## Phase 1: Context Gathering

**Goal:** Create a complete markdown "spec" of the presentation before touching PowerPoint.

### Step 1: Invoke Doc Co-authoring

Use the `/doc-coauthoring` skill to start the conversation. This skill will:

- Ask clarifying questions about audience, goals, and content
- Help structure the narrative arc
- Iterate back and forth until the spec is complete

### Step 2: Use the Template

The markdown spec should follow [templates/presentation-spec.md](templates/presentation-spec.md). Key elements:

```markdown
## Slide N: [Action Title]

**Type:** [Title | Section header | Content | Architecture | Data/Chart | Comparison | Summary]

**Key Message:** [One sentence takeaway]

**Content:**
- [Verbose content - will be trimmed later]

**Visuals:**

Description: [What the visual shows, key elements, relationships]

~~~mermaid
[Mermaid diagram representing the visual - flowchart, sequence, architecture, etc.]
~~~

**Speaker Notes:**
[Detailed notes on what presenter will say, including:
- Key talking points
- Transitions from previous slide
- Context not shown on slide
- Anticipated questions]
```

### Step 3: Agent Generates Content

**The agent should proactively create:**

1. **Speaker notes** - Based on what the user describes, draft detailed speaker notes. Don't wait for the user to write these.

2. **Mermaid diagrams** - For any visual (architecture diagrams, flowcharts, sequences, data flows), create a Mermaid representation in the spec.

3. **Content structure** - Propose how to organize the user's ideas into slides.

**The user provides the ideas; the agent drafts the details.** Iterate together until both are satisfied.

### Step 4: Be Verbose

At this stage, **more detail is better**. Include:

- All the points you might want to make
- Background context
- Data and sources
- Alternative phrasings
- Detailed speaker notes
- Mermaid diagrams for all visuals

We'll trim and refine in Phase 2.

## Phase 2: Validation

**Goal:** Ensure the markdown spec follows slide design best practices before creating the PPTX.

### Review Against Guides

Run through each slide in the markdown spec and check against:

| Check | Guide |
|-------|-------|
| Does each slide have exactly ONE key message? | [one-idea-per-slide.md](slides/one-idea-per-slide.md) |
| Is the title an action title (states takeaway)? | [action-titles.md](slides/action-titles.md) |
| Is the content concise enough? | [less-is-more.md](slides/less-is-more.md) |
| Does the overall deck tell a story? | [storytelling-structure.md](slides/storytelling-structure.md) |
| If for executives, does it lead with conclusion? | [executive-communication.md](slides/executive-communication.md) |
| For architecture slides, does it follow diagram guidelines? | [architecture/diagrams.md](architecture/diagrams.md) |

### Common Adjustments

- **Split slides** that have multiple key messages
- **Rewrite titles** from topics to action statements
- **Move detail** to speaker notes or appendix
- **Reorder slides** to improve narrative flow
- **Add/remove slides** as needed

### Finalize Markdown Spec

Once all checks pass, the markdown spec is ready for Phase 3.

## Phase 3: Slide-by-Slide Creation

**Goal:** Build the PPTX incrementally, collaborating with the user on each slide.

### Why Slide-by-Slide?

- User can review and adjust as we go
- Catches issues early before building on them
- User may manually edit the PPTX between iterations
- Graphics and layout need visual confirmation
- Allows for creative iteration

### Workflow

```
For each slide in markdown spec:
    1. Read the current state of the PPTX (user may have edited)
    2. Create/update the slide based on markdown spec
    3. Present to user for review
    4. User approves OR provides feedback
    5. If feedback: iterate on this slide
    6. If approved: move to next slide
```

### Important: Always Re-read Before Editing

**Before editing any slide**, always read the current PPTX state:

```
The user may have:
- Manually edited slides in PowerPoint
- Adjusted layouts, fonts, or colors
- Added or removed slides
- Reordered slides
```

Never assume the PPTX is in the same state as when you last touched it.

### Step-by-Step Process

#### 1. Create Empty PPTX

Start with an empty presentation or Azure-branded template.

#### 2. Title Slide First

Create the title slide. Confirm with user before proceeding.

#### 3. One Slide at a Time

For each subsequent slide:

**a) Re-read the PPTX**
```
Always check current state - user may have edited
```

**b) Reference the markdown spec**
```
Get the key message, content, and visual requirements
```

**c) Create/update the slide**
```
- Apply Azure colors from palettes/azure-colors.md
- Use icons from icons/Azure_Public_Service_Icons/Icons/
- Follow visual hierarchy principles
```

**d) Per-slide checklist**

Before presenting to user, verify:

| Element | Check |
|---------|-------|
| Title | Matches action title from spec |
| Key message | Clear and matches spec |
| Content | Trimmed from verbose spec to concise slide |
| Visuals | Included and properly aligned (icons, diagrams, charts) |
| Speaker notes | Added to slide from spec's Speaker Notes field |
| Layout | Follows visual hierarchy (size, contrast, white space) |
| Colors | Uses Azure palette consistently |
| Icons | From official Azure icon set, not stretched/recolored |
| Alignment | Elements are aligned to grid |

**Speaker notes are critical.** They contain what the presenter will say verbally. Always transfer speaker notes from the markdown spec to the actual PPTX slide. Do not skip this step.

**e) Present for review**
```
Show the user what was created
Ask for approval or feedback
Confirm speaker notes are accurate
```

**f) Iterate if needed**
```
User feedback → adjust → re-present
Repeat until approved
```

**g) Move to next slide**
```
Only after user approves current slide
```

#### 4. Handle Additions and Changes

The user may want to:

- **Add slides** - Insert new slides into the markdown spec and PPTX
- **Remove slides** - Delete from both spec and PPTX
- **Reorder slides** - Adjust sequence in both
- **Go back** - Revisit and revise earlier slides

This is expected. The process is collaborative, not linear.

#### 5. Final Review

After all slides are created:

- Review the complete deck end-to-end
- Check transitions and flow
- Verify consistency (fonts, colors, alignment)
- Confirm with user

## Quick Reference

| Phase | Tool/Skill | Output |
|-------|------------|--------|
| 1. Context Gathering | `/doc-coauthoring` | Markdown spec |
| 2. Validation | `slides/*.md` guides | Refined markdown spec |
| 3. Creation | `pptx` skill + this skill | Final PPTX |

## See Also

- [SKILL.md](SKILL.md) - Quick reference for icons, colors, and structure
- [templates/presentation-spec.md](templates/presentation-spec.md) - Markdown template
- [pptx skill](https://github.com/anthropics/skills/blob/main/skills/pptx/SKILL.md) - PowerPoint creation workflows
- [slides/](slides/) - Slide design best practices
