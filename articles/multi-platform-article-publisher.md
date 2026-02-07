---
title: "技術記事を4つのプラットフォームに同時投稿するCLIを作った話"
emoji: "🚀"
type: "tech"
topics: ["python", "automation", "techblog", "playwright", "cli"]
published: false
---

# 技術記事を4つのプラットフォームに同時投稿するCLIを作った話

**「Note、Zenn、Qiita、ブログ……毎回コピペするの、もう限界だ。」**

1つのMarkdownファイルから4プラットフォームに同時投稿できるCLIツールを作り、**記事公開にかかる作業時間を90%以上削減**しました。この記事では、その設計と実装を全て公開します。

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

これを **Pydantic** でバリデーションすることで、設定ミスを早期検出しています。

```python
class ZennPlatformConfig(BaseModel):
    enabled: bool = True
    emoji: str = "📝"
    topics: list[str] = []
    article_type: str = "tech"  # "tech" or "idea"
```

### Step 2: プラットフォーム別コンバーター

同じ記事でも、プラットフォームごとにフォーマットが異なります。

| 項目 | Zenn | Note | Qiita | Blog |
|------|------|------|-------|------|
| frontmatter | 独自形式 | なし | なし | Astro形式 |
| タグ | `topics` (配列) | UI操作 | `tags` (オブジェクト配列) | `tags` (配列) |
| 画像 | 相対パス | アップロード | アップロード | public/ |
| 有料部分 | × | `:::message-only` | × | × |

これらの差分を **Converter** クラスで吸収しています。

```python
class ZennConverter:
    def convert(self, article: Article) -> str:
        """Article → Zenn形式のMarkdownに変換"""
        frontmatter = {
            "title": article.title,
            "emoji": article.platforms.zenn.emoji,
            "type": article.platforms.zenn.article_type,
            "topics": article.platforms.zenn.topics,
            "published": True,
        }
        return f"---\n{yaml.dump(frontmatter)}---\n\n{article.content}"
```

### Step 3: プラットフォーム別Publisher

各プラットフォームの投稿方法は大きく異なります。

#### Zenn — Git pushで自動デプロイ

Zennは **GitHubリポジトリ連携** が最もシンプルです。

```python
class ZennPublisher(Publisher):
    async def publish(self, article, content):
        # 1. zenn-content/articles/ にファイルを書き出し
        article_file = self.articles_path / f"{article.slug}.md"
        article_file.write_text(content, encoding="utf-8")

        # 2. git add → commit → push
        await self._git_push(article.slug, f"Add article: {article.title}")

        # 3. Zennが自動デプロイ（1-2分）
        return f"https://zenn.dev/{username}/articles/{article.slug}"
```

GitHubにpushするだけでZennが自動的にデプロイしてくれるため、API認証は不要です。

#### Note — Playwrightでブラウザ自動操作

NoteにはAPIがないため、**Playwright**（ブラウザ自動化）で投稿します。

```python
class NotePublisher(Publisher):
    async def publish(self, article, content):
        async with async_playwright() as p:
            browser = await p.chromium.launch()
            page = await browser.new_page()

            # 1. ログイン
            await self._login(page)

            # 2. 新規記事作成ページへ
            await page.goto("https://note.com/new")

            # 3. タイトル・本文を入力
            await page.fill('[placeholder*="タイトル"]', article.title)
            # ... 本文入力、画像アップロード

            # 4. 有料設定（price > 0の場合）
            if article.platforms.note.price > 0:
                await self._set_price(page, article.platforms.note.price)

            # 5. 公開
            await page.click('button:text("公開")')
```

**💡 ハマりポイント：**
NoteのUIは頻繁に変わるため、セレクタが壊れやすいです。テスト用に `note-login` コマンドを用意して、定期的にログイン確認できるようにしました。

```bash
python -m src.cli note-login  # ログインテスト
```

#### Qiita — REST API v2

Qiitaは公式REST APIがあり、最もシンプルに実装できます。

```python
class QiitaPublisher(Publisher):
    async def publish(self, article, content):
        response = await self.client.post(
            "https://qiita.com/api/v2/items",
            headers={"Authorization": f"Bearer {self.token}"},
            json={
                "title": article.title,
                "body": content,
                "tags": [{"name": t} for t in article.tags],
                "private": False,
            }
        )
        return response.json()["url"]
```

### Step 4: SNS告知の自動化

記事を全プラットフォームに投稿した後、自動的にSNS告知を行います。

```python
class AnnouncementService:
    async def announce(self, article, urls):
        message = self._format_message(article, urls)

        for platform in article.announcement.platforms:
            if platform == "twitter":
                await self.twitter.post(message)
            elif platform == "bluesky":
                await self.bluesky.post(message)
```

X（Twitter）への投稿は **tweepy**（OAuth 1.0a認証）を使用しています。Free Tierでは月間の投稿数に制限がありますが、記事告知には十分です。

---

## 📊 技術スタック

| 要素 | 技術 |
|------|------|
| 言語 | Python 3.11+ |
| CLI | Typer |
| データモデル | Pydantic |
| ブラウザ自動化 | Playwright |
| HTTP通信 | httpx |
| ブログ | Astro + Tailwind CSS |
| ホスティング | Cloudflare Pages |
| X投稿 | tweepy (OAuth 1.0a) |

---

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