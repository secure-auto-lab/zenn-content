---
title: "1つのMarkdownから4つの顔を持つ記事を自動生成する仕組みを作った話"
emoji: "🔄"
type: "tech"
topics: ["python", "automation", "markdown", "zenn", "note"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/platform-content-optimization"
---

![OGP](/images/platform-content-optimization-ogp.png)


# 1つのMarkdownから4つの顔を持つ記事を自動生成する仕組みを作った話

---

### 決め手になった3つのポイント

1. **見出しキーワード判定の汎用性：** 記事ごとにルールを設定する必要がなく、テンプレートに沿って書くだけで自動変換される
2. **双方向の設計：** NoteConverter（技術を除去）とZennConverter（ストーリーを除去）は同じアルゴリズムの裏表。一度実装すれば両方動く
3. **段階的拡張：** キーワードリストに追加するだけで、除去対象を柔軟に調整できる

---

## 🔧 具体的な実装方法

### 全体アーキテクチャ

```
┌─────────────────┐
│  Master Markdown │  ← 1つの記事ファイル
└────────┬────────┘
         │
    ┌────┴────┐
    │ Parser  │  Frontmatter解析 + コンテンツ分離
    └────┬────┘
         │
   ┌─────┼─────────┬──────────┐
   ▼     ▼         ▼          ▼
┌──────┐┌──────┐┌──────┐┌──────────┐
│ Note ││ Zenn ││Qiita ││   Blog   │
│Convtr││Convtr││Convtr││ Converter│
└──┬───┘└──┬───┘└──┬───┘└────┬─────┘
   │       │       │         │
   ▼       ▼       ▼         ▼
 ストーリー  技術    要約      完全版
 重視      重視   +誘導
```

### Step 1: 見出しキーワード判定アルゴリズム

NoteConverterとZennConverterの核となるアルゴリズムは同一です。違いは「何を除去するか」のキーワードリストだけ。

```python
class NoteConverter(PlatformConverter):
    """Note向け：技術セクションを除去してストーリー重視に変換"""

    # 技術セクションと判定するキーワード
    _TECHNICAL_KEYWORDS = [
        "具体的な実装", "実装方法", "アーキテクチャ",
        "環境構築", "セットアップ", "FAQ", "よくある質問",
        "参考リンク", "ハマりポイント", "API仕様",
    ]

    def _remove_technical_sections(self, content: str) -> str:
        lines = content.split("\n")
        result = []
        skip = False
        skip_level = 0

        for line in lines:
            heading_match = re.match(r"^(#{2,6})\s+(.+)$", line)
            if heading_match:
                level = len(heading_match.group(1))
                text = heading_match.group(2)

                if self._is_technical_heading(text):
                    skip = True        # スキップ開始
                    skip_level = level  # この見出しレベルを記録
                    continue
                elif skip and level <= skip_level:
                    skip = False       # 同レベル以上の別見出しでスキップ解除

            if not skip:
                result.append(line)

        return "\n".join(result)
```

**ポイント：** `skip_level` で見出しの階層を追跡することで、`## 実装方法` の下の `### Step 1`、`### Step 2` もまとめてスキップできます。次の `## まとめ` のような同レベル以上の見出しが来た時点でスキップが解除されます。

### Step 2: ZennConverterの逆転キーワード

ZennConverterは同じアルゴリズムを使い、キーワードリストだけが異なります。

```python
class ZennConverter(PlatformConverter):
    """Zenn向け：ストーリーセクションを除去して技術重視に変換"""

    _STORY_KEYWORDS = [
        "悩み", "抱えていませんか",
        "ストーリー", "道のり",
        "なぜこのアプローチ", "壁にぶつかった",
        "教訓", "学んだ", "おわりに",
        "Before", "After", "転機",
        "どん底", "絶望", "突破口",
    ]
```

NoteConverterとZennConverterは**鏡像関係**です。一方が除去するセクションを、もう一方は保持する。

### Step 3: NoteConverter固有の処理

Note向けには、セクション除去に加えて以下の処理も行います。

```python
def convert(self, article: Article) -> str:
    content = self._strip_platform_blocks(article.content, "note")
    content = self._remove_technical_sections(content)  # 技術セクション除去
    content = self._remove_code_blocks(content)          # コードブロック除去
    content = self._remove_ascii_diagrams(content)       # ASCII図除去
    content = self._remove_inline_code(content)          # `バッククォート` → テキスト化
    content = self._convert_callouts(content)            # Callout → 太字テキスト
    content = self._clean_empty_lines(content)           # 連続空行を整理
    content = self._add_blog_link(content, article.slug) # ブログ誘導リンク追加
    return content
```

**インラインコードの処理が地味に重要です。** Noteの読者は `npm install` という表記に馴染みがない人も多い。バッククォートを除去してプレーンテキストにすることで、より自然に読めるようになります。

### Step 4: canonical URLによるSEO対策

ZennとQiitaにcanonical URLを設定し、Googleにブログが正規URLであることを伝えます。

```python
# Zenn: frontmatterに canonical フィールドを追加
def _generate_frontmatter(self, article: Article) -> str:
    canonical_url = f"{self.BLOG_BASE_URL}/{article.slug}"
    return f"""---
title: "{article.title}"
emoji: "{article.platforms.zenn.emoji}"
topics: [{topics_str}]
published: {published}
canonical: "{canonical_url}"
---"""

# Qiita: APIペイロードに canonical_url を追加
def _build_payload(self, article, content):
    canonical_url = f"{self.BLOG_BASE_URL}/{article.slug}"
    return {
        "title": article.title,
        "body": content,
        "canonical_url": canonical_url,  # ブログを正規URLに指定
    }
```

これにより、同じ内容が複数サイトに存在しても、SEO的にはブログに評価が集約されます。

### Step 5: 記事テンプレートの再設計

自動変換が正しく機能するには、記事の構造を設計段階で考慮する必要があります。

テンプレートを以下の11セクション構成に再設計しました。

```
1. フック             → 全プラットフォームに残る
2. 導入               → 全プラットフォームに残る
3. 共感パート（Before）→ Note向け（Zennでは除去）
4. ストーリー（転機）  → Note向け（Zennでは除去）
5. 成果（After）       → Note向け（Zennでは除去）
6. なぜこのアプローチか → Note向け（Zennでは除去）
7. 具体的な実装方法    → Zenn向け（Noteでは除去）
8. 壁と乗り越え方     → Note向け（Zennでは除去）
9. 教訓              → Note向け（Zennでは除去）
10. まとめ            → 全プラットフォームに残る
11. おわりに          → Note向け（Zennでは除去）
```

**ブログには全セクションが残ります。** Note向けには技術セクション（7番）が除去され、Zenn向けにはストーリーセクション（3-6, 8-9, 11番）が除去されます。

---

## 🚀 応用編：OGP画像の視認性向上

今回のアップデートでは、OGP画像（記事のサムネイル）も改善しました。

### タイトルをカード中央に配置

以前のOGP画像はタイトルが小さく、SNSのタイムラインで目立ちませんでした。

```python
# タイトル文字数に応じてフォントサイズを自動調整
if len(title) <= 15:
    font_size = "72px"    # 短いタイトルは大きく
elif len(title) <= 25:
    font_size = "62px"
elif len(title) <= 40:
    font_size = "52px"
elif len(title) <= 55:
    font_size = "44px"
else:
    font_size = "38px"    # 長いタイトルでも読める最小サイズ
```

レイアウトも変更し、タイトルをカードの中央に配置。タグと著者名は下部に小さく表示することで、**タイトルの存在感を最大化**しました。

---

## ❓ よくある質問（FAQ）

### Q1: テンプレートに沿わない構成の記事でも変換できますか？

A: はい。見出しキーワードマッチで判定するため、テンプレート通りでなくても、見出しに「実装」や「ストーリー」が含まれていれば自動除去されます。ただし、テンプレートに沿うことで変換精度が最も高くなります。

### Q2: キーワードの誤判定はありませんか？

A: 絵文字を除去してからキーワードマッチを行うため、見出しに絵文字が含まれていても正しく判定されます。また、`## 実装に至った背景` のような見出しは「実装」を含みますが、`_TECHNICAL_KEYWORDS` に「実装に至った」は含まれていないため、より具体的なキーワード（「具体的な実装」「実装方法」等）でマッチさせることで誤判定を抑えています。

### Q3: canonical URLを設定しないとどうなりますか？

A: Googleが同一コンテンツを複数URLで検出した場合、どのURLを正規とするかを自動判定します。これによりブログではなくZennやQiitaの方が上位表示される可能性があり、ブログへの流入が減少します。canonical URLの設定で、SEO評価をブログに集約できます。

---

## 📝 まとめ：今日からできるアクションプラン

この記事で解説した内容を整理します：

1. **記事テンプレートを設計する：** ストーリーセクションと技術セクションを見出しレベルで明確に分離
2. **キーワードリストを定義する：** 除去対象の見出しキーワードを決める
3. **コンバーターを実装する：** 見出し走査→キーワード判定→セクション単位スキップ
4. **canonical URLを設定する：** Zenn（frontmatter）、Qiita（API）でブログを正規URLに指定

**今日からできる具体的なアクション：**

> まずは自分の記事を「ストーリーパート」と「技術パート」に分けてみてください。
> その境界が見出しレベルで明確になっていれば、自動変換の仕組みは驚くほど簡単に実装できます。

---

## 📚 参考リンク

- [Zenn Front Matter の書き方](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Qiita API v2 ドキュメント](https://qiita.com/api/v2/docs)
- [canonical URL と SEO の関係](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/platform-content-optimization)**
