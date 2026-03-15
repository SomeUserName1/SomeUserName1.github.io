# Homepage Restructure: Founder Positioning

**Date**: 2026-03-15
**Status**: Draft
**Goal**: Restructure the personal website to position Fabian as a deeply technical, scientific founder for an investor/VC/customer audience, without revealing the specific startup idea (formal verification + code synthesis) due to current IBM employment.

## Constraints

- No mention of the startup idea, formal verification, or code synthesis
- No changes to visual design (dark theme, orange accents, Hugo stack)
- No changes to blog content, tags, or structure
- No Calendly or new CTAs — email/LinkedIn/GitHub links stay as-is
- Keep all existing content accessible; restructure, don't delete

## Target Audience

Investors, customers, VCs, and angels. The site should make them think: "deeply technical, scientific founder with range and rigor." When the MVP is ready and the pitch happens, credibility should already be established.

## Changes

### 1. Hero / Tagline

**Current**: "Fabian H. Klopfer — Software Engineer — Neuroscience & Computing Systems"

**New**: "Fabian H. Klopfer — Systems Engineer & Neuroscientist"

Rationale: Drops job-title framing. "Systems Engineer" signals building from the ground up. "Neuroscientist" is specific, striking, and creates a polymath contrast. No employer mentioned.

Update in:
- `hugo.toml` — `params.description`
- `content/_index.md` — hero section

### 2. About Section

**Current**:
> "Software engineer at IBM working on the Db2/BigSQL distributed datalake engine. Dual background in computer science and neuroscience with two master's degrees. Experience spans neural data analysis (electrophysiology and MRI), graph database optimization, deep brain stimulation, and robotics research in Hong Kong. Open source contributor to scikit-learn, SHAP, MONAI, QubesOS, and others. Interested in building systems that process and make sense of complex data."

**New**:
> "Two master's degrees in computer science and neuroscience. Built a graph database storage engine from scratch in C, ran computational pipelines for deep brain stimulation surgery at University Hospital Tuebingen and high field MRI at the Max Planck Institute for Biological Cybernetics, and contributed to open source projects including scikit-learn, SHAP, and MONAI. Currently building distributed datalake systems at IBM. Interested in what happens when formal rigor meets complex systems."

Key changes:
- Leads with credentials, not employer
- Shows concrete achievements instead of listing areas
- Final sentence plants a seed toward formal methods without revealing anything
- Drops Hanson Robotics/Hong Kong and QubesOS to sharpen focus

Update in: `content/_index.md` — about section

### 3. Selected Work (Projects Section)

**Current**: 6 projects listed by repo name and language, no grouping.

**New**: 6 projects in two narrative groups with reframed descriptions.

#### Systems from Scratch

| Project | Language | Description |
|---------|----------|-------------|
| Graph Database Storage Engine | C | Record layout optimization and query algorithms, built from the ground up. 15,000 lines. |
| Generic Blockchain | Rust | Type-generic blockchain with pluggable payloads, async networking, and cryptographic signing. |
| Concept Hierarchy Formation | Python/Java | Automatic type hierarchy inference in property graph databases using conceptual clustering. |

#### Scientific Computing

| Project | Language | Description |
|---------|----------|-------------|
| Brain Microstructure Prediction | Python | Conditional GAN predicting diffusion tensors from bSSFP MRI. 51M parameter model. |
| MEA Analysis Toolbox | Python | Interactive analysis pipeline for 256-electrode neural recordings. Spectral, burst, and network analysis. |
| Data-Driven Biomarkers for DBS | Python | Electrophysiological biomarker discovery for Parkinson's deep brain stimulation using SHAP explainability. 42 patients, 5,122 samples. |

Key changes:
- Grouped by capability category, not flat list
- Human-readable project names replace repo names
- Descriptions emphasize scale and substance (line counts, parameter counts, sample sizes)
- Concept Hierarchy Formation moved to Systems group (closest to formal methods thinking)
- Operating Systems Tutorials removed from featured projects
- Data-Driven Biomarkers for DBS added to Scientific Computing

Each project still links to its GitHub repo and blog post where applicable.

Update in: `content/_index.md` — projects section

### 4. CV Section → Dedicated /cv Page

**Current**: Full CV (education, experience, skills, certifications, publications, open source contributions) embedded on the homepage.

**New**: All CV content moves to a new `/cv` page at `content/cv/_index.md`. Content is unchanged. Homepage navigation adds a "CV" link.

Homepage sections after this change:
1. Hero (name, tagline, photo, links)
2. About
3. Selected Work
4. Footer

Update in:
- `content/_index.md` — remove CV sections
- `content/cv/_index.md` — new page with all CV content
- `layouts/` — may need a CV page template or reuse existing single template
- Navigation links — add CV link alongside Blog, GitHub, LinkedIn, Email

## Files to Modify

| File | Change |
|------|--------|
| `hugo.toml` | Update `params.description` to new tagline |
| `content/_index.md` | Rewrite hero, about, projects; remove CV sections |
| `content/cv/_index.md` | New file with all CV content |
| `layouts/index.html` | Update homepage template to remove CV sections, add project grouping |
| `layouts/_default/baseof.html` or partials | Add CV to navigation if needed |
| `static/css/style.css` | Minor CSS for project group headings if needed |

## What Does NOT Change

- Blog: all posts, tags ("AI Project Summary"), structure, templates
- Visual design: colors, fonts, layout patterns, dark theme
- Contact links: GitHub, LinkedIn, Email — no Calendly
- Footer: as-is
- Deployment: same GitHub Actions workflow
- All existing content remains accessible (CV on /cv, projects via links, blog intact)

## Future Considerations (Not In Scope)

- Calendly link: add when ready to take startup meetings
- Original essay-style blog posts: start with proxy idea (MRI super-resolution) in next 3 months
- Separate blog category for hand-written essays vs. AI summaries
- Shorter blog post intros for investor audience
