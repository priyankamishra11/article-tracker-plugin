# article-tracker

Track newsletter articles from Gmail senders, save locally, and report new ones.

**Usage:** `/article-tracker <sender(s)> [folder] [days]`

| Parameter | Required | Default | Example |
|-----------|----------|---------|---------|
| `sender(s)` | Yes | — | `a@x.com,b@y.com` or `a@x.com b@y.com` |
| `folder` | No | `~/Documents/Articles` | `~/MyReadings` |
| `days` | No | `7` | `14` |

**Examples:**
```
/article-tracker noreply@pragmaticengineer.com
/article-tracker a@x.com,b@y.com ~/MyReadings
/article-tracker a@x.com,b@y.com ~/MyReadings 14
```

> **Tip:** Use the actual `From:` address from the email, not the newsletter's website domain.
> Open any email from the newsletter → check who it's from → use that address.
> e.g. Pragmatic Engineer emails arrive from `pragmaticengineer@substack.com`, not `pragmaticengineer.com`.

---

## STEP 0 — Parse arguments

Read `$ARGUMENTS`. Scan each whitespace/comma-separated token:
- Contains `@` → add to `SENDER_LIST`
- Starts with `~` or `/` → set as `ARTICLE_FOLDER`
- Is a plain integer → set as `LOOKBACK_DAYS`

Apply defaults for anything not provided:
- `ARTICLE_FOLDER` → `~/Documents/Articles`
- `LOOKBACK_DAYS` → `7`

For each sender in `SENDER_LIST`, extract `SENDER_DOMAIN` = the part after `@` (e.g., `noreply@pragmaticengineer.com` → `pragmaticengineer.com`).

If `SENDER_LIST` is empty, stop immediately:
```
❌ No sender email provided.
Usage: /article-tracker <sender(s)> [folder] [days]
Example: /article-tracker noreply@pragmaticengineer.com
```

---

## STEP 1 — Verify Gmail MCP (mandatory, non-skippable)

Call Gmail MCP to list labels. If it fails for any reason, stop immediately:
```
❌ GMAIL MCP FAILURE
Error: {exact error text}
Fix: claude mcp add --scope user gmail -- npx -y @gongrzhe/server-gmail-autoauth-mcp
```
Do not proceed, guess, or retry.

---

## STEP 2 — Gmail search with domain fallback (one call per sender, all at once)

Fire **all** Gmail search calls **simultaneously**.

Set `START_DATE` = today minus `LOOKBACK_DAYS`, formatted as `YYYY/MM/DD`.

Define these as **hosting platforms** (domains shared by many senders):
`substack.com`, `beehiiv.com`, `mailchimp.com`, `convertkit.com`, `klaviyo.com`, `sendgrid.net`, `mailgun.org`, `constantcontact.com`

For each sender in `SENDER_LIST`:

**If `SENDER_DOMAIN` is a hosting platform** — fire only Query A (exact match):
```
Query A: from:{full-sender-address} after:{START_DATE}
```
Do NOT fire Query B. Searching `from:substack` or `from:beehiiv` would return every email from that platform — newsletters, notifications, receipts, all mixed together.

**If `SENDER_DOMAIN` is a first-party domain** (not in the hosting platform list) — fire both simultaneously with `maxResults: 100`:
```
Query A: from:{full-sender-address} after:{START_DATE}     (exact match)
Query B: from:{domain-without-tld} after:{START_DATE}      (partial match)
```
Where `domain-without-tld` = domain with the TLD stripped (e.g., `pragmaticengineer.com` → `pragmaticengineer`).

Query B catches newsletters hosted on Substack/Beehiiv where the technical sender is `@substack.com` but the newsletter name (e.g., `pragmaticengineer`) appears in the display name or email local part. Gmail's `from:` operator matches both.

Merge and deduplicate results from both queries by message ID.

On any individual search failure: log `⚠ SEARCH FAILED for {sender}: {error}` and continue.

**Truncation check:** If the number of results returned for any sender equals exactly 100, log:
`⚠ POSSIBLE TRUNCATION: got exactly 100 results for {sender} — there may be more emails in this window. Consider reducing LOOKBACK_DAYS or checking Gmail manually for missed articles.`

Collect all results. Deduplicate by Gmail message ID.

**If total emails found = 0:**
Do NOT stop. Try one fallback search per sender using `maxResults: 100`:
```
label:{derived-label} after:{START_DATE}
```
Where the derived label is the domain root without TLD (e.g., `pragmaticengineer.com` → try label `Pragmatic` or `pragmaticengineer`). Check the label list from Step 1 for the closest match.

After receiving label results, **filter out any email whose date is before `START_DATE`** — the label search may return emails outside the window if Gmail ignores the `after:` on label queries.

If label results also return exactly 100 items, log the same truncation warning.

- Log: `⚠ No results from domain search — used label fallback for {sender}`
- If label fallback also returns 0: log `⚠ No emails found for {sender} in last {LOOKBACK_DAYS} days. Tip: confirm actual sender address by checking Gmail manually.` and skip this sender.

---

## STEP 3 — Parallel email read (all at once)

Fire **all** Gmail read calls **simultaneously** — one per collected email ID.

On any read failure: log `⚠ READ FAILED: email ID {id} — {error}` and skip that email.

From each successfully read email, collect:

**A. Article URL** — Extract exactly **ONE** canonical URL per email, using this priority:
1. The URL on the line starting with "View this post on the web at" — this is always the canonical article URL
2. If not present: the first URL whose domain contains `{SENDER_DOMAIN}` AND path contains `/p/`, `/post/`, `/article/`, `/issues/`, or `/archive/`
3. If not present: the first URL in the email whose path contains `/p/`, `/post/`, `/article/`, `/issues/`, or `/archive/`

**Take only one URL per email.** Do not collect URLs from "Related articles", "You might also like", "References", or "Mentions" sections — those are links to older articles, not the current one.

Exclude: unsubscribe links, CDN image URLs, tracking pixels, social sharing buttons, Substack redirect wrappers (`substack.com/redirect/...`).

Only use URLs present verbatim in the email body. Do not construct or guess any URL.

**B. PDF attachment** — Note if the email has any attachment with `.pdf` extension. Record the attachment filename and message ID for use in Step 4.

**C. Email body text** — Preserve the full email body text in memory. This is the original article content and will be used as fallback in Step 4 if the web fetch fails or paywalls.

If no article URL found in an email: log `⚠ No article link in: "{subject}"` and skip.

---

## STEP 4 — Save content (PDF first, then web fetch, then email body)

Run `mkdir -p {ARTICLE_FOLDER}` first.

Load tool schemas needed: call ToolSearch with `"select:mcp__gmail__download_attachment,WebFetch"` before proceeding.

For each article, first check if **any** version already exists:
- `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}.pdf`
- `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}.md`
- `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}_email.md`

If any of these files exist: log `ℹ Already saved: {path}` and skip this article entirely — do not attempt any priority.

Otherwise, work through the following priority levels **in order** — stop at the first success:

---

### PRIORITY 1 — PDF attachment (best: original unprocessed file)

If the email has a PDF attachment:
- Call `mcp__gmail__download_attachment` with the message ID and attachment filename
- Save to `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}.pdf`
- On success: log `✅ PDF saved (original): {path}` — **DONE for this article, skip priorities 2 and 3**
- On failure: log `⚠ PDF download failed: {error}` — continue to Priority 2

---

### PRIORITY 2 — Web fetch (acceptable: may be AI-processed by WebFetch tool)

Fetch the article URL using WebFetch with this exact prompt:
```
Do not summarize, paraphrase, restructure, or omit anything. Copy every single word, heading, paragraph, quote, list item, and code block from the article body EXACTLY as it appears — word for word. This is an archiving task; fidelity to the original is paramount. If the article is truncated by a paywall, copy everything visible up to the cutoff point and stop. Do not add commentary.
```

**Paywall detection** — the fetch is paywalled/blocked if the response:
- Contains phrases like "subscribe to read", "this post is for paid subscribers", "sign in to read", "unlock this post"
- Is fewer than ~200 words of article content
- Ends abruptly at a "Subscribe" button section

**If readable** (substantial article content returned, no paywall wall):
- Save to `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}.md`
- File format:
  ```
  # {Title}
  
  Source: {url}
  From: {sender}
  Fetched: {date}
  Content: WebFetch (verbatim article text below)
  
  ---
  
  {full article text as returned by WebFetch}
  ```
- On success: log `✅ Saved (web fetch): {path}` — **DONE for this article, skip priority 3**

**If paywalled or blocked:**
- Log: `⚠ Web fetch paywalled: "{title}" — continuing to email body fallback`
- Continue to Priority 3

**If WebFetch errors:**
- Log: `⚠ Web fetch error: {error}` — continue to Priority 3

---

### PRIORITY 3 — Email body (good: original newsletter content, verbatim)

Newsletter emails contain the full article text in the email body itself. Save it directly.

- Save to `{ARTICLE_FOLDER}/{YYYY-MM-DD}_{sanitized_title}_email.md`
- File format:
  ```
  # {Title}
  
  Source: {url}
  From: {sender}
  Fetched: {date}
  Content: Email body (original newsletter content — verbatim)
  Note: Links may be tracking redirects; canonical URL is above.
  
  ---
  
  {full email body text, verbatim}
  ```
- Strip the following email artifacts (they are not article content):
  - Podcast/video stream header: lines like "Stream the latest episode / Listen and watch now on YouTube, Spotify, Apple"
  - Sponsor / advertisement blocks: sections starting with "Brought to you by" or "Brought to You by" and all bullet points under them until the next article heading
  - Unsubscribe footer: everything from the "Unsubscribe" line to the end of the email
- Keep everything else verbatim: article body, quotes, lists, timestamps, references section, author notes
- On success: log `✅ Saved (email body): {path}`

**If email body is empty or too short (< 100 words):**
- Log: `⚠ No usable content for: "{title}" — skipped`

---

## STEP 5 — Report

Output this block after all processing is complete:

```
────────────────────────────────────────
📚 ARTICLE TRACKER — {date}
Senders: {list} | Lookback: {days} days | Emails scanned: {N}
────────────────────────────────────────
NEW ARTICLES:

{#}. {Title}
   From   : {sender}
   Status : {PDF saved: {path} | Saved (web): {path} | Saved (email): {path} | Paywalled — no content | Skipped}
   Source : {Original PDF | WebFetch (may be AI-processed) | Email body (original)}
   Link   : {url}
   → Say: "explain me {title}"

────────────────────────────────────────
{M} PDF  {N} web fetch  {K} email body  {J} skipped/paywalled

Errors:
{each ⚠ log line, or "None"}
────────────────────────────────────────
```

Always print this block — even when zero articles are found.

---

## CONSTRAINTS

**No hallucination:** Every title, URL, and path must come from an actual tool call response in this session. If a value was not returned by a tool, do not fill it in.

**Fail loudly:** Gmail MCP failure in Step 1 = hard stop. All other errors = log exact text, skip item, continue. Nothing is silently ignored.

**Read-only Gmail:** No send, reply, forward, archive, label, or delete. Search and read only.

**File safety:** Never overwrite existing files. Never delete. Filenames: lowercase, spaces → underscores, no special characters.

**Scope:** Do not summarize or explain any article content. Find, save, notify — stop there.

**Content fidelity:** PDF is original. Email body is original. WebFetch output is AI-processed by an internal model and may paraphrase — always prefer PDF or email body when fidelity matters. Label all saved files with their content source.
