# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TrendRadar is a hot news aggregation and notification system that crawls trending news from 11+ Chinese platforms (Weibo, Zhihu, Douyin, Baidu, Bilibili, etc.), filters by user-defined keywords, and pushes notifications via multiple channels (WeChat Work, Feishu, DingTalk, Telegram, Email, ntfy). The project includes an AI analysis feature via MCP (Model Context Protocol) for intelligent news trend analysis.

## Core Commands

### Development & Testing
```bash
# Install dependencies (use platform-specific setup script)
# Mac/Linux:
./setup-mac.sh
# Windows:
setup-windows.bat  # Chinese version
setup-windows-en.bat  # English version

# Run the main crawler/notifier
python main.py

# Start MCP HTTP server (for AI analysis features)
# Mac/Linux:
./start-http.sh
# Windows:
start-http.bat

# Run via GitHub Actions (production)
# Triggered by: schedule (hourly cron) or manual workflow_dispatch
```

### Docker Deployment
```bash
# Using docker-compose (recommended)
cd docker
docker-compose up -d

# Container management
docker exec -it trend-radar python manage.py status  # Check status
docker exec -it trend-radar python manage.py run     # Manual crawl
docker exec -it trend-radar python manage.py logs    # View logs
docker exec -it trend-radar python manage.py config  # Show config
docker restart trend-radar  # Restart container
```

### Configuration Files
- `config/config.yaml` - Main configuration (platforms, webhooks, report mode, weights)
- `config/frequency_words.txt` - User-defined keywords for filtering news
- `.github/workflows/crawler.yml` - GitHub Actions automation workflow

## Architecture

### Main Entry Point
**`main.py`** (173 KB, single-file design for easy upgrades) handles:
1. **Configuration Loading** - Reads from `config/config.yaml` and environment variables
2. **News Crawling** - Fetches from 11+ platforms via newsnow API
3. **Keyword Filtering** - Matches against `frequency_words.txt` (supports +required, !exclude syntax)
4. **Scoring Algorithm** - Ranks news by weighted formula: `rank(60%) + frequency(30%) + hotness(10%)`
5. **Report Generation** - Outputs HTML (`index.html`) and structured data (`output/*.json`)
6. **Multi-channel Notifications** - Sends to configured webhooks (supports batch sending for large messages)

### Report Modes (configured in `config.yaml`)
- **`incremental`** - Only push newly appeared news (zero duplication)
- **`current`** - Push current trending list (hourly updates)
- **`daily`** - Push all news from today (includes previously sent items)

### MCP Server (`mcp_server/`)
AI-powered analysis system using FastMCP 2.0:
- **`server.py`** - Main MCP server with 13 tools (data query, analytics, search, config, system)
- **`services/`** - Data parsing (`parser_service.py`), caching (`cache_service.py`), data access (`data_service.py`)
- **`tools/`** - Organized by functionality:
  - `data_query.py` - Get latest news, trending topics, news by date
  - `analytics.py` - Topic trends, data insights, sentiment analysis, summaries
  - `search_tools.py` - Keyword search, related news history, similarity search
  - `config_mgmt.py` - Config management
  - `system.py` - System status, trigger crawls

### Data Flow
```
newsnow API → main.py (crawl) → keyword filter → scoring → output/
                                                              ├── daily/*.json (daily accumulation)
                                                              ├── current/*.json (latest batch)
                                                              ├── .push_records/ (push history)
                                                              └── index.html (web view)
```

### Notification System
**Webhook Priority** (env vars override `config.yaml`):
1. Check environment variables (GitHub Secrets in Actions, Docker env vars)
2. Fall back to `config/config.yaml` webhooks section

**Batch Sending** - Messages split automatically if exceeding limits:
- WeChat Work / Telegram: 4KB chunks (MESSAGE_BATCH_SIZE)
- DingTalk: 20KB chunks
- Feishu: 29KB chunks

**Push Window Control** (optional):
- `push_window.enabled: true` - Enable time-based push control
- `time_range: {start: "10:00", end: "19:00"}` - Beijing timezone
- `once_per_day: true` - Push only once within window (prevents spam)

### Keyword System (`config/frequency_words.txt`)
Three syntax types (blank lines separate word groups):
- **Normal word** - Match any: `华为`, `AI`
- **Required word** (`+prefix`) - Must contain: `+发布` (requires "发布" in title)
- **Exclude word** (`!prefix`) - Block: `!广告` (filters out ads)

Example:
```
华为
OPPO
+手机
!广告

A股
+涨跌
!预测
```
First group: Match "华为" or "OPPO" AND "手机", exclude "广告"
Second group: Match "A股" AND "涨跌", exclude "预测"

### Environment Variable Overrides (v3.0.5+)
For Docker/NAS deployments where file edits don't persist:
- `ENABLE_CRAWLER` → `crawler.enable_crawler` (true/false)
- `ENABLE_NOTIFICATION` → `notification.enable_notification` (true/false)
- `REPORT_MODE` → `report.mode` (daily/incremental/current)
- `PUSH_WINDOW_ENABLED` → `notification.push_window.enabled` (true/false)
- `PUSH_WINDOW_START` → `notification.push_window.time_range.start`
- `PUSH_WINDOW_END` → `notification.push_window.time_range.end`
- All webhook URLs: `FEISHU_WEBHOOK_URL`, `WEWORK_WEBHOOK_URL`, etc.

## Key Design Patterns

### Single-File Design
`main.py` is intentionally monolithic (not split into modules) to simplify user upgrades via copy-paste when forking the repository.

### Proxy Support
Configure via `config.yaml`:
```yaml
crawler:
  use_proxy: true
  default_proxy: "http://127.0.0.1:10086"
```

### SMTP Auto-Detection
Email notifications auto-detect SMTP settings for Gmail, QQ, 163, Outlook, etc. (see `SMTP_CONFIGS` in `main.py`)

### Time Tracking
News items store:
- `first_seen_time` - Initial discovery timestamp
- `last_seen_time` - Most recent appearance
- `seen_count` - Appearance frequency
- Used for trend analysis and "🆕" new marker

### Security Notes
- **Never commit webhooks to config.yaml** - Use GitHub Secrets or environment variables
- Webhooks exposed in public repos can be abused for spam/phishing

## Common Development Scenarios

### Adding New Platform
1. Check if platform exists in [newsnow sources](https://github.com/ourongxing/newsnow/tree/main/server/sources)
2. Add to `config/config.yaml` platforms list:
```yaml
platforms:
  - id: "platform-id"  # Must match newsnow API
    name: "显示名称"
```

### Modifying Scoring Algorithm
Edit weights in `config/config.yaml`:
```yaml
weight:
  rank_weight: 0.6      # Prioritize high rankings
  frequency_weight: 0.3 # Value persistence
  hotness_weight: 0.1   # Consider engagement
```

### Adjusting Crawl Frequency
Edit `.github/workflows/crawler.yml`:
```yaml
schedule:
  - cron: "0 * * * *"  # Every hour
  # or "*/30 * * * *"  # Every 30 minutes (not recommended - quota limits)
```

### Testing MCP Tools
```bash
# Start HTTP server
./start-http.sh  # or start-http.bat on Windows

# Test with MCP Inspector
npx @modelcontextprotocol/inspector
# Then connect to: http://localhost:3333/mcp
```

### Debugging Notification Issues
1. Check webhook configuration (env vars take precedence)
2. Verify `ENABLE_NOTIFICATION: true` in config
3. Check push window settings if enabled
4. Review `.push_records/` for push history
5. Check message size (may need batch sending)

## Important Implementation Details

### Date Handling
- All timestamps use Beijing timezone (`Asia/Shanghai`)
- Dates stored in `YYYY-MM-DD` format
- MCP tools accept flexible date formats ("today", "2025-11-25", etc.)

### File Persistence
- `output/daily/` - Cumulative news (persists for trend analysis)
- `output/current/` - Latest crawl snapshot
- `output/.push_records/` - Tracks sent notifications (for deduplication)

### Error Handling
- Network failures: Retry logic with exponential backoff
- Missing config: Fails fast with clear error messages
- Empty keyword file: Pushes ALL news (no filtering)

### Version Management
- `VERSION` constant in `main.py` (currently 3.1.0)
- Optional version check against GitHub main branch
- Configure via `app.show_version_update` in `config.yaml`

## Testing Considerations

When testing changes:
1. **Dry run**: Set `ENABLE_NOTIFICATION: false` to test without sending
2. **Crawler disable**: Set `ENABLE_CRAWLER: false` to test with existing data
3. **Keyword testing**: Use minimal `frequency_words.txt` to reduce noise
4. **Time window**: Disable `push_window.enabled` for immediate testing
5. **Manual trigger**: Use `workflow_dispatch` in GitHub Actions for immediate runs

## Dependencies

Core libraries (see `requirements.txt`):
- `requests` - HTTP client for API calls
- `pytz` - Timezone handling
- `PyYAML` - Configuration parsing
- `fastmcp` - MCP server framework (for AI features)
- `websockets` - WebSocket support for MCP

## Deployment Targets

1. **GitHub Actions** (recommended for beginners) - Zero-cost, automated, GitHub Pages output
2. **Docker** (recommended for advanced users) - Full control, precise scheduling, self-hosted
3. **Local Python** - Development and testing

## AI/MCP Integration

The MCP server (`mcp_server/`) enables AI assistants to:
- Query news data via natural language
- Analyze trends and patterns
- Generate summaries and insights
- Compare cross-platform coverage

Supported clients:
- Cherry Studio (GUI, easiest setup)
- Claude Desktop
- Cursor IDE
- VSCode (Cline/Continue extensions)
- Claude Code CLI

Configuration locations:
- Claude Desktop: `~/Library/Application Support/Claude/claude_desktop_config.json` (Mac) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows)
- Cursor: `.cursor/mcp.json` in project root or user home
- VSCode: Varies by extension

## Troubleshooting Tips

**"No webhooks configured"** - Check environment variables are set (GitHub Secrets for Actions, docker env for containers)

**"Config file not found"** - Ensure `config/config.yaml` exists and `CONFIG_PATH` env var is correct

**News not filtered** - Empty `frequency_words.txt` means ALL news pushed (intentional behavior)

**GitHub Actions failing** - Check `requirements.txt` installed, config files exist, Python 3.10+ used

**Docker config not updating** - Use environment variables instead of editing `config.yaml` (v3.0.5+ feature)

**MCP connection fails** - Verify UV package manager installed, check project path in config, ensure no Chinese characters in path
