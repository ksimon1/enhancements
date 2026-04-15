# VEP #NNNN: AI-Assisted Software Development Lifecycle for KubeVirt Organization

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version:
- This VEP targets beta for version:
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- ~[ ] (R) Target version is explicitly mentioned and approved~
- ~[ ] (R) Graduation criteria filled~

## Overview

This is a meta-VEP proposing a standardized approach to AI-assisted software development
across the KubeVirt organization. The core idea is to create a shared repository
(`kubevirt/ai-sdlc` or similar) containing foundational AI agent guidance — rules,
conventions, workflow patterns, and documentation structures — that individual KubeVirt
projects import and enhance on a per-project basis.

This draws lessons from:
- [OpenAI's Harness Engineering](https://openai.com/index/harness-engineering/) approach to
  agent-first development at scale
- [Harness Skills](https://github.com/harness/harness-skills) repository pattern for
  cross-tool AI agent guidance
- The [kubevirt-tekton-tasks AGENTS.md](https://github.com/kubevirt/kubevirt-tekton-tasks/pull/828)
  effort that demonstrated per-project AI documentation

The term meta-VEP means this VEP is not for a new KubeVirt feature or API change.
It proposes a process and organizational structure for how the KubeVirt community adopts
AI-assisted development practices consistently.

## Motivation

AI coding assistants (Cursor, Claude Code, GitHub Copilot, OpenAI Codex) are increasingly
used by KubeVirt contributors. Today each contributor configures their own AI environment
independently, resulting in:

- **Inconsistent AI behavior** across projects — an agent working on `kubevirt/kubevirt` has
  no awareness of conventions learned in `kubevirt/ssp-operator` or
  `kubevirt/containerized-data-importer`.
- **Duplicated effort** — each project independently writes AGENTS.md, .cursorrules, and
  similar files, often with overlapping content (as noted in
  [kubevirt-tekton-tasks PR #828 review](https://github.com/kubevirt/kubevirt-tekton-tasks/pull/828)).
- **Knowledge rot** — monolithic instruction files become stale, as documented in the
  [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/) experience:
  "A monolithic manual turns into a graveyard of stale rules."
- **No organizational memory** — lessons learned in one project about effective AI prompting,
  code review patterns, or architectural constraints don't propagate to other projects.

A centralized-but-extensible approach allows the community to share proven patterns while
letting each project tailor guidance to its specific needs.

## Goals

- Define a shared repository structure for organization-wide AI development guidance.
- Establish a layered architecture: **shared base rules** that projects import, plus
  **per-project overrides** for project-specific conventions.
- Provide multi-tool support (Cursor, Claude Code, GitHub Copilot, OpenAI Codex) from a
  single source of truth, following the pattern established by
  [harness-skills](https://github.com/harness/harness-skills).
- Standardize AI workflow documentation patterns across KubeVirt projects
  (kubevirt, ssp-operator, containerized-data-importer, common-templates,
  cluster-network-addons-operator, etc.).
- Create a progressive disclosure model: agents start with a small, stable entry point
  (table of contents) and are directed to deeper sources of truth — not overwhelmed with a
  monolithic instruction file.
- Enable mechanical enforcement of documentation freshness through CI linting.

## Non Goals

- Mandating specific AI tools — projects and contributors choose their own editors.
- Replacing human code review — AI assistance is complementary, not a substitute for the
  existing OWNERS-based review process.
- Automating merges or CI decisions — this VEP focuses on developer experience, not CI/CD
  automation.
- Defining per-project rules in this VEP — each project will create its own overlay.

## Definition of Users

- **KubeVirt contributors** who use AI coding assistants for development, code review,
  and documentation.
- **KubeVirt project maintainers** who curate per-project AI guidance and review AI-assisted
  contributions.
- **New contributors** who benefit from AI agents that understand project conventions,
  reducing onboarding friction.
- **SIG leads** who want consistent AI-assisted development practices across their
  SIG's projects.

## User Stories

- As a contributor using Cursor, I want the AI to understand KubeVirt's coding conventions
  (Go style, test patterns, API conventions) without me having to explain them in every prompt.
- As a maintainer of `kubevirt/ssp-operator`, I want to import shared organization-wide rules
  and only maintain the delta specific to my project.
- As a new contributor, I want the AI agent to guide me through the project's build system,
  test requirements, and contribution process based on structured documentation.
- As a SIG lead, I want all projects under my SIG to share consistent AI guidance for
  common patterns (e.g., controller reconciliation, API versioning, e2e testing).
- As a contributor switching between `kubevirt/kubevirt` and
  `kubevirt/containerized-data-importer`, I want the AI to adapt its guidance to each
  project's specifics while maintaining consistent organization-wide conventions.

## Repos

- `kubevirt/ai-sdlc` (new — the shared repository)
- `kubevirt/kubevirt` (consumer — imports shared rules)
- `kubevirt/containerized-data-importer` (consumer)
- `kubevirt/ssp-operator` (consumer)
- `kubevirt/common-templates` (consumer)
- `kubevirt/cluster-network-addons-operator` (consumer)
- `kubevirt/hyperconverged-cluster-operator` (consumer)
- `kubevirt/kubevirt-tekton-tasks` (consumer — already has AGENTS.md via PR #828)
- Additional kubevirt/* repositories as they opt in

## Design

### Shared Repository: `kubevirt/ai-sdlc`

A new repository serving as the single source of truth for organization-wide AI development
guidance. Following the [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/)
principle of treating repository knowledge as the system of record, this repository
provides a map, not an encyclopedia.

#### Repository Structure

```
kubevirt/ai-sdlc/
├── AGENTS.md                    # Entry point for OpenAI Codex
├── CLAUDE.md                    # Entry point for Claude Code
├── .cursor/
│   └── rules/
│       └── kubevirt-base.mdc    # Entry point for Cursor
├── .github/
│   └── copilot-instructions.md  # Entry point for GitHub Copilot
├── docs/
│   ├── go-conventions.md        # Go coding style across kubevirt org
│   ├── api-conventions.md       # Kubernetes API / CRD patterns
│   ├── testing-patterns.md      # Unit, integration, e2e test conventions
│   ├── controller-patterns.md   # Reconciler / controller-runtime conventions
│   ├── ci-patterns.md           # Prow / OpenShift CI conventions
│   ├── commit-conventions.md    # DCO sign-off, commit message format
│   ├── review-conventions.md    # OWNERS, /lgtm, /approve workflow
│   ├── release-conventions.md   # Release process, cherry-picks
│   └── ai-workflow.md           # How to structure AI-assisted contributions
├── rules/
│   ├── base.md                  # Core rules all projects inherit
│   ├── go-project.md            # Rules for Go-based projects
│   ├── operator-project.md      # Rules for operator-pattern projects
│   └── tekton-project.md        # Rules for Tekton-based projects
├── skills/                      # Reusable AI agent skills (Cursor skills format)
│   ├── debug-e2e/
│   │   └── SKILL.md
│   ├── create-vep/
│   │   └── SKILL.md
│   ├── review-pr/
│   │   └── SKILL.md
│   └── onboard-contributor/
│       └── SKILL.md
├── templates/
│   ├── project-agents.md.tmpl   # Template for per-project AGENTS.md
│   ├── project-cursorrules.tmpl # Template for per-project .cursorrules
│   └── project-claude.md.tmpl   # Template for per-project CLAUDE.md
├── scripts/
│   ├── validate-docs.sh         # CI: check doc freshness and cross-links
│   ├── sync-to-project.sh       # Helper: sync shared rules to a consumer repo
│   └── lint-agents-md.sh        # CI: validate AGENTS.md structure
├── CONTRIBUTING.md
└── README.md
```

### Layered Architecture

Following the principle that agents work best with
[strict boundaries and predictable structure](https://openai.com/index/harness-engineering/),
the architecture uses a layered model:

#### Layer 1: Organization-Wide Base Rules (`kubevirt/ai-sdlc`)

Core conventions that apply to every KubeVirt project:

- Go coding style (gofmt, golangci-lint configurations)
- Kubernetes API conventions (CRD versioning, status subresource patterns)
- Commit conventions (DCO sign-off requirement, commit message format)
- Review workflow (OWNERS files, Prow commands: /lgtm, /approve, /hold)
- CI conventions (OpenShift CI / Prow job patterns)
- Testing philosophy (unit test expectations, e2e test patterns)

#### Layer 2: Project-Type Rules (`kubevirt/ai-sdlc/rules/`)

Rules for specific project archetypes:

- **go-project.md**: Go module layout, vendoring (`go mod vendor`), build conventions
- **operator-project.md**: controller-runtime patterns, reconciliation loops,
  `EnqueueRequestForOwner`, status conditions
- **tekton-project.md**: Task/Pipeline YAML generation, ClusterTask conventions

#### Layer 3: Per-Project Overrides (in each consumer repo)

Each project maintains its own `AGENTS.md` (and tool-specific files) that:

1. References the shared `kubevirt/ai-sdlc` as the base
2. Adds project-specific guidance (architecture, build commands, test commands)
3. Can override or extend any shared rule

Example per-project `AGENTS.md`:

```markdown
# Project: kubevirt/ssp-operator

## Base Rules
This project follows the KubeVirt organization AI development guidelines.
See: https://github.com/kubevirt/ai-sdlc

## Project-Specific Rules

### Architecture
- Operator built with operator-sdk and controller-runtime
- Main reconciler: controllers/ssp_controller.go
- ...

### Build
- `make build` — compile
- `make test` — unit tests
- `make functest` — e2e tests (requires running cluster)

### Key Conventions
- All CRDs use `ssp.kubevirt.io` API group
- ...
```

### Multi-Tool Support

Following the [harness-skills](https://github.com/harness/harness-skills) pattern,
the shared repository provides entry points for each major AI tool:

| Tool | Entry Point | Auto-loaded |
|------|-------------|-------------|
| OpenAI Codex | `AGENTS.md` | Yes |
| Claude Code | `CLAUDE.md` | Yes |
| Cursor | `.cursor/rules/*.mdc` | Yes |
| GitHub Copilot | `.github/copilot-instructions.md` | Yes |

All entry points reference the same underlying `docs/` and `rules/` content, avoiding the
duplication problem identified in the
[kubevirt-tekton-tasks PR #828 review](https://github.com/kubevirt/kubevirt-tekton-tasks/pull/828):
"`.claude/CLAUDE.md`, `.cursorrules`, and `.cursor/rules/read-agents-first.mdc` contain
largely the same content... Consider having .cursorrules and CLAUDE.md simply point to
AGENTS.md rather than duplicating the rules list."

### Progressive Disclosure

Following the OpenAI lesson that "a giant instruction file crowds out the task," the entry
point files are kept short (~100 lines) and serve as a table of contents pointing to deeper
documentation:

```
AGENTS.md (map)
  ├── docs/go-conventions.md (detail)
  ├── docs/testing-patterns.md (detail)
  ├── docs/ci-patterns.md (detail)
  └── ...
```

Agents discover context progressively rather than being overwhelmed upfront.

### Skills (Reusable Agent Capabilities)

Inspired by the [harness-skills](https://github.com/harness/harness-skills) pattern, the
shared repository includes reusable skills that any KubeVirt project can reference:

- **debug-e2e**: Systematic approach to diagnosing e2e test failures across KubeVirt projects
- **create-vep**: Guide for creating a new VEP following the template
- **review-pr**: Checklist-driven PR review following KubeVirt conventions
- **onboard-contributor**: Step-by-step guide for new contributor setup

### Documentation Freshness Enforcement

Following the OpenAI practice of mechanically enforcing documentation quality, CI jobs in
`kubevirt/ai-sdlc` validate:

- Cross-links between documents are valid
- Required sections are present in all docs
- No stale references to removed code patterns
- Consumer projects reference a compatible version of the shared rules

### Import Mechanism

Consumer projects reference the shared rules through one of:

1. **Git submodule** — `git submodule add https://github.com/kubevirt/ai-sdlc .ai-sdlc`
2. **Direct URL references** — Entry points link to raw GitHub URLs of shared docs
3. **Copy + version pin** — Scripts sync shared rules into the project with a version tag,
   CI validates the pinned version is not too old

The recommended approach is **direct URL references** for simplicity, with a CI check that
validates the referenced commit/tag is not stale.

## API Examples

Not applicable — this is a process and tooling proposal, not an API change.

## Alternatives

### Alternative 1: Per-Project Only (Status Quo)

Each project independently maintains its own AI guidance files with no shared base.

**Pros**: Full autonomy, no cross-repo dependencies.
**Cons**: Duplication, inconsistency, knowledge doesn't propagate, higher maintenance burden
as each project reinvents the same patterns.

### Alternative 2: Monolithic Organization-Wide AGENTS.md

A single large file checked into every repo via automation.

**Pros**: Single source of truth.
**Cons**: Context overload for agents (proven to fail at scale per OpenAI's experience),
no room for per-project customization, stale rules accumulate.

### Alternative 3: Wiki / External Documentation

Maintain AI guidance in GitHub wiki or external docs site.

**Pros**: Easy to edit.
**Cons**: Not version-controlled with code, invisible to AI agents (agents can only see
repository-local content), no CI enforcement.

## Scalability

This approach scales naturally:
- New projects opt in by creating a thin `AGENTS.md` that references the shared base.
- Shared rules evolve independently from per-project overrides.
- Skills are additive — new skills don't affect existing projects unless explicitly imported.
- The kubevirt organization currently has 15+ repositories; this approach avoids the
  N×M problem of maintaining AI guidance independently in each.

## Update/Rollback Compatibility

No code or API compatibility concerns. Projects can adopt the shared rules incrementally.
Rolling back means removing the reference to the shared repository, reverting to
project-local-only guidance.

## Functional Testing Approach

- **CI linting**: Validate document structure, cross-links, and freshness in the shared repo.
- **Consumer validation**: CI in consumer repos validates that referenced shared docs exist
  and are accessible.
- **Periodic review**: Quarterly review of shared rules against actual project practices,
  similar to the "doc-gardening" agent approach described in
  [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/).

## Implementation History

- 2025-04: [kubevirt-tekton-tasks PR #828](https://github.com/kubevirt/kubevirt-tekton-tasks/pull/828)
  — First KubeVirt project to add AGENTS.md and modular documentation structure,
  demonstrating the per-project approach and identifying the duplication problem.
- 2026-04: This VEP — proposing the shared organization-wide approach.

## Graduation Requirements

As this is a meta-VEP about process and tooling (not a feature), graduation is defined
differently:

### Alpha

- [ ] Create the `kubevirt/ai-sdlc` repository with initial structure
- [ ] Populate base rules covering Go conventions, testing patterns, and CI conventions
- [ ] Provide multi-tool entry points (AGENTS.md, CLAUDE.md, .cursor/rules/, copilot-instructions.md)
- [ ] Migrate `kubevirt/kubevirt-tekton-tasks` to use shared base + project overlay
- [ ] At least 2 projects adopt the shared base

### Beta

- [ ] At least 5 projects adopt the shared base
- [ ] CI linting for documentation freshness is operational
- [ ] At least 3 reusable skills are available
- [ ] Community feedback incorporated from at least 2 SIGs
- [ ] Templates for per-project AGENTS.md are tested and documented

### GA

- [ ] Majority of active kubevirt/* repositories use the shared base
- [ ] Process documented in kubevirt/community
- [ ] Skills cover the most common contributor workflows
- [ ] Quarterly doc-gardening process established
