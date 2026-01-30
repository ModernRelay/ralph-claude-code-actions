# Agentic UI Tests

Run end-to-end UI tests written in plain English markdown using Claude Code and [agent-browser](https://agent-browser.dev/).

## Features

- **Plain English tests** - Write tests in markdown, no code required
- **AI-powered execution** - Claude interprets and executes your tests
- **Self-healing** - Claude adapts to UI changes without brittle selectors
- **Rich reporting** - GitHub job summary, PR comments, JSON artifacts
- **Screenshots** - Automatic capture on failure or always

## Quick Start

### 1. Create a test file

Create `e2e-tests/login.test.md`:

```markdown
---
name: User Login
timeout: 120000
tags: [auth, smoke]
credentials:
  - TEST_EMAIL
  - TEST_PASSWORD
---

# User Login

## Steps

1. Navigate to {{BASE_URL}}/login
2. Enter "{{TEST_EMAIL}}" in the email field
3. Enter "{{TEST_PASSWORD}}" in the password field
4. Click the "Sign In" button
5. **Assert**: URL contains "/dashboard"
6. **Assert**: Welcome message is visible
```

### 2. Add the workflow

Create `.github/workflows/e2e-tests.yml`:

```yaml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start dev server
        run: |
          npm ci
          npm run dev &
          npx wait-on http://localhost:3000

      - name: Run E2E Tests
        uses: ModernRelay/ralph-claude-code-actions/agentic-ui-tests@main
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          base_url: 'http://localhost:3000'
          test_dir: './e2e-tests'
          secrets_json: |
            {
              "TEST_EMAIL": "${{ secrets.TEST_EMAIL }}",
              "TEST_PASSWORD": "${{ secrets.TEST_PASSWORD }}",
              "BASE_URL": "http://localhost:3000"
            }
```

### 3. Set up secrets

1. Run `claude /login` locally to get your OAuth token
2. Add these secrets to your GitHub repository:
   - `CLAUDE_CODE_OAUTH_TOKEN` - Your Claude Code OAuth token
   - `TEST_EMAIL` - Test user email
   - `TEST_PASSWORD` - Test user password

## Test File Format

### Frontmatter Options

```yaml
---
name: Test Name                    # Required: Test name
timeout: 120000                    # Optional: Timeout in ms (default: 120000)
tags: [auth, critical]             # Optional: Tags for filtering
retries: 1                         # Optional: Retry count
session: isolated                  # Optional: isolated | shared
credentials:                       # Optional: Required secret names
  - SECRET_NAME
screenshotMode: on-failure         # Optional: always | on-failure | never
skip: false                        # Optional: Skip this test
only: false                        # Optional: Run only this test
dependsOn: [login]                 # Optional: Tests that must run first (by name)
saveProfile: authenticated         # Optional: Save browser state after test
profile: authenticated             # Optional: Load saved browser state before test
---
```

### Test Dependencies & Profiles

Tests can declare dependencies on other tests and share authentication state using profiles.

**Login test (saves profile):**
```yaml
---
name: login
saveProfile: authenticated         # Save cookies/localStorage after login
---
```

**Dependent test (loads profile):**
```yaml
---
name: Create Repository
dependsOn: [login]                 # Wait for login test to complete
profile: authenticated             # Load the saved auth state
---
```

**How it works:**
1. Tests with no `dependsOn` run first
2. After a test with `saveProfile` passes, agent-browser saves state: `agent-browser state save profiles/<name>.json`
3. Tests with `profile` load that state before starting: `agent-browser state load profiles/<name>.json`
4. Tests with `dependsOn` wait for all dependencies to pass before running

### Step Format

- **Numbered steps**: `1. Do something`
- **Assertions**: `5. **Assert**: Condition to verify`
- **Variables**: `{{VARIABLE_NAME}}` - Replaced with secrets

### Sections

```markdown
# Test Title

Optional description paragraph.

## Prerequisites
- Precondition 1
- Precondition 2

## Steps
1. First action
2. Second action
3. **Assert**: Verification

## Cleanup
- Cleanup action 1
- Cleanup action 2
```

## Action Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `claude_code_oauth_token` | Yes | - | Claude Code OAuth token |
| `base_url` | Yes | - | Application URL |
| `test_dir` | No | `./e2e-tests` | Test files directory |
| `secrets_json` | No | `{}` | JSON with credentials |
| `filter` | No | - | Filter by name/tag |
| `tags` | No | - | Filter by tags (comma-separated) |
| `timeout` | No | `120000` | Test timeout (ms) |
| `screenshot_mode` | No | `on-failure` | Screenshot capture mode |
| `fail_fast` | No | `false` | Stop on first failure |
| `model` | No | `sonnet` | Claude model to use |

## Action Outputs

| Output | Description |
|--------|-------------|
| `result` | `passed` or `failed` |
| `passed` | Number of passed tests |
| `failed` | Number of failed tests |

## How It Works

1. **Claude Code** receives your test markdown files
2. **Claude** reads and interprets each step
3. **agent-browser** executes browser commands:
   - `open <url>` - Navigate
   - `snapshot -i` - Get interactive elements
   - `click @e1` - Click using element refs
   - `fill @e1 "text"` - Fill inputs
4. **Claude** verifies assertions and reports results
5. **GitHub Action** posts summary and PR comment


## Roadmap

### Learned Execution Paths

Currently, Claude interprets and executes one step at a time.

Soon, Claude will record the "trail" on first run - mapping intents to selectors:

```
"Click the Sign In button"  →  button[type=submit], text="Sign In"
"Enter email"               →  input[name=email], label="Email address"
```

On subsequent runs, Claude executes multiple steps as chained commands using the cached selectors, only re-interpreting when one fails.

The trail is a hint, not a script - Claude still adapts when the UI changes.

## License

MIT
