# Adding New Platforms to opencli-rs

This guide walks you through creating a YAML adapter so that `opencli-rs` can work with any new website or desktop app. No Rust knowledge required — adapters are plain YAML files you drop into a folder.

---

## Overview: Three Platform Modes

Choose the mode that matches how the target platform works:

| Mode | When to use | Requires |
|------|-------------|----------|
| **Public** | The site exposes a public JSON/RSS API — no login needed | Nothing extra |
| **Browser** | The site requires login or renders content dynamically via JavaScript | Chrome + opencli-rs extension |
| **Desktop** | The target is a desktop app (Cursor, Notion, ChatGPT, etc.) | The app must be running |

---

## Step 0: Discover Before You Build

Before writing a single line of YAML, run these commands to learn about the target site. They can save you a lot of manual work.

```bash
# 1. Check if the platform is already supported
opencli-rs <site> --help
#    → If this prints a help page, the platform is already built in.
#    → If this returns an error, proceed below.

# 2. Let opencli-rs scan the site's structure for you
opencli-rs explore https://example.com
#    → Prints DOM patterns, API endpoints, and authentication hints it finds.

# 3. Auto-detect the authentication strategy
opencli-rs cascade https://example.com
#    → Tells you whether the site uses cookies, tokens, headers, or public access.

# 4. Attempt fully automatic adapter generation
opencli-rs generate https://example.com --goal "get trending posts"
#    → If this succeeds, you're done! The adapter is written for you.
#    → If this fails or the output is wrong, continue with the manual steps below.
```

---

## Step 1: Choose Your Mode

Use this decision tree:

```
Does the site have a public JSON or RSS API?
  YES → Use strategy: public  (simplest, fastest)
  NO  ↓

Does it require a login or use JavaScript to render content?
  YES → Use strategy: browser
  NO  ↓

Is the target a desktop application?
  YES → Use strategy: desktop
```

---

## Step 2: Write the YAML Adapter

Adapters live at:

```
~/.opencli-rs/adapters/<site>/<command>.yaml
```

For example, a `hot` command for a site called `devnews` goes in:

```
~/.opencli-rs/adapters/devnews/hot.yaml
```

### Full YAML Field Reference

```yaml
# ── Core identity ──────────────────────────────────────────────────
site: mysite           # The CLI keyword: opencli-rs mysite <command>
                       # Use lowercase; hyphens are fine (e.g. "yahoo-finance")

name: hot              # The subcommand name: opencli-rs mysite hot
                       # Common names: hot, trending, feed, search, post, profile

description: Get trending posts from MySite
                       # Shown in --help output

domain: mysite.com     # The site's base domain, used for routing

# ── Execution mode ─────────────────────────────────────────────────
strategy: browser      # "public"  = HTTP/API, no browser needed
                       # "browser" = Chrome + opencli-rs extension required
                       # "desktop" = controls a running desktop app

browser: true          # true  = needs Chrome extension
                       # false = pure HTTP (set false when strategy is "public")

# ── Timing & throttling ────────────────────────────────────────────
timeout: 30            # Seconds to wait before the pipeline is considered failed.
                       # Default is usually 30. Increase for slow-loading pages.

rate_limit:
  delay_ms: 500        # Milliseconds to pause between paginated requests.
                       # Use this to avoid hitting rate limits on public APIs
                       # or triggering bot detection on browser-mode sites.

# ── Pagination ─────────────────────────────────────────────────────
pagination:
  type: page           # How the site pages through results:
                       #   "page"   → ?page=1, ?page=2, ...
                       #   "offset" → ?offset=0, ?offset=20, ...
                       #   "cursor" → a token returned in the response
  param: page          # The query-string parameter name (for type: page/offset)
                       # or the JSON field containing the next cursor (for type: cursor)
  max_pages: 3         # Stop after this many pages even if more exist

# ── CLI arguments ──────────────────────────────────────────────────
args:
  query:
    type: string
    required: true
    description: Search term
  limit:
    type: int
    default: 20
    description: Number of results to return
  # Supported types: string, int, bool, float

# ── Execution pipeline ─────────────────────────────────────────────
pipeline:
  # See per-mode pipeline steps below

# ── Output columns ─────────────────────────────────────────────────
columns: [rank, title, author, score, url]
                       # These are the keys returned by your pipeline's last step.
                       # They become the column headers in table output.
```

---

## Step 3: Pipeline Steps by Mode

The `pipeline` field is a list of steps executed in order.

### Public Mode Pipeline

Use `fetch` to call an HTTP endpoint and `parse_json` to extract the data.

```yaml
pipeline:
  - fetch: https://api.mysite.com/v1/hot?limit=${{ args.limit }}
  # Reference any arg with ${{ args.<name> }}

  - parse_json: $.data.posts
  # Use a dot-path to navigate the JSON response.
  # $.data.posts → response["data"]["posts"]
  # $.items      → response["items"]
  # $.            → the whole response (if it's already an array)
```

### Browser Mode Pipeline

Use `navigate` to open a page, `wait_for` to block until content loads, and `evaluate` to run JavaScript that extracts data.

```yaml
pipeline:
  - navigate: https://mysite.com/trending

  - wait_for: "[data-test='post-item']"
  # CSS selector. The pipeline blocks here until the element appears.
  # You can also wait_for a URL pattern: "https://mysite.com/trending"

  - scroll: 3
  # Scroll down this many times before extracting.
  # Useful for infinite-scroll pages that load content lazily.

  - evaluate: |
      (async () => {
        const limit = ${{ args.limit }};
        const seen = new Set();
        const results = [];

        for (const el of document.querySelectorAll('[data-test="post-item"]')) {
          const title = el.querySelector('[data-test="title"]')?.textContent?.trim();
          if (!title || seen.has(title)) continue;
          seen.add(title);

          results.push({
            title,
            author: el.querySelector('[data-test="author"]')?.textContent?.trim(),
            score: el.querySelector('[data-test="score"]')?.textContent?.trim(),
            url: el.querySelector('a[data-test="link"]')?.href,
          });

          if (results.length >= limit) break;
        }
        return results;
      })()
  # The evaluate step must return an array of plain objects.
  # The keys you use here must match your `columns` list.
```

For **write operations** (posting, liking, commenting), add steps after `navigate`:

```yaml
pipeline:
  - navigate: https://mysite.com/compose
  - wait_for: "[data-test='compose-textarea']"
  - fill:
      selector: "[data-test='compose-textarea']"
      value: ${{ args.text }}
  - click: "[data-test='submit-button']"
  - wait_for: "[data-test='success-message']"
```

### Desktop Mode Pipeline

Desktop mode controls a running desktop app. The YAML structure is the same; the pipeline steps use app-focused actions. Look at existing desktop adapters for reference:

```bash
opencli-rs cursor --help
opencli-rs notion --help
```

---

## Step 4: Writing a Good JavaScript Extractor

The `evaluate` step runs inside your Chrome tab. Here are the patterns that work reliably across many sites:

**Choose selectors in this order of preference:**

1. `data-test="..."` or `data-testid="..."` — most stable, rarely change
2. `data-*` attributes with semantic meaning (e.g., `data-post-id`)
3. Semantic class names (e.g., `.post-title`, `.author-name`) — avoid long generated hashes like `.sc-abc123`
4. Element structure (e.g., `article h2 a`) — fragile, use as a last resort

**Standard extractor skeleton:**

```javascript
(async () => {
  const limit = ${{ args.limit }};
  const seen = new Set();   // prevents duplicate entries
  const results = [];

  for (const el of document.querySelectorAll('<your-item-selector>')) {
    // Extract each field
    const title = el.querySelector('<title-selector>')?.textContent?.trim();
    const url   = el.querySelector('a')?.href;

    // Skip empty or duplicate items
    if (!title || seen.has(title)) continue;
    seen.add(title);

    results.push({ title, url });  // keys must match `columns`
    if (results.length >= limit) break;
  }

  return results;
})()
```

**Locating sibling elements** (a tagline next to a title):

```javascript
const titleEl = el.querySelector('[data-test="title"]');
const tagline = titleEl?.parentElement?.querySelector('[data-test="tagline"]')
             ?? titleEl?.nextElementSibling?.textContent?.trim();
```

**Extracting a number from text** (e.g., "1,234 points" → 1234):

```javascript
const score = parseInt(el.querySelector('.score')?.textContent?.replace(/,/g, '') ?? '0');
```

---

## Step 5: Place and Test

1. **Place the file:**

   ```bash
   mkdir -p ~/.opencli-rs/adapters/mysite
   # write your YAML to:
   ~/.opencli-rs/adapters/mysite/hot.yaml
   ```

2. **Verify it loads:**

   ```bash
   opencli-rs mysite --help
   # Should print your command and description.
   ```

3. **Run it:**

   ```bash
   opencli-rs mysite hot --format json
   # Check: all columns present, no null values, correct count.
   ```

4. **Diagnose problems:**

   ```bash
   opencli-rs doctor
   # Checks Chrome extension status, binary version, config paths.
   ```

---

## Step 6: Register the Platform in SKILL.md

After your adapter works, teach your AI agent about it by updating `SKILL.md` in this repository.

**a. Add the site name to the `description:` list in the front matter:**

```yaml
---
name: opencli-rs
description: |
  Use opencli-rs CLI to interact with social/content websites (HackerNews, ...,
  MySite         ← add here
  ...) via the user's Chrome login session.
```

**b. Add a command table in the correct mode section:**

```markdown
### MySite

| Command | Args | Description |
|---------|------|-------------|
| `mysite hot` | `--limit N` (default 20) | Trending posts |
| `mysite search` | `--query <str>`, `--limit N` | Search posts |
```

**c. Add a quick example:**

```markdown
opencli-rs mysite hot --limit 10 --format json
```

---

## Worked Examples

### Example A: Public Mode — News Aggregator with a JSON API

The site exposes `GET /api/v1/stories?limit=N&type=top` with no authentication.

```yaml
site: devnews
name: top
description: Top stories from DevNews
domain: devnews.example.com
strategy: public
browser: false
timeout: 15

args:
  limit:
    type: int
    default: 20
    description: Number of stories

pipeline:
  - fetch: https://devnews.example.com/api/v1/stories?type=top&limit=${{ args.limit }}
  - parse_json: $.stories

columns: [rank, title, points, comments, url]
```

And a paginated variant that fetches multiple pages:

```yaml
site: devnews
name: top
description: Top stories from DevNews (paginated)
domain: devnews.example.com
strategy: public
browser: false
timeout: 15

args:
  limit:
    type: int
    default: 60
    description: Total number of stories across pages

pagination:
  type: page
  param: page
  max_pages: 3

rate_limit:
  delay_ms: 300

pipeline:
  - fetch: https://devnews.example.com/api/v1/stories?type=top&limit=20&page=${{ pagination.page }}
  - parse_json: $.stories

columns: [rank, title, points, comments, url]
```

---

### Example B: Browser Mode — Reading a Social Feed

The site requires login and renders the feed via JavaScript.

```yaml
site: mysocial
name: feed
description: Get your home feed from MySocial
domain: mysocial.example.com
strategy: browser
browser: true
timeout: 30

args:
  limit:
    type: int
    default: 20
    description: Number of posts

pipeline:
  - navigate: https://mysocial.example.com/home
  - wait_for: "[data-test='post-card']"
  - evaluate: |
      (async () => {
        const limit = ${{ args.limit }};
        const seen = new Set();
        const results = [];

        for (const el of document.querySelectorAll('[data-test="post-card"]')) {
          const text = el.querySelector('[data-test="post-text"]')?.textContent?.trim();
          if (!text || seen.has(text)) continue;
          seen.add(text);

          results.push({
            text,
            author:  el.querySelector('[data-test="author-name"]')?.textContent?.trim(),
            likes:   el.querySelector('[data-test="like-count"]')?.textContent?.trim(),
            url:     el.querySelector('a[data-test="post-link"]')?.href,
          });

          if (results.length >= limit) break;
        }
        return results;
      })()

columns: [text, author, likes, url]
```

---

### Example C: Browser Mode — Write Operation (Posting)

Always show the user what will be posted before executing write operations.

```yaml
site: mysocial
name: post
description: Create a new post on MySocial
domain: mysocial.example.com
strategy: browser
browser: true
timeout: 30

args:
  text:
    type: string
    required: true
    description: Post content

pipeline:
  - navigate: https://mysocial.example.com/compose
  - wait_for: "[data-test='compose-textarea']"
  - fill:
      selector: "[data-test='compose-textarea']"
      value: ${{ args.text }}
  - click: "[data-test='submit-button']"
  - wait_for: "[data-test='success-toast']"

columns: [status]
```

---

### Example D: Browser Mode — Search

A search command that navigates to the results page for a user-supplied query.

```yaml
site: mysocial
name: search
description: Search MySocial posts
domain: mysocial.example.com
strategy: browser
browser: true
timeout: 30

args:
  query:
    type: string
    required: true
    description: Search term
  limit:
    type: int
    default: 20

pipeline:
  - navigate: https://mysocial.example.com/search?q=${{ args.query | urlencode }}
  - wait_for: "[data-test='result-item']"
  - evaluate: |
      (async () => {
        const limit = ${{ args.limit }};
        const seen = new Set();
        const results = [];

        for (const el of document.querySelectorAll('[data-test="result-item"]')) {
          const title = el.querySelector('[data-test="result-title"]')?.textContent?.trim();
          if (!title || seen.has(title)) continue;
          seen.add(title);

          results.push({
            title,
            snippet: el.querySelector('[data-test="result-snippet"]')?.textContent?.trim(),
            url:     el.querySelector('a')?.href,
          });

          if (results.length >= limit) break;
        }
        return results;
      })()

columns: [title, snippet, url]
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `error: unknown site` | Adapter not found | Check file path: `~/.opencli-rs/adapters/<site>/<command>.yaml` |
| `timeout after Ns` | Page too slow or selector never appears | Increase `timeout`, check that the selector exists on the live page |
| Output missing columns | JS returns different key names | `console.log` intermediate results, verify keys match `columns` |
| All values are `null` | Selector changed or page not loaded yet | Add/adjust `wait_for`, re-run `opencli-rs explore <url>` |
| Login state not detected | Extension not active | Make sure Chrome is open and logged in; run `opencli-rs doctor` |
| Duplicate rows | No deduplication | Add `seen = new Set()` and skip items already in the set |
| Bot detection / CAPTCHA | Requests too fast | Add `rate_limit.delay_ms`, lower `pagination.max_pages` |
| `generate` produced no output | Complex SPA or auth required | Skip to manual YAML, use `explore` to find the selectors |
