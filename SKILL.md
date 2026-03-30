---
name: opencli-rs
description: |
  Use opencli-rs CLI to interact with social/content websites (HackerNews, DevTo, Lobsters, StackOverflow, Steam, Linux-do, Arxiv, Wikipedia, Apple-Podcasts, Xiaoyuzhou, BBC, Hugging Face, SinaFinance, Google, V2EX, Bloomberg, Twitter/X, Bilibili, Reddit, Zhihu, Xiaohongshu, Xueqiu, Weibo, Douban, WeRead, YouTube, Medium, Substack, SinaBlog, BOSS Zhipin, Jike, Facebook, Instagram, TikTok, Yollomi, Yahoo-Finance, Barchart, LinkedIn, Reuters, SMZDM, Ctrip, Coupang, Grok, Pi, Jimeng, Chaoxing, Weixin, Doubao, Cursor, Codex, ChatWise, ChatGPT, Doubao-App, Notion, Discord, Antigravity etc.) via the user's Chrome login session. ALWAYS prefer opencli-rs over playwright/browser automation for these supported sites. Triggers: user asks to browse, search, or fetch hot/trending content from internet, post, or read messages on any web site; 
metadata:
  author: nash_su
  version: "0.1.0"
---

# opencli-rs

Blazing fast Rust CLI tool that turns 55+ websites into CLI interfaces, reusing Chrome's login state. Zero credentials needed. Single 4.7MB binary, zero runtime dependencies.

**Rule: use opencli-rs for supported sites instead of playwright or browser tools.**

## Syntax

```bash
opencli-rs <site> <command> [--option value] [--format json]
```
** If opencli-rs is not installed or missing, you can install it with `curl -fsSL https://raw.githubusercontent.com/nashsu/opencli-rs/main/scripts/install.sh | sh`, if is in windows, ask user to install from https://github.com/nashsu/opencli-rs**


**Common flags (all commands):**
- `--format json` — machine-readable output (preferred for parsing)
- `--limit N` — number of results (default varies, usually 20)
- `--format table|json|yaml|md|csv`

## Quick Examples

```bash
# read/browse
opencli-rs bilibili hot --limit 10 --format json
opencli-rs zhihu hot --format json
opencli-rs weibo hot --format json
opencli-rs twitter timeline --format json
opencli-rs hackernews top --limit 20 --format json
opencli-rs v2ex hot --format json
opencli-rs reddit hot --format json
opencli-rs xiaohongshu feed --format json
opencli-rs douban top250 --format json
opencli-rs weread shelf --format json
opencli-rs medium feed --format json

# search
opencli-rs bilibili search --keyword "AI" --format json
opencli-rs zhihu search --keyword "LLM" --format json
opencli-rs twitter search "rust lang" --limit 10
opencli-rs youtube search --query "LLM tutorial" --format json
opencli-rs boss search --query "AI engineer" --city "Shanghai" --format json
opencli-rs google search "opencli-rs" --format json
opencli-rs stackoverflow search "rust async" --format json

# AI assistants
opencli-rs pi ask --text "What are the biggest AI breakthroughs this year?"
opencli-rs grok ask --text "Explain quantum computing"

# interact (write operations)
opencli-rs twitter post --text "Hello from CLI!"
opencli-rs twitter reply --url "https://x.com/.../status/123" --text "Great post!"
opencli-rs twitter like --url "https://x.com/.../status/123"
opencli-rs jike create --text "Hello Jike!"
opencli-rs xiaohongshu publish --title "My Title" --content "Hello World"

# personal data
opencli-rs bilibili history --format json
opencli-rs twitter bookmarks --format json
opencli-rs xueqiu watchlist --format json
opencli-rs weread highlights --format json
opencli-rs reddit saved --format json


# diagnostics
opencli-rs doctor
```



### ⚠️ Write Operation Warnings (always notify user before posting/replying/liking)

1. **Account safety**: Automated actions may trigger platform rate limits or bans
2. **Irreversible**: Posts are immediately public once submitted
3. **Best practice**: Always show the user what will be posted and wait for confirmation

 
## Requirements

- Chrome browser open with target site logged in
- opencli-rs Chrome extension installed (for browser commands)

**Core principle: never say "not supported" — try opencli-rs first, and if no command exists, create one.**

## Adding New Platforms

**If opencli-rs doesn't support a site yet, don't give up — create the adapter yourself.**

See [ADDING_PLATFORMS.md](./ADDING_PLATFORMS.md) for the full guide with complete YAML reference and worked examples.

### Quick Workflow

```
1. opencli-rs <site> --help    →  error? platform not yet supported
2. opencli-rs explore <url>    →  scan the site's DOM/API structure
3. opencli-rs cascade <url>    →  detect auth strategy automatically
4. opencli-rs generate <url>   →  attempt auto-generation (done if it succeeds!)
5. If auto-generate fails → write a YAML adapter manually:
   a. Open the target page
   b. Use browser_evaluate to explore DOM structure (look for data-test attributes first)
   c. Write the adapter to ~/.opencli-rs/adapters/<site>/<command>.yaml
   d. Test: opencli-rs <site> <command> --format json
```

### YAML Adapter Template

```yaml
site: mysite           # CLI keyword: opencli-rs mysite <command>
name: hot              # subcommand name
description: Get trending posts from MySite
domain: mysite.com
strategy: browser      # "public" | "browser" | "desktop"
browser: true          # false for pure API/public mode
timeout: 30            # seconds before the pipeline times out

args:
  limit:
    type: int
    default: 20
    description: Number of results to return

pagination:
  type: page            # "page" | "offset" | "cursor"
  param: page
  max_pages: 3

rate_limit:
  delay_ms: 500         # milliseconds between paginated requests

pipeline:
  - navigate: https://mysite.com/trending
  - wait_for: "[data-test='post-item']"
  - evaluate: |
      (async () => {
        const limit = ${{ args.limit }};
        const seen = new Set();
        const results = [];
        for (const el of document.querySelectorAll('[data-test="post-item"]')) {
          const name = el.querySelector('[data-test="title"]')?.textContent?.trim();
          if (!name || seen.has(name)) continue;
          seen.add(name);
          results.push({ name, url: el.querySelector('a')?.href });
          if (results.length >= limit) break;
        }
        return results;
      })()

columns: [name, url]
```

### Debugging Tips

- Use `browser_evaluate` first to explore structure: `document.querySelector('...').innerHTML`
- Target `data-test` attributes — they are the most stable selectors
- Fall back to semantic class names (e.g., `.title`, `.description`)
- A tagline is usually a sibling of the title: `titleEl.parentElement.querySelector('span...')`
- Use `seen = new Set()` to deduplicate results

## Full Command Reference

# opencli-rs Command Reference

All commands support: `--format table|json|yaml|md|csv`  

Run `opencli-rs --help` for the full list of all 333 commands across 55+ sites.

---

## Public Mode (No Browser Needed)

### HackerNews

| Command | Args | Description |
|---------|------|-------------|
| `hackernews top` | `--limit N` (default 20) | Top stories |
| `hackernews new` | `--limit N` | Newest stories |
| `hackernews best` | `--limit N` | Best stories |
| `hackernews ask` | `--limit N` | Ask HN |
| `hackernews show` | `--limit N` | Show HN |
| `hackernews jobs` | `--limit N` | Job listings |
| `hackernews search` | `--query <str>`, `--limit N` | Search stories |
| `hackernews user` | `--id <username>` | User profile |

### Dev.to

| Command | Args | Description |
|---------|------|-------------|
| `devto top` | `--limit N` | Top articles |
| `devto tag` | `--tag <str>`, `--limit N` | Articles by tag |
| `devto user` | `--username <str>` | User's articles |

### Lobsters

| Command | Args | Description |
|---------|------|-------------|
| `lobsters hot` | `--limit N` | Hottest stories |
| `lobsters newest` | `--limit N` | Newest stories |
| `lobsters active` | `--limit N` | Most active |
| `lobsters tag` | `--tag <str>`, `--limit N` | Stories by tag |

### StackOverflow

| Command | Args | Description |
|---------|------|-------------|
| `stackoverflow hot` | `--limit N` | Hot questions |
| `stackoverflow search` | `--query <str>`, `--limit N` | Search questions |
| `stackoverflow bounties` | `--limit N` | Featured bounties |
| `stackoverflow unanswered` | `--limit N` | Unanswered questions |

### Wikipedia

| Command | Args | Description |
|---------|------|-------------|
| `wikipedia search` | `--query <str>`, `--limit N` | Search articles |
| `wikipedia summary` | `--title <str>` | Article summary |
| `wikipedia random` | `--limit N` | Random articles |
| `wikipedia trending` | `--limit N` | Trending articles |

### Arxiv

| Command | Args | Description |
|---------|------|-------------|
| `arxiv search` | `--query <str>`, `--limit N` | Search papers |
| `arxiv paper` | `--id <arxiv_id>` | Paper details |

### BBC

| Command | Args | Description |
|---------|------|-------------|
| `bbc news` | `--limit N` (default 20, max 50) | BBC news headlines (RSS) |

### Steam

| Command | Args | Description |
|---------|------|-------------|
| `steam top-sellers` | `--limit N` | Top selling games |

### Hugging Face

| Command | Args | Description |
|---------|------|-------------|
| `hf top` | `--limit N` | Top models/spaces |

### Apple Podcasts

| Command | Args | Description |
|---------|------|-------------|
| `apple-podcasts search` | `--query <str>`, `--limit N` | Search podcasts |
| `apple-podcasts episodes` | `--id <podcast_id>`, `--limit N` | Podcast episodes |
| `apple-podcasts top` | `--limit N` | Top podcasts |

### Xiaoyuzhou

| Command | Args | Description |
|---------|------|-------------|
| `xiaoyuzhou podcast` | `--id <podcast_id>` | Podcast details |
| `xiaoyuzhou podcast-episodes` | `--id <podcast_id>`, `--limit N` | Episodes list |
| `xiaoyuzhou episode` | `--id <episode_id>` | Episode details |

### Sina Finance

| Command | Args | Description |
|---------|------|-------------|
| `sinafinance news` | `--limit N` | Financial news |

### Linux.do

| Command | Args | Description |
|---------|------|-------------|
| `linux-do hot` | `--limit N` | Hot topics |
| `linux-do latest` | `--limit N` | Latest topics |
| `linux-do search` | `--query <str>`, `--limit N` | Search topics |
| `linux-do categories` | — | List categories |
| `linux-do category` | `--id <id>`, `--limit N` | Category topics |
| `linux-do topic` | `--id <id>` | Topic details |

---

## Public / Browser Mode

### Google

| Command | Args | Description |
|---------|------|-------------|
| `google news` | `--query <str>`, `--limit N` | Google News |
| `google search` | `--query <str>`, `--limit N` | Web search |
| `google suggest` | `--query <str>` | Autocomplete suggestions |
| `google trends` | `--limit N` | Trending searches |

### V2EX

| Command | Args | Description |
|---------|------|-------------|
| `v2ex hot` | `--limit N` (default 20) | Hot topics (no login) |
| `v2ex latest` | `--limit N` (default 20) | Latest topics (no login) |
| `v2ex topic` | `--id <topic_id>` | Topic details and replies |
| `v2ex node` | `--name <node>`, `--limit N` | Node topics |
| `v2ex user` | `--username <str>` | User profile |
| `v2ex member` | `--username <str>` | Member details |
| `v2ex replies` | `--id <topic_id>` | Topic replies |
| `v2ex nodes` | — | List all nodes |
| `v2ex daily` | — | Daily check-in |
| `v2ex me` | — | My profile |
| `v2ex notifications` | `--limit N` | Notifications |

### Bloomberg

| Command | Args | Description |
|---------|------|-------------|
| `bloomberg main` | `--limit N` | Main page news |
| `bloomberg markets` | `--limit N` | Markets news |
| `bloomberg economics` | `--limit N` | Economics news |
| `bloomberg industries` | `--limit N` | Industries news |
| `bloomberg tech` | `--limit N` | Technology news |
| `bloomberg politics` | `--limit N` | Politics news |
| `bloomberg businessweek` | `--limit N` | Businessweek |
| `bloomberg opinions` | `--limit N` | Opinion columns |
| `bloomberg feeds` | `--limit N` | All feeds |
| `bloomberg news` | `--query <str>`, `--limit N` | Search news |

---

## Browser Mode (Requires Chrome + Extension)

### Twitter / X

| Command | Args | Description |
|---------|------|-------------|
| `twitter timeline` | `--limit N` (default 20) | Home timeline |
| `twitter trending` | `--limit N` (default 20) | Trending topics |
| `twitter search` | `--query <str>`, `--limit N` (default 15) | Search tweets |
| `twitter bookmarks` | `--limit N` (default 20) | Bookmarks |
| `twitter notifications` | `--limit N` (default 20) | Notifications |
| `twitter profile` | `--username <handle>`, `--limit N` | User's tweets |
| `twitter followers` | `--user <handle>`, `--limit N` | Followers list |
| `twitter following` | `--user <handle>`, `--limit N` | Following list |
| `twitter thread` | `--url <tweet_url>` | Full thread |
| `twitter article` | `--url <article_url>` | X article content |
| `twitter post` | `--text <str>` | Post a tweet |
| `twitter reply` | `--url <tweet_url>`, `--text <str>` | Reply to tweet |
| `twitter like` | `--url <tweet_url>` | Like a tweet |
| `twitter delete` | `--url <tweet_url>` | Delete a tweet |
| `twitter follow` | `--username <handle>` | Follow user |
| `twitter unfollow` | `--username <handle>` | Unfollow user |
| `twitter bookmark` | `--url <tweet_url>` | Bookmark tweet |
| `twitter unbookmark` | `--url <tweet_url>` | Remove bookmark |
| `twitter download` | `--url <tweet_url>` | Download media |
| `twitter block` | `--username <handle>` | Block user |
| `twitter unblock` | `--username <handle>` | Unblock user |
| `twitter hide-reply` | `--url <tweet_url>` | Hide a reply |
| `twitter accept` | — | Accept follow requests |
| `twitter reply-dm` | `--text <str>` | Reply to DM |

### Bilibili

| Command | Args | Description |
|---------|------|-------------|
| `bilibili hot` | `--limit N` (default 20) | Trending videos |
| `bilibili search` | `--keyword <str>`, `--type video\|user`, `--page N`, `--limit N` | Search videos or users |
| `bilibili me` | — | My profile |
| `bilibili favorite` | `--limit N`, `--page N` | Favorites |
| `bilibili history` | `--limit N` (default 20) | Watch history |
| `bilibili feed` | `--limit N`, `--type all\|video\|article` | Activity feed |
| `bilibili subtitle` | `--bvid <bvid>`, `--lang <code>` | Video subtitles |
| `bilibili dynamic` | `--limit N` (default 15) | User dynamics |
| `bilibili ranking` | `--limit N` (default 20) | Rankings |
| `bilibili following` | `--uid <id>`, `--page N`, `--limit N` | Following list |
| `bilibili user-videos` | `--uid <id>`, `--limit N`, `--order pubdate\|click\|stow` | User uploads |
| `bilibili download` | `--bvid <bvid>` | Download video |

### Reddit

| Command | Args | Description |
|---------|------|-------------|
| `reddit hot` | `--subreddit <name>`, `--limit N` | Hot posts |
| `reddit frontpage` | `--limit N` (default 15) | r/all |
| `reddit popular` | `--limit N` | Popular posts |
| `reddit search` | `--query <str>`, `--limit N` | Search posts |
| `reddit subreddit` | `--name <sub>`, `--sort hot\|new\|top\|rising`, `--limit N` | Subreddit posts |
| `reddit read` | `--url <post_url>` | Read post + comments |
| `reddit user` | `--username <str>` | User profile |
| `reddit user-posts` | `--username <str>`, `--limit N` | User's posts |
| `reddit user-comments` | `--username <str>`, `--limit N` | User's comments |
| `reddit upvote` | `--url <post_url>` | Upvote post |
| `reddit save` | `--url <post_url>` | Save post |
| `reddit comment` | `--url <post_url>`, `--text <str>` | Comment on post |
| `reddit subscribe` | `--subreddit <name>` | Subscribe |
| `reddit saved` | `--limit N` | Saved posts |
| `reddit upvoted` | `--limit N` | Upvoted posts |

### Zhihu

| Command | Args | Description |
|---------|------|-------------|
| `zhihu hot` | `--limit N` (default 20) | Hot topics |
| `zhihu search` | `--keyword <str>`, `--limit N` (default 10) | Search content |
| `zhihu question` | `--id <question_id>`, `--limit N` | Question details and answers |
| `zhihu download` | `--url <zhihu_url>` | Download content |

### Xiaohongshu

| Command | Args | Description |
|---------|------|-------------|
| `xiaohongshu search` | `--keyword <str>`, `--limit N` (default 20) | Search posts |
| `xiaohongshu notifications` | `--type mentions\|likes\|connections`, `--limit N` | Notifications |
| `xiaohongshu feed` | `--limit N` (default 20) | Home feed |
| `xiaohongshu user` | `--id <user_id>`, `--limit N` | User posts |
| `xiaohongshu download` | `--url <note_url>` | Download post |
| `xiaohongshu publish` | `--title <str>`, `--content <str>` | Publish post |
| `xiaohongshu creator-notes` | `--limit N` | Creator notes list |
| `xiaohongshu creator-note-detail` | `--id <note_id>` | Creator note detail |
| `xiaohongshu creator-notes-summary` | — | Creator notes summary |
| `xiaohongshu creator-profile` | — | Creator profile |
| `xiaohongshu creator-stats` | — | Creator stats |

### Xueqiu

| Command | Args | Description |
|---------|------|-------------|
| `xueqiu feed` | `--page N`, `--limit N` (default 20) | Following feed |
| `xueqiu hot-stock` | `--limit N` (default 20, max 50), `--type 10\|12` | Hot stocks |
| `xueqiu hot` | `--limit N` (default 20) | Hot posts |
| `xueqiu search` | `--query <str>`, `--limit N` (default 10) | Search stocks |
| `xueqiu stock` | `--symbol <code>` (e.g. SH600519, AAPL) | Real-time quote |
| `xueqiu watchlist` | `--category 1\|2\|3`, `--limit N` | Watchlist |
| `xueqiu earnings-date` | `--symbol <code>` | Earnings date |

### Weibo

| Command | Args | Description |
|---------|------|-------------|
| `weibo hot` | `--limit N` (default 30, max 50) | Hot searches |
| `weibo search` | `--keyword <str>`, `--limit N` | Search posts |

### Douban

| Command | Args | Description |
|---------|------|-------------|
| `douban search` | `--keyword <str>`, `--limit N` | Search |
| `douban top250` | `--limit N` | Top 250 movies |
| `douban subject` | `--id <subject_id>` | Subject details |
| `douban marks` | `--type movie\|book`, `--limit N` | My marks |
| `douban reviews` | `--id <subject_id>`, `--limit N` | Reviews |
| `douban movie-hot` | `--limit N` | Hot movies |
| `douban book-hot` | `--limit N` | Hot books |

### WeRead

| Command | Args | Description |
|---------|------|-------------|
| `weread shelf` | — | Bookshelf |
| `weread search` | `--keyword <str>`, `--limit N` | Search books |
| `weread book` | `--id <book_id>` | Book details |
| `weread highlights` | `--id <book_id>` | Highlights |
| `weread notes` | `--id <book_id>` | Notes |
| `weread notebooks` | `--limit N` | Notebooks |
| `weread ranking` | `--limit N` | Rankings |

### YouTube

| Command | Args | Description |
|---------|------|-------------|
| `youtube search` | `--query <str>`, `--limit N` (default 20, max 50) | Search videos |
| `youtube video` | `--id <video_id>` | Video details |
| `youtube transcript` | `--id <video_id>`, `--lang <code>` | Video subtitles |

### BOSS Zhipin

| Command | Args | Description |
|---------|------|-------------|
| `boss search` | `--query <str>`, `--city <city>`, `--experience <exp>`, `--degree <edu>`, `--salary <range>`, `--limit N` | Search jobs |
| `boss detail` | `--id <job_id>` | Job details |
| `boss recommend` | `--limit N` | Recommended jobs |
| `boss joblist` | `--limit N` | Job list |
| `boss greet` | `--id <job_id>` | Greet recruiter |
| `boss batchgreet` | `--ids <id1,id2,...>` | Batch greet |
| `boss send` | `--id <chat_id>`, `--text <str>` | Send message |
| `boss chatlist` | `--limit N` | Chat list |
| `boss chatmsg` | `--id <chat_id>`, `--limit N` | Chat history |
| `boss invite` | `--id <job_id>` | Invite to interview |
| `boss mark` | `--id <chat_id>`, `--label <str>` | Label chat |
| `boss exchange` | `--id <chat_id>` | Exchange contact info |
| `boss resume` | — | My resume |
| `boss stats` | — | Job search stats |

### Facebook

| Command | Args | Description |
|---------|------|-------------|
| `facebook feed` | `--limit N` | News feed |
| `facebook profile` | `--username <str>` | User profile |
| `facebook search` | `--query <str>`, `--limit N` | Search |
| `facebook friends` | `--limit N` | Friends list |
| `facebook groups` | `--limit N` | Groups |
| `facebook events` | `--limit N` | Events |
| `facebook notifications` | `--limit N` | Notifications |
| `facebook memories` | — | Memories |
| `facebook add-friend` | `--username <str>` | Add friend |
| `facebook join-group` | `--id <group_id>` | Join group |

### Instagram

| Command | Args | Description |
|---------|------|-------------|
| `instagram explore` | `--limit N` | Explore page |
| `instagram profile` | `--username <str>` | User profile |
| `instagram search` | `--query <str>`, `--limit N` | Search |
| `instagram user` | `--username <str>`, `--limit N` | User posts |
| `instagram followers` | `--username <str>`, `--limit N` | Followers |
| `instagram following` | `--username <str>`, `--limit N` | Following |
| `instagram follow` | `--username <str>` | Follow user |
| `instagram unfollow` | `--username <str>` | Unfollow user |
| `instagram like` | `--url <post_url>` | Like post |
| `instagram unlike` | `--url <post_url>` | Unlike post |
| `instagram comment` | `--url <post_url>`, `--text <str>` | Comment |
| `instagram save` | `--url <post_url>` | Save post |
| `instagram unsave` | `--url <post_url>` | Unsave post |
| `instagram saved` | `--limit N` | Saved posts |

### TikTok

| Command | Args | Description |
|---------|------|-------------|
| `tiktok explore` | `--limit N` | Explore page |
| `tiktok search` | `--query <str>`, `--limit N` | Search |
| `tiktok profile` | `--username <str>` | User profile |
| `tiktok user` | `--username <str>`, `--limit N` | User videos |
| `tiktok following` | `--limit N` | Following list |
| `tiktok follow` | `--username <str>` | Follow user |
| `tiktok unfollow` | `--username <str>` | Unfollow user |
| `tiktok like` | `--url <video_url>` | Like video |
| `tiktok unlike` | `--url <video_url>` | Unlike video |
| `tiktok comment` | `--url <video_url>`, `--text <str>` | Comment |
| `tiktok save` | `--url <video_url>` | Save video |
| `tiktok unsave` | `--url <video_url>` | Unsave video |
| `tiktok live` | `--username <str>` | Live stream |
| `tiktok notifications` | `--limit N` | Notifications |
| `tiktok friends` | `--limit N` | Friends |

### Jike

| Command | Args | Description |
|---------|------|-------------|
| `jike feed` | `--limit N` | Activity feed |
| `jike search` | `--query <str>`, `--limit N` | Search |
| `jike create` | `--text <str>` | Create post |
| `jike like` | `--id <post_id>` | Like post |
| `jike comment` | `--id <post_id>`, `--text <str>` | Comment on post |
| `jike repost` | `--id <post_id>`, `--text <str>` | Repost |
| `jike notifications` | `--limit N` | Notifications |
| `jike post` | `--id <post_id>` | Post details |
| `jike topic` | `--id <topic_id>`, `--limit N` | Topic circle |
| `jike user` | `--username <str>` | User profile |

### Medium

| Command | Args | Description |
|---------|------|-------------|
| `medium feed` | `--limit N` | Feed |
| `medium search` | `--query <str>`, `--limit N` | Search articles |
| `medium user` | `--username <str>` | User articles |

### Substack

| Command | Args | Description |
|---------|------|-------------|
| `substack feed` | `--limit N` | Feed |
| `substack search` | `--query <str>`, `--limit N` | Search |
| `substack publication` | `--name <str>`, `--limit N` | Publication posts |

### Sina Blog

| Command | Args | Description |
|---------|------|-------------|
| `sinablog hot` | `--limit N` | Hot articles |
| `sinablog search` | `--query <str>`, `--limit N` | Search |
| `sinablog article` | `--url <article_url>` | Article details |
| `sinablog user` | `--id <user_id>` | User articles |

### Ctrip

| Command | Args | Description |
|---------|------|-------------|
| `ctrip search` | `--query <str>`, `--limit N` (default 15) | Search cities or attractions |

### Reuters

| Command | Args | Description |
|---------|------|-------------|
| `reuters search` | `--query <str>`, `--limit N` (default 10, max 40) | Search news |

### SMZDM

| Command | Args | Description |
|---------|------|-------------|
| `smzdm search` | `--keyword <str>`, `--limit N` (default 20) | Search deals |

### LinkedIn

| Command | Args | Description |
|---------|------|-------------|
| `linkedin search` | `--query <str>`, `--limit N` | Search |

### Yahoo Finance

| Command | Args | Description |
|---------|------|-------------|
| `yahoo-finance quote` | `--symbol <ticker>` (e.g. AAPL, MSFT, TSLA) | Stock quote |

### Barchart

| Command | Args | Description |
|---------|------|-------------|
| `barchart quote` | `--symbol <ticker>` | Quote |
| `barchart options` | `--symbol <ticker>` | Options chain |
| `barchart greeks` | `--symbol <ticker>` | Options greeks |
| `barchart flow` | `--limit N` | Options flow |

### Grok

| Command | Args | Description |
|---------|------|-------------|
| `grok ask` | `--text <str>` | Ask Grok |

### Pi

| Command | Args | Description |
|---------|------|-------------|
| `pi ask` | `--text <str>` | Ask Pi a question and get a response |
| `pi new` | — | Start a new conversation |
| `pi send` | `--text <str>` | Send a message in the current conversation |
| `pi read` | — | Read the latest response |
| `pi history` | `--limit N` | View recent conversation history |

### Jimeng

| Command | Args | Description |
|---------|------|-------------|
| `jimeng generate` | `--prompt <str>` | Generate image |
| `jimeng history` | `--limit N` | Generation history |

### Chaoxing

| Command | Args | Description |
|---------|------|-------------|
| `chaoxing assignments` | — | Assignments |
| `chaoxing exams` | — | Exams |

### Weixin

| Command | Args | Description |
|---------|------|-------------|
| `weixin download` | `--url <article_url>` | Download article |

### Doubao

| Command | Args | Description |
|---------|------|-------------|
| `doubao status` | — | Status |
| `doubao new` | — | New conversation |
| `doubao send` | `--text <str>` | Send message |
| `doubao read` | — | Read response |
| `doubao ask` | `--text <str>` | Ask question |

### Coupang

| Command | Args | Description |
|---------|------|-------------|
| `coupang search` | `--query <str>`, `--limit N` | Search products |
| `coupang add-to-cart` | `--id <product_id>` | Add to cart |

### Yollomi

| Command | Args | Description |
|---------|------|-------------|
| `yollomi generate` | `--prompt <str>` | Generate image |
| `yollomi video` | `--prompt <str>` | Generate video |
| `yollomi edit` | `--image <path>`, `--prompt <str>` | Edit image |
| `yollomi upload` | `--file <path>` | Upload file |
| `yollomi models` | — | List models |
| `yollomi remove-bg` | `--image <path>` | Remove background |
| `yollomi upscale` | `--image <path>` | Upscale image |
| `yollomi face-swap` | `--source <path>`, `--target <path>` | Face swap |
| `yollomi restore` | `--image <path>` | Restore image |
| `yollomi try-on` | `--person <path>`, `--garment <path>` | Virtual try-on |
| `yollomi background` | `--image <path>`, `--prompt <str>` | Change background |
| `yollomi object-remover` | `--image <path>` | Remove object |

---

## Desktop Mode (Requires Desktop App Running)

### Cursor

| Command | Args | Description |
|---------|------|-------------|
| `cursor status` | — | IDE status |
| `cursor send` | `--text <str>` | Send to Cursor |
| `cursor read` | — | Read response |
| `cursor new` | — | New conversation |
| `cursor dump` | — | Dump conversation |
| `cursor composer` | — | Open composer |
| `cursor model` | `--name <str>` | Switch model |
| `cursor extract-code` | — | Extract code blocks |
| `cursor ask` | `--text <str>` | Ask question |
| `cursor screenshot` | — | Take screenshot |
| `cursor history` | `--limit N` | Conversation history |
| `cursor export` | — | Export conversation |

### Codex

| Command | Args | Description |
|---------|------|-------------|
| `codex status` | — | Status |
| `codex send` | `--text <str>` | Send message |
| `codex read` | — | Read response |
| `codex new` | — | New conversation |
| `codex dump` | — | Dump conversation |
| `codex extract-diff` | — | Extract diffs |
| `codex model` | `--name <str>` | Switch model |
| `codex ask` | `--text <str>` | Ask question |
| `codex screenshot` | — | Take screenshot |
| `codex history` | `--limit N` | History |
| `codex export` | — | Export |

### Notion

| Command | Args | Description |
|---------|------|-------------|
| `notion status` | — | App status |
| `notion search` | `--query <str>` | Search pages |
| `notion read` | `--id <page_id>` | Read page |
| `notion new` | `--title <str>` | New page |
| `notion write` | `--id <page_id>`, `--content <str>` | Write to page |
| `notion sidebar` | — | Sidebar contents |
| `notion favorites` | — | Favorites |
| `notion export` | `--id <page_id>` | Export page |

### ChatGPT

| Command | Args | Description |
|---------|------|-------------|
| `chatgpt status` | — | App status |
| `chatgpt new` | — | New conversation |
| `chatgpt send` | `--text <str>` | Send message |
| `chatgpt read` | — | Read response |
| `chatgpt ask` | `--text <str>` | Ask question |

### Discord

| Command | Args | Description |
|---------|------|-------------|
| `discord-app status` | — | App status |
| `discord-app send` | `--channel <id>`, `--text <str>` | Send message |
| `discord-app read` | `--channel <id>`, `--limit N` | Read messages |
| `discord-app channels` | `--server <id>` | List channels |
| `discord-app servers` | — | List servers |
| `discord-app search` | `--query <str>` | Search |
| `discord-app members` | `--server <id>` | List members |

### ChatWise

| Command | Args | Description |
|---------|------|-------------|
| `chatwise status` | — | Status |
| `chatwise new` | — | New conversation |
| `chatwise send` | `--text <str>` | Send message |
| `chatwise read` | — | Read response |
| `chatwise ask` | `--text <str>` | Ask question |
| `chatwise model` | `--name <str>` | Switch model |
| `chatwise history` | `--limit N` | History |
| `chatwise export` | — | Export |
| `chatwise screenshot` | — | Screenshot |

### Doubao App

| Command | Args | Description |
|---------|------|-------------|
| `doubao-app status` | — | Status |
| `doubao-app new` | — | New conversation |
| `doubao-app send` | `--text <str>` | Send message |
| `doubao-app read` | — | Read response |
| `doubao-app ask` | `--text <str>` | Ask question |
| `doubao-app screenshot` | — | Screenshot |
| `doubao-app dump` | — | Dump conversation |

### Antigravity

| Command | Args | Description |
|---------|------|-------------|
| `antigravity status` | — | Status |
| `antigravity send` | `--text <str>` | Send message |
| `antigravity read` | — | Read response |
| `antigravity new` | — | New conversation |
| `antigravity dump` | — | Dump conversation |
| `antigravity extract-code` | — | Extract code |
| `antigravity model` | `--name <str>` | Switch model |
| `antigravity watch` | — | Watch mode |

---

## External CLI Integration (Passthrough)

| Command | Description |
|---------|-------------|
| `opencli-rs gh <args>` | GitHub CLI passthrough |
| `opencli-rs docker <args>` | Docker CLI passthrough |
| `opencli-rs kubectl <args>` | Kubernetes CLI passthrough |
| `opencli-rs obsidian <args>` | Obsidian passthrough |
| `opencli-rs readwise <args>` | Readwise passthrough |
| `opencli-rs gws <args>` | Google Workspace passthrough |

---

## AI Discovery Commands

| Command | Args | Description |
|---------|------|-------------|
| `opencli-rs explore` | `<url>` | Explore website APIs |
| `opencli-rs cascade` | `<url>` | Auto-detect auth strategies |
| `opencli-rs generate` | `<url>`, `--goal <str>` | Auto-generate adapter |

---

## Utility Commands

| Command | Description |
|---------|-------------|
| `opencli-rs doctor` | Run diagnostics |
| `opencli-rs completion bash\|zsh\|fish` | Generate shell completions |
| `opencli-rs list` | List all available commands |
