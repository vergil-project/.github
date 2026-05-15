# Vergil-Project Org Profile README — Design Spec

> **Issue:** vergil-project/vergil-tooling#821
>
> **Target file:** `profile/README.md` in `vergil-project/.github`
>
> **What this is:** The public-facing landing page at
> github.com/vergil-project. Marketing-first, not a technical
> spec sheet.

## Goal

Rewrite the vergil-project org profile README from a technical
inventory into a narrative that presents the complete vision:
what VERGIL is, why it exists, how it works at a high level,
and — critically — the Vergil/Mimir adversarial duality that
drives the project's design philosophy.

## Approach

**Problem-first narrative (Approach A).** Lead with the pain
point, build to the solution, introduce the adversary. The
duality is woven throughout rather than isolated in a single
section.

## Sections

### 1. Banner + Opening

Keep the existing `Vergil-Banner.png` and title. The acronym
expansion stays: *Validation Engine for Repository Governance,
Integration & Lifecycle*.

Opening paragraph: one sentence that says what VERGIL is.
Broader than AI — it's a developer platform for enforcing
engineering standards across a multi-repo, multi-language fleet.
AI agent governance is a primary focus but the tooling is
generally useful for repository governance.

Close to the current opening, lightly tweaked for breadth.

### 2. Why This Exists

Problem-first. Three beats:

1. **Standards decay.** Multi-repo engineering organizations
   enforce standards through wikis, onboarding docs, and code
   review. It works until someone skips the linter, a release
   goes out without a security scan, a new contributor doesn't
   know the process exists.

2. **AI makes it urgent.** AI coding agents don't read wikis.
   They bypass sandboxes, auto-merge things they shouldn't,
   hallucinate compliance. The pace of AI-assisted development
   means the gap between "standard exists" and "standard is
   enforced" gets wider, faster.

3. **VERGIL's thesis.** Replace "please follow the process"
   with "the process won't let you skip it." Every standard
   is mechanically enforced at the tooling layer — not
   documented and hoped for.

Frame as a problem for all engineering (humans skip linters
too), with AI as the accelerant that made building VERGIL
urgent. Do not position VERGIL as exclusively for AI-assisted
development.

### 3. How It Works

Brief architectural highlights — overview, not inventory.
Four short paragraphs:

- **Standards as code.** Every engineering standard is
  mechanically enforced. Commit format, branch naming, PR
  linkage, pre-merge validation, security scanning, release
  process — none depend on someone choosing to comply.

- **One system, many languages.** Same validation pipeline,
  same CI workflows, same container-based execution model —
  whether the repo is Python, Go, Ruby, Java, or Rust.
  Extensible to additional languages.

- **Container-based validation.** What passes locally passes
  in CI — same containers, same tools, same results.

- **AI agent governance.** The plugin layer intercepts agent
  actions at the tool-call boundary and enforces the same
  rules that apply to human developers.

Follow with a compact component table (four repos, one-line
descriptions each) and a link to the docs site for the full
architecture. No ASCII art diagram — that detail belongs in
the documentation, not the landing page.

### 4. The Adversary

The signature section. Tone shifts slightly: still
professional, but with an edge. This is where personality
shows.

Core message: every guardrail needs an adversary. VERGIL
builds the walls;
[Mimir](https://github.com/vergils-nemesis) tries to break
them. The adversarial testing identity exercises every denied
path in the permission model — if the tooling catches it, the
guardrail works; if it doesn't, we found a bug before someone
else found an exploit.

The philosophy: confidence without adversarial testing is just
optimism with better marketing.

One or two paragraphs plus a memorable closing line. Vergil
*wants* Mimir to try — that's the whole point.

Include a link to an
[Infrastructure Mindset article](https://the-infrastructure-mindset.ghost.io)
introducing the Vergil/Mimir concept (placeholder — article
is being written). Link to the
[vergils-nemesis org](https://github.com/vergils-nemesis).

### 5. Documentation

Single line linking to the VERGIL docs site:
`https://vergil-project.github.io/vergil-tooling/`

### 6. Contributing

Brief paragraph linking to the
[contributing guidelines](https://github.com/vergil-project/.github/blob/develop/CONTRIBUTING.md).

### 7. Author

One line. Name, GitHub link, blog link. No bio paragraph.

## What Gets Cut

Compared to the current README:

| Removed | Reason |
|---------|--------|
| ASCII architecture diagram | Too detailed for a landing page; belongs in docs |
| "By the numbers" metrics table | Reads like a resume; narrative is stronger without it |
| "What makes this different" section | Best content woven into sections 2-4 |
| "Current state" section | Can be a single phrase in the opening or dropped |
| Detailed component descriptions (4 long paragraphs) | Replaced by compact table; details live in docs |

## Tone

- Professional but not corporate. A working engineer talking
  to other engineers.
- The Mimir introduction is where personality shows — wry,
  confident, a little fun.
- Honest about AI being both powerful and dangerous. No hype,
  no fear-mongering.
- The Vergil and Mimir references should feel like they belong
  in the same universe as the
  [Mimir manifesto](https://github.com/vergils-nemesis) —
  compatible in tone, opposite in perspective.

## Constraints

- Keep the existing banner image (`Vergil-Banner.png`)
- Link to documentation for details; the README is overview
  and highlights only
- Do not include setup instructions (README is what/why;
  docs site is how)
- The Infrastructure Mindset article link is a placeholder
  until the article is published — use a generic link to the
  blog landing page for now
