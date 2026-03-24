# Product Ideation Pipeline

End-to-end workflow from raw idea to published wireframes and presentations using the Titan methodology.

## When to Use This Workflow

- Starting a new product feature or startup idea
- Validating a product concept before architecture/implementation
- Creating visual prototypes for stakeholder review
- Generating investor pitch materials

## Pipeline Overview

```
Raw Idea (one sentence)
    |
    v
/research — Mine Reddit for real pain points
    |
    v
/backlog — Generate validated feature backlog (20-40 features)
    |
    v
/shortlist — Select MVP features using Titan scoring (8.0+ threshold)
    |
    +----> /wireframe — Premium Aurora 2026 dual-theme wireframes
    |
    +----> /sketch-wireframe — Hand-drawn lo-fi sketch wireframes
    |
    v
/review-wireframe — Dual-persona scoring /100 (Elon + Steve)
    |
    +-- >=80: APPROVED -> /publish-wireframes
    |
    +-- 60-79: ITERATE -> /iterate-wireframe -> re-review
    |
    +-- <60: REDESIGN -> back to /wireframe
    |
    v
/publish-wireframes — Deploy to Firebase Hosting
    |
    v
/slides — Create presentation from artifacts
```

## Step-by-Step

### Step 1: Market Research (Optional but Recommended)

```
/research property management apps
```

- Runs `reddit-research` agent
- Mines Reddit threads for real customer frustrations
- Outputs ranked top-10 pain points with behavioral evidence
- Saves to `research/[topic]/pain-points.md`

### Step 2: Generate Feature Backlog

```
/backlog <your product idea>
```
Example: `/backlog An app that helps landlords manage rental properties and tenant communication`

- Runs `idea-to-backlog` agent
- Performs competitive analysis, Five Whys, assumption stress-testing
- Generates 20-40 features classified as Core/Differentiator/Nice-to-Have
- Outputs structured backlog ready for MVP shortlisting

### Step 3: Select MVP Features

```
/shortlist <path to backlog>
```

- Runs `mvp-shortlist` agent using Titan methodology
- Scores each feature on Elon's 5 criteria + Steve's 5 criteria
- Applies Subtraction Game (features are guilty until proven innocent)
- Outputs 2-3 MVP options with screen mapping and kill criteria
- Threshold: combined score >= 8.0 to include in MVP

### Step 4: Generate Wireframes

**Premium (for stakeholder review):**
```
/wireframe <MVP spec or idea>
```

**Lo-fi sketch (for early exploration):**
```
/sketch-wireframe <MVP spec or idea>
```

### Step 5: Review

```
/review-wireframe <path/to/wireframe.html>
```

- Scores /100 using Elon (efficiency) + Steve (delight) personas
- Returns: APPROVED (>=80) / ITERATE (60-79) / REDESIGN (<60)
- Generates Lock Document when approved

### Step 6: Iterate (if score 60-79)

```
/iterate-wireframe <path/to/wireframe.html>
```

- 3-pass system: P0 Critical -> P1 Recommended -> P2 Polish
- Max 3 iteration cycles before escalating to redesign

### Step 7: Publish

```
/publish-wireframes
```

- Discovers all wireframe HTML files
- Regenerates index landing page
- Deploys to Firebase Hosting

### Step 8: Create Presentation

```
/slides <topic or path to artifacts>
```

- Generates animation-rich HTML slide deck
- Supports: new from scratch, PPT conversion, from backlog/wireframe artifacts
- Output: single self-contained HTML file

## Quality Gates

| Phase | Gate |
|-------|------|
| Backlog | >= 20 features with validated pain points |
| MVP | Combined Titan score >= 8.0/10 |
| Wireframe | 100% MVP feature coverage |
| Review | Score >= 80/100 for approval |
| Publish | Lock document exists, no P0 issues |

## Titan Methodology Reference

See `rules/titan-methodology.md` for full scoring rubric.

**Elon's Lens (50%)**: Problem Magnitude (25%), 10x Potential (25%), Technical Feasibility (20%), Execution Speed (15%), Scalability (15%)

**Steve's Lens (50%)**: User Delight (30%), Simplicity (25%), Design Quality (20%), Emotional Connection (15%), Market Positioning (10%)

## Related Workflows

- `docs/workflows/ideation-to-spec.md` — for turning ideas into technical specifications
- `docs/workflows/architecture-design.md` — for system architecture after MVP is defined
- `docs/workflows/feature-angular-spa.md` — for implementing validated features in Angular
- `docs/workflows/feature-flutter-mobile.md` — for implementing validated features in Flutter
