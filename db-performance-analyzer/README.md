# Claude DB Performance Analyzer

Analyze PostgreSQL database performance using Claude Code with EXPLAIN ANALYZE.

## Features

- **Query plan analysis** - Uses EXPLAIN ANALYZE for actual execution statistics
- **Function profiling** - Analyzes PL/pgSQL functions for performance issues
- **AI-powered insights** - Claude interprets execution plans and provides recommendations
- **PR mode** - Analyzes changed SQL files and posts comments on pull requests
- **Comprehensive mode** - Full database analysis with GitHub issue creation
- **Index recommendations** - Suggests missing indexes and identifies unused ones

## Prerequisites

This action requires:
- **PostgreSQL database** - Accessible from the GitHub Actions runner
- **psql client** - Automatically installed by the action if not present
- **Database URL** - Connection string with appropriate permissions

## Quick Start

### 1. Set up database access

Ensure your database is accessible from GitHub Actions:
- For local development databases: Use a tunneling solution or hosted database
- For cloud databases: Whitelist GitHub Actions IP ranges or use a connection proxy

### 2. Add the workflow

Create `.github/workflows/db-performance.yml`:

```yaml
name: Database Performance Analysis

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "**/*.sql"
      - "**/migrations/**"

  schedule:
    # Weekly comprehensive analysis - Monday 10:00 AM UTC
    - cron: "0 10 * * 1"

  workflow_dispatch:
    inputs:
      functions:
        description: "Specific functions to analyze (comma-separated)"
        required: false

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

      - name: Analyze Database Performance
        uses: ModernRelay/ralph-claude-code-actions/db-performance-analyzer@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          database_url: ${{ secrets.DATABASE_URL }}
          functions: ${{ github.event.inputs.functions }}
```

### 3. Set up secrets

Add these secrets to your repository:
- `CLAUDE_CODE_OAUTH_TOKEN` - From `claude /login`
- `DATABASE_URL` - PostgreSQL connection string

## Usage Modes

### PR Mode (Default for Pull Requests)

When triggered by a pull request with SQL changes:
1. Detects changed SQL files
2. Extracts function definitions
3. Runs EXPLAIN ANALYZE on new/modified functions
4. Posts a comment with performance analysis

```yaml
- uses: ModernRelay/ralph-claude-code-actions/db-performance-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    database_url: ${{ secrets.DATABASE_URL }}
    slow_query_threshold_ms: "100"
```

### Comprehensive Mode (Scheduled/Manual)

For scheduled or manual runs:
1. Catalogs all functions in the database
2. Profiles frequently-used functions
3. Checks for missing indexes
4. Creates a GitHub issue with findings

```yaml
- uses: ModernRelay/ralph-claude-code-actions/db-performance-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    database_url: ${{ secrets.DATABASE_URL }}
    mode: "comprehensive"
    create_issue: "true"
```

### Specific Mode (Targeted Analysis)

Analyze specific functions by name:

```yaml
- uses: ModernRelay/ralph-claude-code-actions/db-performance-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    database_url: ${{ secrets.DATABASE_URL }}
    functions: "get_user_data,calculate_totals,process_orders"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `claude_code_oauth_token` | Yes | - | Claude Code OAuth token |
| `database_url` | Yes | - | PostgreSQL connection string |
| `github_token` | No | - | GitHub token for PR comments/issues |
| `pr_number` | No | Auto-detected | Pull request number |
| `sql_files` | No | Auto-detected | SQL files to analyze |
| `functions` | No | - | Specific functions to analyze |
| `mode` | No | Auto-detected | `pr`, `comprehensive`, or `specific` |
| `schemas_dir` | No | - | Directory with schema files |
| `migrations_dir` | No | - | Directory with migration files |
| `slow_query_threshold_ms` | No | `100` | Slow query threshold (ms) |
| `model` | No | `sonnet` | Claude model to use |
| `max_turns` | No | `150` | Max Claude turns |
| `create_issue` | No | `true` | Create issue in comprehensive mode |
| `issue_labels` | No | `performance,database,automated` | Labels for issues |

## Outputs

| Output | Description |
|--------|-------------|
| `result` | `optimal`, `warnings`, or `issues-found` |
| `issues_count` | Number of issues found |
| `slow_queries_count` | Number of slow queries |
| `report_path` | Path to the report file |

## What It Analyzes

### Query Performance
- **Sequential scans** - Identifies full table scans on large tables
- **Missing indexes** - Suggests indexes for frequently filtered columns
- **Index usage** - Verifies existing indexes are being used
- **Join performance** - Analyzes JOIN operations for efficiency

### Function Performance
- **Execution time** - Profiles actual execution with EXPLAIN ANALYZE
- **Buffer usage** - Checks for excessive I/O operations
- **Row estimates** - Identifies statistics issues causing bad plans
- **N+1 patterns** - Detects queries that should be batched

### Database Health
- **Table bloat** - Identifies tables needing VACUUM
- **Statistics freshness** - Checks if ANALYZE is needed
- **Unused indexes** - Finds indexes consuming space but not used
- **Connection usage** - Monitors connection pool efficiency

## Example Output

### PR Comment

```markdown
## 📊 Database Performance Analysis

| Metric | Value |
|--------|-------|
| Functions Analyzed | 3 |
| Issues Found | 2 |
| Slow Queries | 1 |

### Issues Found

#### 🔴 Critical
- **`get_user_orders`** - Sequential scan on `orders` table (500k rows)
  - **Execution time**: 1,250ms
  - **Fix**: Add index on `(user_id, created_at)`
  ```sql
  CREATE INDEX CONCURRENTLY idx_orders_user_created
  ON orders (user_id, created_at DESC);
  ```

#### 🟡 Warnings
- **`calculate_monthly_totals`** - Query time: 180ms (threshold: 100ms)
  - **Suggestion**: Consider materialized view for aggregations

<details>
<summary>Execution Plans</summary>

**get_user_orders**
```
Seq Scan on orders  (cost=0.00..15234.00 rows=500000 width=48)
  Filter: (user_id = $1)
  Rows Removed by Filter: 499000
  Buffers: shared hit=12500
Planning Time: 0.5ms
Execution Time: 1250.3ms
```
</details>
```

## Database Connection

### Connection String Format

```
postgresql://username:password@host:port/database?sslmode=require
```

### Security Recommendations

1. **Use a read-only user** for analysis
2. **Restrict permissions** to SELECT and EXPLAIN only
3. **Use SSL** for connections (`?sslmode=require`)
4. **Consider a replica** for production databases

Example read-only user setup:
```sql
CREATE USER analyzer WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE mydb TO analyzer;
GRANT USAGE ON SCHEMA public TO analyzer;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyzer;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO analyzer;
```

## Supabase Integration

For Supabase projects, you can use the database URL from your project settings:

```yaml
- name: Start Supabase (local)
  run: npx supabase start

- name: Analyze Database
  uses: ModernRelay/ralph-claude-code-actions/db-performance-analyzer@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    database_url: "postgresql://postgres:postgres@localhost:54322/postgres"
```

Or for hosted Supabase:
```yaml
database_url: ${{ secrets.SUPABASE_DB_URL }}
```

## Tips for Best Results

### Provide Schema Context

Include your schema directory for better analysis:
```yaml
schemas_dir: "./supabase/schemas"
migrations_dir: "./supabase/migrations"
```

### Test with Representative Data

Ensure your test database has realistic data volumes for accurate EXPLAIN ANALYZE results.

### Set Appropriate Thresholds

Adjust `slow_query_threshold_ms` based on your application requirements:
- Real-time APIs: 50-100ms
- Background jobs: 500-1000ms
- Reports: 5000-10000ms

## License

MIT
