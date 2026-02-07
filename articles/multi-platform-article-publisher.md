---
title: "技術記事を4つのプラットフォームに同時投稿するCLIを作った話"
emoji: "🚀"
type: "tech"
topics: ["python", "automation", "techblog", "playwright", "cli"]
published: false
---

# 技術記事を4つのプラットフォームに同時投稿するCLIを作った話

**「Note、Zenn、Qiita、ブログ……毎回コピペするの、もう限界だ。」**

1つのMarkdownファイルから4プラットフォームに同時投稿できるCLIツールを作り、**記事公開にかかる作業時間を90%以上削減**しました。この記事では、その設計と実装を解説します。

---

## 🎯 この記事で得られること

> **この記事を読むと、あなたはマルチプラットフォーム記事投稿システムを自分で構築できるようになります。**
>
> - ✅ 1つのMarkdownから複数プラットフォームに自動投稿する仕組み
> - ✅ Note（Playwright自動化）、Zenn（GitHub連携）、Qiita（REST API）、自作ブログ（Astro）の各実装方法
> - ✅ プラットフォームごとのフォーマット変換ロジック
> - ✅ SNS告知まで含めた完全自動化パイプライン

---

## 😰 あなたもこんな悩みを抱えていませんか？

技術記事を書くエンジニアなら、一度はこう思ったことがあるはずです。

- 「Note、Zenn、Qiita……全部に投稿したいけど、手動コピペが面倒すぎる」
- 「プラットフォームごとにMarkdownの書き方が微妙に違って、毎回修正が必要」
- 「記事を公開した後のSNS告知も、それぞれ手動でやっている」
- 「収益化のチャンスを逃しているかもしれないけど、手が回らない」

**私も全く同じでした。**

---

## 📖 Before：手動運用の限界

私は技術ブログを複数のプラットフォームで書いています。理由は単純で、**プラットフォームごとに読者層が違う**からです。

- **Note** → 有料記事で収益化。ストーリー重視の読者が多い
- **Zenn** → 技術的に深い内容が好まれる。エンジニアコミュニティ
- **Qiita** → エラー解決やベストプラクティスがバズりやすい
- **自作ブログ** → SEO・広告収益。自分のドメインに資産を蓄積

問題は、**1つの記事を書くたびに4回の投稿作業が発生する**ことでした。

| 作業 | 所要時間 |
|------|----------|
| 記事執筆 | 2-3時間 |
| Note用にフォーマット調整 | 15分 |
| Zenn用にfrontmatter修正 | 10分 |
| Qiita用にタグ形式変換 | 10分 |
| ブログ用にAstro形式に変換 | 15分 |
| 各プラットフォームにアクセスして投稿 | 20分 |
| X、Bluesky等でSNS告知 | 10分 |
| **合計（記事執筆除く）** | **約80分** |

記事を書くこと自体は楽しいのに、**投稿作業に毎回1時間以上かかる**。これでは記事を書くモチベーションが下がる一方です。

---

## 💡 転機：「1コマンドで全部やればいいのでは？」

ある日、ふと思いました。

> 「Markdownの中にプラットフォーム設定を書いて、CLIで一発投稿できたら最高では？」

やることは明確でした。

1. **統一フォーマット**を定義する（1つのMarkdownファイルに全設定を集約）
2. **プラットフォーム別に自動変換**する（frontmatter、タグ形式、本文の差異を吸収）
3. **各プラットフォームのAPIまたは自動化で投稿**する
4. **SNS告知も自動化**する

これを実現するため、**article-publisher** というPython CLIツールを開発しました。

---

## 🚀 After：1コマンドで全プラットフォームに配信

完成後のワークフローはこうなりました。

```bash
# 記事を書く
python -m src.cli init --title "記事タイトル" --slug "article-slug"
# → articles/drafts/article-slug.md が生成される

# 記事を書き終えたら検証
python -m src.cli validate articles/drafts/article-slug.md

# 全プラットフォームに一括投稿
python -m src.cli publish articles/drafts/article-slug.md
```

**これだけです。**

| 項目 | Before | After |
|------|--------|-------|
| 投稿作業時間 | 80分 | **5分以下** |
| 手動コピペ | 4回 | **0回** |
| フォーマット修正 | 毎回手動 | **自動変換** |
| SNS告知 | 手動 | **自動** |

投稿作業は **90%以上削減** され、記事を書くことだけに集中できるようになりました。

---

## 🔧 具体的な実装方法

### 全体アーキテクチャ

```
Markdown（統一フォーマット）
  ↓ [Parser]
Article オブジェクト
  ↓ [Converter]
プラットフォーム別フォーマット
  ↓ [Publisher]
  ├── Blog:  Astro + Cloudflare Pages
  ├── Note:  Playwright → 自動投稿
  ├── Zenn:  Git push → GitHub連携で自動デプロイ
  └── Qiita: REST API v2
  ↓ [Announcer]
  └── X(Twitter): tweepy → ツイート投稿
```

### Step 1: 統一Markdownフォーマットの設計

全ての設定を1つのfrontmatterに集約しました。

```yaml
---
title: "記事タイトル"
slug: "article-slug"
description: "記事の説明"
tags: [Python, 自動化]

platforms:
  note:
    enabled: true
    price: 0          # 0=無料, 500-1000=有料記事
  zenn:
    enabled: true
    emoji: "🚀"
    topics: [python, automation]
  qiita:
    enabled: true
  blog:
    enabled: true

announcement:
  enabled: true
  platforms: [twitter, bluesky]
---

# ここから記事本文
```

**設計のポイント：**

- `platforms` で投稿先を個別にON/OFF
- Zenn固有の `emoji` や `topics`、Note固有の `price` など、プラットフォーム特有の設定を吸収
- `announcement` でSNS告知もfrontmatterで制御

これを **dataclass** でバリデーションすることで、設定ミスを早期検出しています。

```python
@dataclass
class ZennPlatformConfig:
    enabled: bool = True
    emoji: str = "📝"
    topics: list[str] = field(default_factory=list)
    article_type: str = "tech"  # "tech" or "idea"
```

### Step 2: プラットフォーム別コンバーター

同じ記事でも、プラットフォームごとにフォーマットが異なります。

| 項目 | Zenn | Note | Qiita | Blog |
|------|------|------|-------|------|
| frontmatter | 独自形式 | なし | なし | Astro形式 |
| タグ | `topics` (配列) | UI操作 | `tags` (オブジェクト配列) | `tags` (配列) |
| 画像 | 相対パス | アップロード | アップロード | public/ |
| 有料部分 | - | `:::message-only` | - | - |

これらの差分を **Converter** クラスで吸収しています。コンバーターはプラットフォーム固有のブロック（`"
        def replacer(match):
            if match.group(1) == keep:
                return match.group(2)
            return ""
        return re.sub(pattern, replacer, content, flags=re.DOTALL)


class NoteConverter(PlatformConverter):
    def convert(self, article: Article) -> str:
        content = self._strip_platform_blocks(article.content, "note")
        return re.sub(
            r"```mermaid\n(.*?)\n```",
            "[図: 画像に変換が必要です]",
            content, flags=re.DOTALL,
        )


class ZennConverter(PlatformConverter):
    def convert(self, article: Article) -> str:
        content = self._strip_platform_blocks(article.content, "zenn")
        topics = ", ".join(
            f'"{t}"' for t in article.platforms.zenn.topics[:5]
        )
        fm = f'''---
title: "{article.title}"
emoji: "{article.platforms.zenn.emoji}"
type: "{article.platforms.zenn.article_type}"
topics: [{topics}]
published: true
---'''
        return f"{fm}\n\n{content}"


class BlogConverter(PlatformConverter):
    def convert(self, article: Article) -> str:
        content = self._strip_platform_blocks(article.content, "blog")
        tags = ", ".join(f'"{t}"' for t in article.tags)
        fm = f'''---
title: "{article.title}"
description: "{article.description}"
pubDate: "{article.created_at.strftime('%Y-%m-%d')}"
tags: [{tags}]
author: "{article.author}"
---'''
        return f"{fm}\n\n{content}"
```

---

### 3. Zenn Publisher（zenn.py）— Git操作

```python
import os
import subprocess
from pathlib import Path
from .base import Publisher, PublishResult
from ..transformer.article import Article


class ZennPublisher(Publisher):
    platform_name = "zenn"

    def __init__(self, zenn_content_path=None):
        self.content_path = Path(
            zenn_content_path
            or os.getenv("ZENN_CONTENT_PATH", "./zenn-content")
        )
        self.articles_path = self.content_path / "articles"

    async def publish(self, article, content):
        self.articles_path.mkdir(parents=True, exist_ok=True)
        file = self.articles_path / f"{article.slug}.md"
        file.write_text(content, encoding="utf-8")

        if await self._git_push(article.slug, f"Add: {article.title}"):
            user = os.getenv("ZENN_USERNAME", "tinou")
            url = f"https://zenn.dev/{user}/articles/{article.slug}"
            return PublishResult.success_result("zenn", url)
        return PublishResult.failure_result("zenn", "Git push failed")

    async def _git_push(self, slug, msg):
        try:
            cwd = str(self.content_path)
            for cmd in [
                ["git", "add", f"articles/{slug}.md"],
                ["git", "commit", "-m", msg],
                ["git", "push", "origin", "main"],
            ]:
                subprocess.run(cmd, cwd=cwd, check=True, capture_output=True)
            return True
        except subprocess.CalledProcessError:
            return False
```

---

### 4. Note Publisher（note.py）— Playwright自動化

```python
import asyncio
import json
import os
from pathlib import Path
from playwright.async_api import async_playwright
from .base import Publisher, PublishResult


class NotePublisher(Publisher):
    platform_name = "note"

    def __init__(self, email=None, password=None, headless=True):
        self.email = email or os.getenv("NOTE_EMAIL")
        self.password = password or os.getenv("NOTE_PASSWORD")
        self.cookies_path = Path("./.note_cookies.json")
        self.headless = headless

    async def publish(self, article, content):
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=self.headless)
            ctx = await browser.new_context()

            if self.cookies_path.exists():
                cookies = json.loads(self.cookies_path.read_text())
                await ctx.add_cookies(cookies)

            page = await ctx.new_page()

            if not await self._is_logged_in(page):
                await self._login(page)
                cookies = await ctx.cookies()
                self.cookies_path.write_text(json.dumps(cookies))

            url = await self._create_article(page, article, content)
            await browser.close()

            if url and "/notes/new" not in url:
                return PublishResult.success_result("note", url)
            return PublishResult.failure_result("note", "URL取得失敗")

    async def _is_logged_in(self, page):
        await page.goto("https://note.com/")
        await page.wait_for_load_state("networkidle")
        await asyncio.sleep(3)
        for sel in ['a[href*="/notes/new"]', '[class*="avatar"]']:
            if await page.query_selector(sel):
                return True
        return not await page.query_selector('a[href="/login"]')

    async def _login(self, page):
        await page.goto("https://note.com/login")
        await page.wait_for_load_state("networkidle")
        await asyncio.sleep(2)
        inputs = await page.query_selector_all("input")
        if len(inputs) >= 2:
            await inputs[0].fill(self.email)
            await inputs[1].fill(self.password)
        for btn in await page.query_selector_all("button"):
            text = await btn.text_content()
            if text and "ログイン" in text:
                await btn.click()
                break
        await asyncio.sleep(5)

    async def _create_article(self, page, article, content, publish=False):
        await page.goto("https://note.com/notes/new")
        await page.wait_for_load_state("networkidle")
        await asyncio.sleep(3)

        textarea = await page.query_selector("textarea")
        if textarea:
            await textarea.fill(article.title)

        editor = await page.query_selector('[contenteditable="true"]')
        if editor:
            await editor.click()
            for line in content.split("\n"):
                if line.strip():
                    await page.keyboard.type(line)
                await page.keyboard.press("Enter")
                await asyncio.sleep(0.05)

        await asyncio.sleep(2)
        for sel in ['button:has-text("下書き保存")', 'button:has-text("下書き")']:
            btn = await page.query_selector(sel)
            if btn:
                await btn.click()
                break
        await asyncio.sleep(5)
        return page.url
```

---

### 5. Qiita Publisher（qiita.py）— REST API

```python
import os
import httpx
from .base import Publisher, PublishResult


class QiitaPublisher(Publisher):
    platform_name = "qiita"
    BASE_URL = "https://qiita.com/api/v2"

    def __init__(self, access_token=None):
        self.token = access_token or os.getenv("QIITA_ACCESS_TOKEN")
        self.headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json",
        }

    async def publish(self, article, content):
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.BASE_URL}/items",
                headers=self.headers,
                json={
                    "title": article.title,
                    "body": content,
                    "tags": [{"name": t} for t in article.tags[:5]],
                    "private": False,
                },
                timeout=30.0,
            )
            if resp.status_code == 201:
                return PublishResult.success_result(
                    "qiita", resp.json()["url"]
                )
            return PublishResult.failure_result(
                "qiita", f"HTTP {resp.status_code}"
            )
```

---

### 6. SNS告知（service.py）

```python
import os
import tweepy


class TwitterAnnouncer:
    def __init__(self):
        self._client = tweepy.Client(
            consumer_key=os.getenv("X_API_KEY"),
            consumer_secret=os.getenv("X_API_SECRET"),
            access_token=os.getenv("X_ACCESS_TOKEN"),
            access_token_secret=os.getenv("X_ACCESS_TOKEN_SECRET"),
        )

    async def post(self, message):
        resp = self._client.create_tweet(text=message)
        tweet_id = resp.data["id"]
        return f"https://twitter.com/i/web/status/{tweet_id}"
```

---

以上が **article-publisher** の完全なソースコードです。

このコードをそのまま使えば、あなたも1コマンドで複数プラットフォームに記事を投稿できるようになります。自分のプロジェクトに合わせてカスタマイズしてみてください。

:::
<!-- endplatform -->



---

## 🔗 他のプラットフォームでも公開中

この記事は複数のプラットフォームで公開しています。

| プラットフォーム | 内容 |
|---|---|
| **Blog** | 全文 + SEO最適化版 → [blog.secure-auto-lab.com](https://blog.secure-auto-lab.com/articles/multi-platform-article-publisher) |
| **Note** | 全文 + 完全なソースコード（有料） → [note.com/secure_auto_lab](https://note.com/secure_auto_lab/) |
| **X** | 最新記事の告知 → [@secure_auto_lab](https://x.com/secure_auto_lab) |

フォロー・いいね・バッジで応援いただけると励みになります！

## 📝 まとめ：今日からできるアクションプラン

この記事で解説した内容をまとめます：

1. **統一フォーマット** — 1つのMarkdownのfrontmatterに全プラットフォームの設定を集約
2. **自動変換** — Converter でプラットフォーム別の差分を吸収
3. **自動投稿** — Publisher で各プラットフォームに配信（API、Git、Playwright）
4. **SNS告知** — Announcer で投稿後のSNS告知も自動化

**今日からできる具体的なアクション：**

> 📌 まずは、あなたが最も使うプラットフォーム2つだけの自動投稿から始めてみてください。
> Zenn（Git push）とQiita（REST API）の組み合わせなら、1日で実装できます。

---

## 🙏 おわりに

最後まで読んでいただき、ありがとうございました。

この記事を書いた理由は、「**書くことに集中したい**」という想いからです。

エンジニアは技術記事を書くことでアウトプット力が鍛えられ、キャリアにもプラスになります。しかし、投稿作業が面倒で書くこと自体を諦めてしまう人も多いのではないでしょうか。

投稿の面倒さをゼロにすれば、もっと多くのエンジニアが気軽にアウトプットできるようになる。そう信じて、このツールを作りました。

**あなたの技術記事が、誰かの助けになることを願っています。**

質問や感想があれば、ぜひコメントやXでお知らせください。

---

## 📚 参考リンク

- [Zenn CLIで記事を管理する](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Qiita API v2 ドキュメント](https://qiita.com/api/v2/docs)
- [Playwright Python公式ガイド](https://playwright.dev/python/)
- [Astro公式ドキュメント](https://docs.astro.build/)

---

**この記事が役に立ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じ悩みを持つ誰かの助けになります。