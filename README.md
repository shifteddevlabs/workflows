# shifteddevlabs/workflows

Shared GitHub Actions workflows for the Shifted Dev Labs organization.

## Available Workflows

### Claude Code Review (`claude-review.yml`)

AI-powered code review that runs on PRs and pushes to main.

- **PR Review:** Uses `anthropics/claude-code-action@v1` to post inline comments with file:line citations
- **Push Review:** Calls the Anthropic API directly, sends a structured email digest via Resend

Both use a verification-first methodology — the reviewer must check for existing mitigations before reporting, and every finding requires a specific file, line, exploit path, and confidence score.

---

## Quick Start

Create `.github/workflows/claude-review.yml` in your repo:

```yaml
name: Claude Code Review

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    uses: shifteddevlabs/workflows/.github/workflows/claude-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      RESEND_API_KEY: ${{ secrets.RESEND_API_KEY }}
```

That's it. ~15 lines gets you the full review pipeline.

---

## Required Secrets

| Secret | Required | Used By |
|--------|----------|---------|
| `ANTHROPIC_API_KEY` | Yes | Both PR and push reviews |
| `RESEND_API_KEY` | No | Push review email digest (skip email if not set) |

Set these as org-level secrets or per-repo in Settings > Secrets and variables > Actions.

---

## Optional Inputs

Override defaults by passing `with:` in the caller:

```yaml
jobs:
  review:
    uses: shifteddevlabs/workflows/.github/workflows/claude-review.yml@main
    with:
      review_model: 'claude-sonnet-4-6'
      email_to: 'team@example.com'
      confidence_threshold: 0.8
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      RESEND_API_KEY: ${{ secrets.RESEND_API_KEY }}
```

| Input | Default | Description |
|-------|---------|-------------|
| `review_model` | `claude-sonnet-4-6` | Anthropic model for push reviews |
| `email_to` | `shiftedmediagroup@gmail.com` | Email recipient for push digest |
| `email_from` | `Code Review <codereview@shiftedmediagroup.com>` | Email sender |
| `diff_limit` | `100000` | Max chars for diff (truncated beyond this) |
| `file_content_limit` | `80000` | Max chars for full file content |
| `confidence_threshold` | `0.7` | Min confidence (0-1) for main report. Below-threshold findings appear in a separate "not actionable" section |

---

## Per-Repo Context (Optional)

Create `.github/review-context.md` in your repo to give the reviewer project-specific knowledge. The workflow automatically reads this file and injects it into the prompt.

Use it to document:
- Auth model and security patterns
- Database constraints and race condition handling
- Known-safe patterns the reviewer should not flag
- Key files and their purposes

**Example:** See [smg-ep/.github/review-context.md](https://github.com/shifteddevlabs/smg-ep/blob/main/.github/review-context.md)

---

## Push Review Output Format

The push review returns structured JSON — each finding requires evidence:

```json
{
  "summary": "One-line summary",
  "severity": "none|low|medium|high|critical",
  "issues": [
    {
      "severity": "medium",
      "confidence": 0.9,
      "file": "src/api/route.ts",
      "line": 42,
      "title": "Short description",
      "exploit_path": "How to trigger this",
      "why_not_mitigated": "Why existing code doesn't handle it",
      "fix": "Recommended fix"
    }
  ],
  "verified_safe": [
    "Token validated against DB on lines 50-54",
    "UUID has 122-bit entropy — brute force infeasible"
  ]
}
```

The `verified_safe` array shows what the reviewer checked and confirmed is not an issue — proving it did the verification work even on "all clear" reviews.

---

## Severity Definitions

| Level | Meaning | Evidence Required |
|-------|---------|-------------------|
| **CRITICAL** | Exploitable security vulnerability or data exposure | Exact exploit path |
| **HIGH** | Bug that will cause incorrect behavior in production | Trigger scenario |
| **MEDIUM** | Missing error handling or edge case with real impact | Edge case description |
| **LOW** | Code quality improvement or minor optimization | Suggestion only |
| **NONE** | All clear — no issues found | What was verified |

---

## How It Reduces False Positives

1. **Full file context** — push reviews see entire changed files (with line numbers), not just diffs
2. **Multi-commit range** — reviews all commits in a push, not just the last one
3. **Verification-first methodology** — reviewer must try to disprove each finding before reporting
4. **Known safe patterns** — common false-positive triggers (DB constraints, UUID entropy, JSX escaping) are pre-documented
5. **Structured evidence** — every finding needs file, line, exploit path, and confidence score
6. **Confidence filtering** — low-confidence findings are separated from actionable ones
7. **Per-repo context** — project-specific knowledge prevents architectural misunderstandings
