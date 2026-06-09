# MacBook + Chrome Debug + Playwright 定时爬取 X For You 帖子方案

> 适用场景：  
> - MacBook  
> - 使用已登录的 Chrome Debug 浏览器  
> - 定时抓取 X / Twitter For You 最新帖子  
> - 保存为 Markdown  
> - 使用 tweet ID 防重复  
>
> 请确保你的使用符合 X 的服务条款和当地法律。

---

## 1. 推荐技术栈

- **浏览器自动化**：Playwright，优先推荐；也可用 Selenium + remote debugging
- **去重方式**：使用 tweet ID，最可靠
- **定时运行**：macOS `launchd`，比 `cron` 更稳定
- **本地存储**：
  - `seen_tweets.json`：记录已保存的 tweet ID
  - `X_ForYou/日期/tweet_id.md`：每条推文保存为单独 Markdown 文件

---

## 2. 安装依赖

```bash
pip install playwright
playwright install chromium
```

---

## 3. 启动 Chrome Debug

先关闭普通 Chrome，然后用下面命令启动一个独立调试窗口：

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/ChromeDebugProfile"
```

在这个 Chrome 里登录 X，然后打开：

```text
https://x.com/home
```

确认已经进入 For You 页面。

---

## 4. 完整 Python 脚本

保存为：

```text
x_for_you_scraper.py
```

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
X / Twitter For You scraper with Playwright CDP.

Features:
- Connects to an already-open Chrome via remote debugging port 9222
- Scrapes visible For You tweets
- Uses tweet ID for deduplication
- Saves each tweet as Markdown
- Persists seen tweet IDs to JSON
- Supports incremental scrolling
- Writes logs to file and console

Usage:
    python3 x_for_you_scraper.py

Before running:
    Start Chrome with:

    /Applications/Google\\ Chrome.app/Contents/MacOS/Google\\ Chrome \\
      --remote-debugging-port=9222 \\
      --user-data-dir="$HOME/ChromeDebugProfile"

Then login to X manually in that Chrome window.
"""

from __future__ import annotations

import argparse
import json
import logging
import random
import re
import sys
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Optional

from playwright.sync_api import (
    sync_playwright,
    Page,
    Browser,
    BrowserContext,
    TimeoutError as PlaywrightTimeoutError,
)


# =========================
# Default configuration
# =========================

DEFAULT_CDP_URL = "http://localhost:9222"
DEFAULT_OUTPUT_DIR = Path("X_ForYou")
DEFAULT_SEEN_FILE = Path("seen_tweets.json")
DEFAULT_LOG_FILE = Path("x_for_you_scraper.log")

DEFAULT_MAX_NEW_TWEETS = 100
DEFAULT_MAX_SCROLLS = 80
DEFAULT_SCROLL_PAUSE_MIN = 1.5
DEFAULT_SCROLL_PAUSE_MAX = 3.5


# =========================
# Data model
# =========================

@dataclass
class Tweet:
    id: str
    url: str
    text: str
    author: str
    timestamp: str
    scraped_at: str


# =========================
# Logging
# =========================

def setup_logging(log_file: Path) -> None:
    log_file.parent.mkdir(parents=True, exist_ok=True)

    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s | %(levelname)s | %(message)s",
        handlers=[
            logging.StreamHandler(sys.stdout),
            logging.FileHandler(log_file, encoding="utf-8"),
        ],
    )


# =========================
# Persistence
# =========================

def load_seen(seen_file: Path) -> set[str]:
    if not seen_file.exists():
        return set()

    try:
        data = json.loads(seen_file.read_text(encoding="utf-8"))
        if isinstance(data, list):
            return set(str(x) for x in data)
        logging.warning("seen file is not a list; starting with empty seen set")
        return set()
    except Exception as exc:
        logging.warning("failed to read seen file %s: %s", seen_file, exc)
        return set()


def save_seen(seen_file: Path, seen: set[str]) -> None:
    seen_file.parent.mkdir(parents=True, exist_ok=True)
    tmp_file = seen_file.with_suffix(".tmp")
    tmp_file.write_text(
        json.dumps(sorted(seen), ensure_ascii=False, indent=2),
        encoding="utf-8",
    )
    tmp_file.replace(seen_file)


def sanitize_filename(value: str) -> str:
    value = value.strip()
    value = re.sub(r"[^\w\-_.]", "_", value)
    return value[:180] or "unknown"


def markdown_escape(text: str) -> str:
    # 不做过度转义，保留原文排版。
    return text.replace("\r\n", "\n").replace("\r", "\n").strip()


def save_tweet_as_markdown(tweet: Tweet, output_dir: Path) -> Path:
    date = tweet.timestamp[:10] if tweet.timestamp else "unknown-date"
    date = sanitize_filename(date)

    day_dir = output_dir / date
    day_dir.mkdir(parents=True, exist_ok=True)

    md_path = day_dir / f"{sanitize_filename(tweet.id)}.md"

    if md_path.exists():
        return md_path

    content = f"""# {markdown_escape(tweet.author) or "Unknown author"}

- Tweet ID: `{tweet.id}`
- Time: {tweet.timestamp or "Unknown"}
- URL: {tweet.url}
- Scraped at: {tweet.scraped_at}

---

{markdown_escape(tweet.text) or "_No text content found._"}
"""

    md_path.write_text(content, encoding="utf-8")
    return md_path


# =========================
# Browser / page helpers
# =========================

def connect_to_chrome(cdp_url: str) -> tuple[Browser, BrowserContext, Page]:
    playwright = sync_playwright().start()

    try:
        browser = playwright.chromium.connect_over_cdp(cdp_url)
    except Exception:
        playwright.stop()
        raise

    if not browser.contexts:
        browser.close()
        playwright.stop()
        raise RuntimeError("No Chrome context found. Is Chrome opened with remote debugging?")

    context = browser.contexts[0]

    if context.pages:
        page = context.pages[0]
    else:
        page = context.new_page()

    # Store playwright instance on browser object so it does not get garbage-collected.
    # We close it manually in main().
    browser._playwright_instance = playwright  # type: ignore[attr-defined]

    return browser, context, page


def open_for_you(page: Page) -> None:
    current_url = page.url or ""

    if "x.com" not in current_url and "twitter.com" not in current_url:
        logging.info("opening https://x.com/home")
        page.goto("https://x.com/home", wait_until="domcontentloaded", timeout=60_000)

    try:
        page.wait_for_selector('article[data-testid="tweet"]', timeout=30_000)
    except PlaywrightTimeoutError:
        logging.warning(
            "No tweet articles found. Make sure you are logged in and on the For You page."
        )


def human_pause(min_seconds: float, max_seconds: float) -> None:
    time.sleep(random.uniform(min_seconds, max_seconds))


def scroll_page(page: Page) -> None:
    """
    Gradual scrolling to trigger virtualized feed loading.
    """
    page.evaluate(
        """
        () => {
            const distance = Math.floor(window.innerHeight * (0.75 + Math.random() * 0.6));
            window.scrollBy({ top: distance, left: 0, behavior: "smooth" });
        }
        """
    )


def get_scroll_height(page: Page) -> int:
    try:
        return int(page.evaluate("() => document.body.scrollHeight"))
    except Exception:
        return 0


# =========================
# Tweet extraction
# =========================

def extract_status_id_from_href(href: str) -> Optional[str]:
    """
    Extract tweet/status ID from href like:
        /username/status/1234567890
        https://x.com/username/status/1234567890?s=20
    """
    if not href:
        return None

    match = re.search(r"/status/(\d+)", href)
    if not match:
        return None

    return match.group(1)


def normalize_tweet_url(href: str) -> str:
    if href.startswith("http://") or href.startswith("https://"):
        return href.split("?")[0]
    return "https://x.com" + href.split("?")[0]


def extract_tweets(page: Page) -> list[Tweet]:
    tweets: list[Tweet] = []

    articles = page.query_selector_all('article[data-testid="tweet"]')

    for article in articles:
        try:
            links = article.query_selector_all('a[href*="/status/"]')

            status_href = ""
            tweet_id = None

            for link in links:
                href = link.get_attribute("href") or ""
                extracted_id = extract_status_id_from_href(href)
                if extracted_id:
                    status_href = href
                    tweet_id = extracted_id
                    break

            if not tweet_id:
                continue

            text_el = article.query_selector('[data-testid="tweetText"]')
            text = text_el.inner_text(timeout=2_000).strip() if text_el else ""

            time_el = article.query_selector("time")
            timestamp = time_el.get_attribute("datetime") if time_el else ""

            user_el = article.query_selector('[data-testid="User-Name"]')
            author = user_el.inner_text(timeout=2_000).strip() if user_el else ""

            tweet = Tweet(
                id=tweet_id,
                url=normalize_tweet_url(status_href),
                text=text,
                author=author,
                timestamp=timestamp or "",
                scraped_at=time.strftime("%Y-%m-%dT%H:%M:%S%z"),
            )

            tweets.append(tweet)

        except Exception as exc:
            logging.debug("failed to parse one tweet article: %s", exc)
            continue

    return tweets


# =========================
# Main scraping loop
# =========================

def scrape_for_you(
    page: Page,
    output_dir: Path,
    seen_file: Path,
    max_new_tweets: int,
    max_scrolls: int,
    pause_min: float,
    pause_max: float,
) -> int:
    seen = load_seen(seen_file)
    logging.info("loaded %d seen tweet IDs", len(seen))

    new_saved = 0
    unchanged_rounds = 0
    last_seen_count = len(seen)
    last_height = get_scroll_height(page)

    for scroll_index in range(1, max_scrolls + 1):
        visible_tweets = extract_tweets(page)
        logging.info(
            "scroll %d/%d | visible tweets: %d",
            scroll_index,
            max_scrolls,
            len(visible_tweets),
        )

        for tweet in visible_tweets:
            if tweet.id in seen:
                continue

            md_path = save_tweet_as_markdown(tweet, output_dir)
            seen.add(tweet.id)
            new_saved += 1

            logging.info("saved new tweet %s -> %s", tweet.id, md_path)

            if new_saved >= max_new_tweets:
                save_seen(seen_file, seen)
                logging.info("reached max_new_tweets=%d", max_new_tweets)
                return new_saved

        save_seen(seen_file, seen)

        current_seen_count = len(seen)
        current_height = get_scroll_height(page)

        if current_seen_count == last_seen_count and current_height == last_height:
            unchanged_rounds += 1
        else:
            unchanged_rounds = 0

        last_seen_count = current_seen_count
        last_height = current_height

        if unchanged_rounds >= 6:
            logging.info("no new content detected after several rounds; stopping")
            break

        scroll_page(page)
        human_pause(pause_min, pause_max)

    save_seen(seen_file, seen)
    return new_saved


# =========================
# CLI
# =========================

def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Scrape X For You tweets via Playwright CDP and save as Markdown."
    )

    parser.add_argument(
        "--cdp-url",
        default=DEFAULT_CDP_URL,
        help=f"Chrome DevTools Protocol URL. Default: {DEFAULT_CDP_URL}",
    )

    parser.add_argument(
        "--output-dir",
        type=Path,
        default=DEFAULT_OUTPUT_DIR,
        help=f"Output directory. Default: {DEFAULT_OUTPUT_DIR}",
    )

    parser.add_argument(
        "--seen-file",
        type=Path,
        default=DEFAULT_SEEN_FILE,
        help=f"Seen tweet ID JSON file. Default: {DEFAULT_SEEN_FILE}",
    )

    parser.add_argument(
        "--log-file",
        type=Path,
        default=DEFAULT_LOG_FILE,
        help=f"Log file. Default: {DEFAULT_LOG_FILE}",
    )

    parser.add_argument(
        "--max-new-tweets",
        type=int,
        default=DEFAULT_MAX_NEW_TWEETS,
        help=f"Maximum number of new tweets to save. Default: {DEFAULT_MAX_NEW_TWEETS}",
    )

    parser.add_argument(
        "--max-scrolls",
        type=int,
        default=DEFAULT_MAX_SCROLLS,
        help=f"Maximum scroll rounds. Default: {DEFAULT_MAX_SCROLLS}",
    )

    parser.add_argument(
        "--pause-min",
        type=float,
        default=DEFAULT_SCROLL_PAUSE_MIN,
        help=f"Minimum pause between scrolls. Default: {DEFAULT_SCROLL_PAUSE_MIN}",
    )

    parser.add_argument(
        "--pause-max",
        type=float,
        default=DEFAULT_SCROLL_PAUSE_MAX,
        help=f"Maximum pause between scrolls. Default: {DEFAULT_SCROLL_PAUSE_MAX}",
    )

    return parser.parse_args()


def main() -> int:
    args = parse_args()
    setup_logging(args.log_file)

    browser: Optional[Browser] = None

    try:
        logging.info("connecting to Chrome: %s", args.cdp_url)
        browser, context, page = connect_to_chrome(args.cdp_url)

        open_for_you(page)

        logging.info("starting scrape")
        count = scrape_for_you(
            page=page,
            output_dir=args.output_dir,
            seen_file=args.seen_file,
            max_new_tweets=args.max_new_tweets,
            max_scrolls=args.max_scrolls,
            pause_min=args.pause_min,
            pause_max=args.pause_max,
        )

        logging.info("done. saved %d new tweets", count)
        return 0

    except KeyboardInterrupt:
        logging.warning("interrupted by user")
        return 130

    except Exception as exc:
        logging.exception("scraper failed: %s", exc)
        return 1

    finally:
        if browser is not None:
            try:
                playwright = getattr(browser, "_playwright_instance", None)
                browser.close()
                if playwright:
                    playwright.stop()
            except Exception:
                pass


if __name__ == "__main__":
    raise SystemExit(main())
```

---

## 5. 运行方式

基础运行：

```bash
python3 x_for_you_scraper.py
```

指定最多保存 200 条新推文：

```bash
python3 x_for_you_scraper.py --max-new-tweets 200
```

指定输出目录：

```bash
python3 x_for_you_scraper.py --output-dir "$HOME/Documents/X_ForYou"
```

指定日志文件：

```bash
python3 x_for_you_scraper.py \
  --output-dir "$HOME/Documents/X_ForYou" \
  --log-file "$HOME/Documents/X_ForYou/x_for_you_scraper.log"
```

---

## 6. 定时运行：launchd 示例

创建文件：

```bash
nano ~/Library/LaunchAgents/com.x.foryou.scraper.plist
```

内容示例，记得把路径改成你自己的：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.x.foryou.scraper</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/Users/YOUR_NAME/path/to/x_for_you_scraper.py</string>
        <string>--output-dir</string>
        <string>/Users/YOUR_NAME/Documents/X_ForYou</string>
        <string>--max-new-tweets</string>
        <string>100</string>
    </array>

    <key>StartInterval</key>
    <integer>3600</integer>

    <key>RunAtLoad</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/x_foryou_scraper.out.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/x_foryou_scraper.err.log</string>
</dict>
</plist>
```

加载：

```bash
launchctl load ~/Library/LaunchAgents/com.x.foryou.scraper.plist
```

卸载：

```bash
launchctl unload ~/Library/LaunchAgents/com.x.foryou.scraper.plist
```

查看日志：

```bash
tail -f /tmp/x_foryou_scraper.out.log
tail -f /tmp/x_foryou_scraper.err.log
```

---

## 7. 推荐目录结构

运行后会生成类似结构：

```text
X_ForYou/
  2026-06-02/
    1234567890123456789.md
    1234567890123456790.md
  2026-06-03/
    1234567890123456791.md

seen_tweets.json
x_for_you_scraper.log
```

每个 Markdown 文件大概是：

```md
# Author Name

- Tweet ID: `1234567890123456789`
- Time: 2026-06-02T03:12:00.000Z
- URL: https://x.com/user/status/1234567890123456789
- Scraped at: 2026-06-02T12:00:01+0800

---

Tweet content here.
```

---

## 8. 常见问题

### 8.1 提示连接失败

如果出现类似：

```text
ECONNREFUSED http://localhost:9222
```

说明 Chrome Debug 没有启动成功。重新用下面命令启动：

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/ChromeDebugProfile"
```

### 8.2 找不到推文

可能原因：

1. 还没有登录 X。
2. 页面不在 `https://x.com/home`。
3. X 页面结构变了。
4. 网络加载太慢。

可以先手动打开 X 首页，确认 For You 已经正常显示，再运行脚本。

### 8.3 想保存图片

可以在 `extract_tweets()` 里继续提取：

```python
imgs = article.query_selector_all('img[src*="twimg"]')
```

然后把图片 URL 写入 Markdown，或者用 `requests` 下载到本地。

### 8.4 想每小时自动运行

推荐用上面的 `launchd`。  
运行前请确保 Chrome Debug 窗口保持打开，并且账号处于登录状态。

---

## 9. 注意事项

- For You 是动态信息流，内容可能会随时间变化。
- `data-testid` 相对稳定，但 X 页面结构可能更新，脚本需要相应调整。
- 建议降低运行频率，避免对网站造成过多请求。
- 请遵守 X 的服务条款、robots 规则和当地法律。
