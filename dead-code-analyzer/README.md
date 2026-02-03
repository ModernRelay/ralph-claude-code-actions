# Claude Dead Code Analyzer

Analyze dead code in TypeScript/JavaScript projects using Claude Code and [Knip](https://knip.dev/).

## Features

- **Static analysis** - Uses Knip for comprehensive dead code detection
- **AI-powered insights** - Claude analyzes results and provides actionable recommendations
- **PR mode** - Focuses on changed files and posts comments on pull requests
- **Comprehensive mode** - Full codebase analysis with GitHub issue creation
- **Rich reporting** - Markdown reports, JSON artifacts, and GitHub integration

## Prerequisites

This action requires:
- **[Knip](https://knip.dev/)** - Dead code detection tool
  - Install: `npm install -D knip` (or `pnpm add -D knip`)
  - Optional: Create `knip.json` for configuration

## Quick Start

### 1. Install Knip in your project

```bash
pnpm add -D knip
```

Optionally create `knip.json`:

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "entry": ["src/index.ts"],
  "project": ["src/**/*.ts"]
}
```

### 2. Add the workflow

Create `.github/workflows/dead-code.yml`:

```yaml
name: Dead Code Analysis

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "**/*.ts"
      - "**/*.tsx"
      - "!**/*.test.ts"
      - "!**/*.spec.ts"

  schedule:
    # Weekly comprehensive analysis - Monday 9:00 AM UTC
    - cron: "0 9 * * 1"

  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile

      - name: Analyze Dead Code
        uses: ModernRelay/ralph-claude-code-actions/dead-code-analyzer@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### 3. Set up secrets

Add `CLAUDE_CODE_OAUTH_TOKEN` to your repository secrets:
1. Run `claude /login` locally
2. Copy the OAuth token
3. Add to GitHub repository secrets

## Usage Modes

### PR Mode (Default for Pull Requests)

When triggered by a pull request, the action:
1. Detects changed TypeScript/JavaScript files
2. Runs Knip for static analysis
3. Analyzes changes for new dead code
4. Posts a comment on the PR with findings

```yaml
- uses: ModernRelay/ralph-claude-code-actions/dead-code-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    pr_number: ${{ github.event.pull_request.number }}
```

### Comprehensive Mode (Scheduled/Manual)

For scheduled or manual runs, the action:
1. Analyzes the entire codebase
2. Categorizes issues by priority
3. Creates a GitHub issue with findings

```yaml
- uses: ModernRelay/ralph-claude-code-actions/dead-code-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    create_issue: "true"
    issue_labels: "dead-code,tech-debt,automated"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `claude_code_oauth_token` | Yes | - | Claude Code OAuth token |
| `github_token` | No | - | GitHub token for PR comments/issues |
| `pr_number` | No | Auto-detected | Pull request number |
| `changed_files` | No | Auto-detected | Changed files (newline-separated) |
| `config_path` | No | `knip.json` | Path to Knip config |
| `working_directory` | No | `.` | Working directory |
| `package_manager` | No | `pnpm` | Package manager (npm/yarn/pnpm) |
| `install_command` | No | Auto | Custom install command |
| `knip_args` | No | - | Additional Knip arguments |
| `model` | No | `sonnet` | Claude model to use |
| `max_turns` | No | `100` | Max Claude turns |
| `create_issue` | No | `true` | Create issue in comprehensive mode |
| `issue_labels` | No | `dead-code,automated` | Labels for issues |

## Outputs

| Output | Description |
|--------|-------------|
| `result` | `clean` or `issues-found` |
| `issues_count` | Number of issues found |
| `report_path` | Path to the report file |

## What It Detects

### High Priority
- **Unused files** - Files not imported anywhere
- **Unused dependencies** - Packages in package.json not used
- **Unresolved imports** - Imports pointing to non-existent files
- **Backward-compat shims** - `_unused` variables, unnecessary re-exports

### Medium Priority
- **Unused exports** - Exported but never imported elsewhere
- **Commented-out code** - Large blocks of commented code
- **Duplicate exports** - Same thing exported multiple ways

### Low Priority
- **Unused types** - Type definitions never referenced
- **Dead patterns** - Always-true conditions, unreachable code

## Example Output

### PR Comment

```markdown
## 🧹 Dead Code Analysis

| Status | Issues |
|--------|--------|
| ⚠️ | 3 |

### Issues Found
- **src/utils/helpers.ts:42** - Unused export `formatDate`
- **src/components/Button.tsx:15** - Unused import `useCallback`
- **src/lib/api.ts:78** - Unused function `legacyFetch`

<details>
<summary>Knip Output</summary>

Unused exports:
- src/utils/helpers.ts: formatDate
...
</details>
```

## Knip Configuration

For best results, configure Knip for your project. Example `knip.json`:

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "entry": ["src/index.ts", "src/main.tsx"],
  "project": ["src/**/*.{ts,tsx}"],
  "ignore": ["**/*.test.ts", "**/*.spec.ts"],
  "ignoreDependencies": ["@types/*"],
  "ignoreWorkspaces": ["packages/legacy"]
}
```

See [Knip documentation](https://knip.dev/overview/configuration) for all options.

## Monorepo Support

For monorepos, Knip automatically detects workspaces. Configure each workspace:

```json
{
  "workspaces": {
    "packages/*": {
      "entry": ["src/index.ts"]
    },
    "apps/web": {
      "entry": ["src/main.tsx", "src/pages/**/*.tsx"]
    }
  }
}
```

## License

MIT
