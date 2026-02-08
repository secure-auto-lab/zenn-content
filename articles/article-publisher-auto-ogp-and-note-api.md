---
title: "記事投稿ツールを大幅アップデート｜OGP自動生成・Note API化・Qiita対応を一気に実装した話"
emoji: "🔧"
type: "tech"
topics: ["python", "automation", "cli", "techblog", "ogp"]
published: false
---

![OGP](/images/article-publisher-auto-ogp-and-note-api-ogp.png)


# 記事投稿ツールを大幅アップデート｜OGP自動生成・Note API化・Qiita対応を一気に実装した話

---

## この記事で得られること

> **この記事を読むと、以下のことがわかります。**
>
> - OGP画像をHTML+Playwrightで自動生成し、複数プラットフォームに配信する方法
> - NoteのPlaywright操作が壊れた時に、内部APIで完全代替する手法
> - 収益化できないプラットフォーム（Qiita）をブログ誘導に活用する戦略

---

## あなたもこんな悩みを抱えていませんか？

- 「記事を書いたけど、各プラットフォーム用にOGP画像を手動で設定するのが面倒…」
- 「Noteのエディタが変わって、Playwright自動投稿が動かなくなった…」
- 「Qiitaにも投稿したいけど、収益化できないから全文載せたくない…」

**私も以前は、まったく同じ状況でした。**

特にNoteのエディタ移行は深刻でした。ある日突然、自動投稿スクリプトが全く動かなくなったのです。

---

## Before：壊れた自動投稿と手作業の苦痛

### Noteの自動投稿が完全に壊れた

以前のarticle-publisherは、Noteへの投稿にPlaywrightを使ったブラウザ自動化を採用していました。しかし、Noteがエディタを `editor.note.com` （Next.js SPA）に移行した結果、以下の問題が発生しました。

- `headless=True` でエディタが読み込まない（白い画面のまま）
- ProseMirrorエディタへの直接入力が不安定
- ページ遷移のタイミングが予測不能

**数日かけて調整しても、安定動作には至りませんでした。**

### OGP画像は毎回手作業

記事を投稿するたびに、Canvaを開いてOGP画像を作り、各プラットフォームに手動でアップロード。1記事あたり15-20分のロスが発生していました。

### Qiitaは放置状態

アカウントは作ったものの、収益化できないプラットフォームに全文を載せるメリットを感じず、完全に放置していました。

---

## 転機：NoteのAPIを直接叩けることに気づいた

Noteのエディタが壊れた時、「もうNoteは諦めよう」と思いかけました。

しかし、ブラウザの開発者ツールでネットワーク通信を観察していると、**エディタが内部APIを叩いていること**に気づいたのです。

```
POST /api/v1/text_notes → 201 Created（ドラフト作成）
POST /api/v1/text_notes/draft_save?id={id} → 201（コンテンツ保存）
```

**Playwrightでブラウザを操作する必要はない。APIを直接叩けばいい。**

この発見が、すべてを変えました。

---

## After：1コマンドで6プラットフォームに自動配信

アップデート後の投稿フローはこうなりました。

```
python -m src.cli publish articles/drafts/my-article.md
```

**たった1コマンドで、以下が自動実行されます。**

| ステップ | 処理内容 |
|---------|---------|
| OGP生成 | 1200x630のタイトル画像を自動生成 |
| Blog | 記事+OGP画像をAstroプロジェクトに配置 |
| Note | 内部APIでドラフト作成+OGP画像埋め込み |
| Zenn | 記事+OGP画像をGitリポジトリにpush |
| Qiita | 要約+ブログ誘導リンクをAPI投稿 |
| SNS告知 | X・Bluesky・Misskeyに自動投稿 |

**以前は30分以上かかっていた作業が、1分で完了するようになりました。**

---

## 具体的な実装方法

### 1. OGP画像の自動生成

HTML+CSSでテンプレートを作り、Playwrightでスクリーンショットを撮る方式を採用しました。

```python
class OgpGenerator:
    async def generate(self, title, tags, output, author, theme="default"):
        html = self._render_template(title, tags, author, theme)
        async with async_playwright() as p:
            browser = await p.chromium.launch()
            page = await browser.new_page(viewport={"width": 1200, "height": 630})
            await page.set_content(html)
            await page.screenshot(path=output)
```

4つのカラーテーマ（ブルー・パープル・グリーン・オレンジ）を用意し、Google Fonts（Noto Sans JP + Inter）を使用。幾何学模様の背景パターンとアクセントラインで、プロフェッショナルな見た目を実現しています。

**ポイント：** タイトルの文字数に応じてフォントサイズを自動調整しています。20文字以下なら52px、60文字以上なら30pxと、常に最適なバランスで表示されます。

### 2. Note内部APIへの移行

PlaywrightによるGUI操作を完全に廃止し、httpxによるAPI直接呼び出しに切り替えました。

```python
class NotePublisher:
    async def publish(self, article, content, ogp_path=None):
        async with httpx.AsyncClient() as client:
            # Step 1: ドラフト作成
            resp = await client.post(
                f"{self.BASE_URL}/api/v1/text_notes",
                headers=headers,
                json={"template_key": None},
            )
            note_id = resp.json()["data"]["id"]

            # Step 2: Markdown → Note HTML変換
            body_html, body_length = self.html_converter.convert(content)

            # Step 3: OGP画像をブログURLから参照
            if ogp_path:
                img_url = f"{BLOG_URL}/images/{article.slug}-ogp.png"
                body_html = f'<figure><img src="{img_url}"></figure>' + body_html

            # Step 4: コンテンツ保存
            await client.post(
                f"{self.BASE_URL}/api/v1/text_notes/draft_save?id={note_id}",
                json={"body": body_html, "name": article.title, ...},
            )
```

**発見した重要なポイント：**

- Note APIには `X-Requested-With: XMLHttpRequest` ヘッダーが必須
- HTML形式は `<p name="uuid" id="uuid">` のように各要素にUUIDが必要（ProseMirrorエディタの仕様）
- 画像アップロードAPIは非公開のため、ブログの公開URLから `<img>` 参照で対応

### 3. OGP画像の自動配信

OGP画像は生成後、各プラットフォームに自動配信されます。

```
articles/images/{slug}-ogp.png（生成元）
  ├── blog/public/images/  → ブログサーバーで公開
  ├── zenn-content/images/  → 記事先頭に ![OGP] 埋め込み
  └── Note: ブログURLから <img> 参照
```

**Blog**にOGP画像をコピーすることで公開URLが生まれ、その URLを**Note**が参照する。**Zenn**はGitリポジトリ内の画像を直接参照。プラットフォームの特性に合わせた配信戦略です。

### 4. Qiitaブログ誘導戦略

Qiitaは収益化できないため、全文ではなく**要約+ブログ誘導リンク**を投稿する方式にしました。

```python
class QiitaConverter:
    BLOG_BASE_URL = "https://blog.secure-auto-lab.com/articles"

    def convert(self, article):
        blog_url = f"{self.BLOG_BASE_URL}/{article.slug}"
        return f"""# {article.title}

{article.description}

## この記事について

本記事の全文は以下のブログで公開しています。

**[>> 全文を読む: {article.title}]({blog_url})**
"""
```

Qiitaの検索流入をブログに誘導することで、AdSenseやAmazon Associateの収益に繋げる戦略です。

---

## 技術的なハマりポイントと解決策

### Windows cp932エンコーディング問題

Richライブラリのスピナーが Braille文字（`⠋⠙⠹`）を使用しており、Windowsコンソール（cp932）でUnicodeEncodeErrorが発生しました。

```python
# NG: デフォルトスピナーはBraille文字を使う
console.status(status="Processing...")

# OK: ASCIIのみのlineスピナーを指定
console.status(spinner="line", status="Processing...")
```

`spinner="line"` を指定することで、ASCII文字のみのアニメーションに切り替わり、問題を解決しました。

### Note headless問題

Noteのエディタは `headless=True` では正常に読み込みませんでした。SPAのレンダリングにGPUアクセラレーションが必要だったと推測されます。

**解決策：** API方式に切り替えたことで、Playwright自体がログイン時のみの使用となり、headless問題は完全に消滅しました。

---

## まとめ：今日からできるアクションプラン

この記事で解説した実装を整理すると：

1. **OGP自動生成**: HTML+Playwright screenshotで高品質な画像を自動生成
2. **Note API化**: 内部APIの直接呼び出しでPlaywright依存を最小化
3. **Qiita誘導**: 要約+リンクで検索流入をブログに集約
4. **マルチ配信**: 1コマンドで6プラットフォームに同時配信

**今日からできる具体的なアクション：**

> まずは自分の投稿フローを見直してみてください。手動で繰り返している作業があれば、それは自動化のチャンスです。
> 特にOGP画像の生成は、HTMLテンプレート+スクリーンショットの方式なら30分で実装できます。

---

## おわりに

最後まで読んでいただき、ありがとうございました。

「Noteの自動投稿が壊れた」という問題を、APIの直接呼び出しで解決できた時は本当に嬉しかったです。プラットフォームのUI変更に振り回されず、安定した自動化基盤を作ることの大切さを実感しました。

技術記事の投稿は、書くこと自体が大変なのに、各プラットフォームへの配信作業で消耗するのはもったいない。この記事が、あなたの執筆活動の効率化に少しでも役立てば幸いです。

質問や感想があれば、ぜひコメントやSNSでお知らせください。

---

## 参考リンク

- [article-publisher リポジトリ](https://github.com/secure-auto-lab/article-publisher)
- [Playwright 公式ドキュメント](https://playwright.dev/python/)
- [Zenn CLIで記事を管理する](https://zenn.dev/zenn/articles/zenn-cli-guide)

---

**この記事が役に立ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じ悩みを持つ誰かの助けになります。