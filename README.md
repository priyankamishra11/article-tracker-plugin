# article-tracker — Claude Code Skill

A Claude Code skill that tracks newsletter articles from Gmail, saves them locally as markdown files, and reports what's new — with no manual reading required.

## What it does

1. Searches your Gmail for emails from the newsletter sender(s) you specify
2. Reads each email and extracts the canonical article URL
3. Saves content in priority order: **PDF attachment → web fetch → email body**
4. Reports every new article with its saved path

Handles paywalled articles gracefully — falls back to the email body, which for most newsletters contains the full article text verbatim.

---

## Prerequisites

### 1. Claude Code
Install from [claude.ai/code](https://claude.ai/code) or via npm:
```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Gmail MCP
This skill requires the Gmail MCP server to read your emails.

```bash
claude mcp add --scope user gmail -- npx -y @gongrzhe/server-gmail-autoauth-mcp
```

On first run, a browser window will open for Google OAuth — sign in to grant read-only Gmail access. Credentials are stored in `~/.gmail-mcp/`.

> **Gmail access is read-only.** The skill never sends, replies, archives, labels, or deletes anything.

---

## Installation

### Option A — Global (available in every Claude Code session)
```bash
curl -o ~/.claude/commands/article-tracker.md \
  https://raw.githubusercontent.com/priyankamishra11/article-tracker-plugin/main/.claude/commands/article-tracker.md
```

Or manually:
```bash
cp .claude/commands/article-tracker.md ~/.claude/commands/article-tracker.md
```

### Option B — Project-level (only available inside this repo)
Clone the repo into your project. Claude Code automatically picks up skills from `.claude/commands/` in the current directory.

```bash
git clone https://github.com/priyankamishra11/article-tracker-plugin.git
cd article-tracker-plugin
claude  # skill is now available as /article-tracker
```

---

## Usage

```
/article-tracker <sender(s)> [folder] [days]
```

| Parameter | Required | Default | Example |
|-----------|----------|---------|---------|
| `sender(s)` | Yes | — | `a@x.com` or `a@x.com,b@y.com` |
| `folder` | No | `~/Documents/Articles` | `~/MyReadings` |
| `days` | No | `7` | `14` |

**Examples:**
```
/article-tracker pragmaticengineer@substack.com
/article-tracker lenny@substack.com,tldrnewsletter@substack.com
/article-tracker lenny@substack.com ~/MyReadings 14
```

> **Important:** Use the actual `From:` address from the email, not the newsletter's website domain.
> Open any email → check the sender → use that address.
> Example: Pragmatic Engineer arrives from `pragmaticengineer@substack.com`, not `pragmaticengineer.com`.

---

## Output

Saved files go to `~/Documents/Articles/` (or your chosen folder):

| Source | Filename | Quality |
|--------|----------|---------|
| PDF attachment | `YYYY-MM-DD_title.pdf` | Original |
| Web fetch | `YYYY-MM-DD_title.md` | May be AI-processed |
| Email body | `YYYY-MM-DD_title_email.md` | Original verbatim |

Each run skips articles already saved — safe to run on a schedule.

---

## Supported newsletter platforms

Works with newsletters from any platform. Smart search handles hosting platforms differently to avoid noise:

- **Substack, Beehiiv, Mailchimp, ConvertKit** — uses exact sender address search only
- **First-party domains** (e.g. `newsletter.yourname.com`) — also tries partial name match to find the real sending address

---

## License

MIT
