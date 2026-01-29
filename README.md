# Ralph Claude Code Actions

A collection of GitHub Actions powered by Claude Code for automating development workflows.

## Available Actions

| Action | Description |
|--------|-------------|
| [agentic-ui-tests](./agentic-ui-tests) | Run E2E UI tests written in plain English markdown |

## Usage

Each action can be used directly from this repository:

```yaml
- uses: ModernRelay/ralph-claude-code-actions/<action-name>@main
```

See individual action READMEs for detailed usage instructions.

## Requirements

All actions require a Claude Code OAuth token. To obtain one:

1. Install Claude Code CLI
2. Run `claude /login`
3. Add the token to your repository secrets as `CLAUDE_CODE_OAUTH_TOKEN`

## License

MIT
