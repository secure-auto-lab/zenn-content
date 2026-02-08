---
title: "NoteのProseMirror対応HTML変換を作った話｜アイキャッチ画像APIも発見"
emoji: "🔍"
type: "tech"
topics: ["python", "automation", "api", "prosemirror", "note"]
published: false
---

![OGP](/images/note-prosemirror-html-and-eyecatch-api-ogp.png)


# NoteのProseMirror対応HTML変換を作った話｜アイキャッチ画像APIも発見

---

## この記事で得られること

> **この記事を読むと、以下のことがわかります。**
>
> - NoteのProseMirrorエディタが受け付けるHTML形式の完全な対応表
> - Markdown→Note HTML変換器の設計と実装方法
> - JSバンドル解析による非公開APIエンドポイントの発見手法
> - アイキャッチ画像APIの仕様と実装

---

## あなたもこんな悩みを抱えていませんか？

- 「Noteに内部APIで投稿したけど、コードブロックが表示されない…」
- 「テーブルを書いたのに、ただのテキストになってしまう…」
- 「アイキャッチ画像を自動で設定したいけど、APIが見つからない…」

**私も先日、まったく同じ壁にぶつかりました。**

前回の記事で紹介したNote内部API投稿の仕組みは動いていましたが、実際に投稿された記事を見ると、コードブロックが崩壊し、テーブルは消滅し、改行も効いていない。そしてアイキャッチ画像は手動設定のまま。

これでは「自動投稿」とは呼べません。

---

## Before：崩壊した記事フォーマット

### コードブロックが表示されない

最初の実装では、Noteのエディタが `<code-block>` タグを使っていると推測し、こう変換していました。

```
<code-block data-lang="python">コード内容</code-block>
```

しかし実際に投稿してみると、**コードブロックが完全に消えていました**。エディタのDOM構造を見て推測した形式が、実際にはAPIからの保存時に無視されていたのです。

### テーブルとインライン書式も全滅

HTMLの `<table>` タグ、インラインの `<code>` タグ、`<em>` タグ。すべてNoteのProseMirrorエディタに無視されていました。

投稿された記事は、素のテキストが羅列されるだけの悲惨な状態でした。

### アイキャッチ画像は手動

OGP画像の自動生成は実装済みでしたが、Noteの下書きにアイキャッチとして設定する方法が見つからない。`draft_save` APIにどんなフィールド名を送っても、すべて無視されていました。

---

## 転機：テストドリブンで仕様を特定する

「推測ではなく、テストで確かめよう。」

そう決意し、NoteのProseMirrorエディタが実際に何を受け付けるのかを、体系的にテストすることにしました。

---

## After：完全対応のHTML変換器とアイキャッチ自動設定

4回のテストイテレーションと、JSバンドル解析という予想外のアプローチを経て、以下を達成しました。

| 項目 | Before | After |
|------|--------|-------|
| コードブロック | 表示されない | 完全動作 |
| テーブル | 消滅 | 太字ヘッダー+パイプ区切りで表示 |
| インライン書式 | 崩壊 | 対応/非対応を正しく処理 |
| 空行スペーシング | 無視される | 正常動作 |
| アイキャッチ画像 | 手動設定 | API自動アップロード |

---

## 具体的な実装方法

### 1. ProseMirror対応HTMLの特定（4回のテスト）

テスト用のNoteドラフトを作成し、様々なHTML形式を送信して実際の表示を確認するという、地道な作業を4回繰り返しました。

**テスト1: 基本要素**

8種類のHTML形式でドラフトを作成。結果、`<pre>` タグのみがコードブロックとして表示され、`<code-block>` は完全に無視されることが判明しました。

**テスト2: 修正版の検証**

`<pre><code>` 形式に切り替え、太字・引用・リスト・テーブルの代替形式もテスト。コードブロック、太字、引用、リストがOKであることを確認。

**テスト3: スペーシングとインライン**

空行の表現方法と、インライン書式の対応状況をテスト。`<p><br></p>` で空行が機能すること、そして**インライン書式はすべてNG**であることが確定しました。

**テスト4: LaTeXテーブル**

テーブルの代替としてLaTeXの `$\begin{array}...\end{array}$` 形式を8パターンテスト。結果は「全部だめ」でした。

### 最終的な対応表

この4回のテストで、NoteのProseMirror対応を完全に特定できました。

**対応するHTML形式:**

- `<pre><code>...</code></pre>` - コードブロック
- `<strong>` - 太字（唯一のインライン装飾）
- `<ul><li>`, `<ol><li>` - リスト
- `<blockquote>` - 引用
- `<p><br></p>` - 空行スペーシング
- `<a href>` - リンク
- `<h2>`, `<h3>` 等 - 見出し
- `<hr>` - 水平線
- `<figure><img>` - 画像

**非対応（無視される）:**

- `<code>` インライン - 除去される
- `<em>`, `<i>` - 除去される
- `<table>` - 除去される
- `<code-block>` - 除去される
- `<mark>` - 除去される
- LaTeX数式 - すべての形式が非対応

### 2. Markdown→Note HTML変換器

テスト結果を元に、変換器を実装しました。

```python
class NoteHtmlConverter:
    def convert(self, markdown: str) -> tuple[str, int]:
        # 各行を解析してNote互換HTMLに変換
        while i < len(lines):
            line = lines[i]

            # コードブロック → <pre><code>
            if line.startswith("```"):
                code_lines = []
                # ...collect code lines...
                html_parts.append(
                    f'<pre name="{p_id}" id="{p_id}">'
                    f'<code>{escaped_code}</code></pre>'
                )

            # テーブル → 太字ヘッダー+パイプ区切りテキスト
            if line.strip().startswith("|"):
                self._convert_table(table_lines, html_parts)

            # 空行 → スペーシング段落
            if not line.strip():
                html_parts.append(f'<p name="{p_id}" id="{p_id}"><br></p>')
```

**重要なポイント：** NoteのProseMirrorエディタは、すべての要素に `name` と `id` 属性としてUUIDを要求します。これがないと正しく保存されません。

### 3. インライン書式の処理戦略

Noteがインラインの `<code>` と `<em>` を除去する以上、Markdownの原文をできるだけ活かす方針にしました。

```python
def _inline(self, text: str) -> str:
    # 太字 → <strong>（唯一のインライン装飾）
    text = re.sub(r"\*\*(.+?)\*\*", r"<strong>\1</strong>", text)
    # 斜体 → プレーンテキスト化
    text = re.sub(r"\*(.+?)\*", r"\1", text)
    # インラインコード → バッククォート文字をそのまま保持
    # リンク → <a href>
    text = re.sub(r"\[([^\]]+)\]\(([^)]+)\)", r'<a href="\2">\1</a>', text)
    return text
```

インラインコードは `<code>` に変換すると消えてしまうため、バッククォート文字をそのまま残すことで、読者には「ここがコードである」ということが伝わります。

### 4. テーブルの代替表現

HTMLの `<table>` が使えないため、太字ヘッダー + パイプ区切りのテキスト段落で表現します。

```python
def _convert_table(self, table_lines, html_parts):
    # ヘッダー行: <strong>で太字
    header_text = " | ".join(
        f"<strong>{h}</strong>" for h in headers
    )
    html_parts.append(f'<p name="{p_id}" id="{p_id}">{header_text}</p>')

    # データ行: パイプ区切り
    for row in data_rows:
        row_text = " | ".join(row)
        html_parts.append(f'<p name="{p_id}" id="{p_id}">{row_text}</p>')
```

表形式にはなりませんが、ヘッダーが太字で区別できるため、十分に読みやすい形式です。

---

### 5. アイキャッチ画像APIの発見

ここからが今回最大の発見です。

`draft_save` APIにどんなフィールド名（`key_visual_image_src`, `eyecatch`, `thumbnail` 等）を送っても、すべて無視されました。`editor.note.com` の画像アップロードAPIを叩くと403。

しかし403のレスポンスを見ると、**CloudFrontのエラー**でした。

```
This distribution is not configured to allow the HTTP request method
that was used for this request.
```

つまり `editor.note.com` はCloudFrontで配信される静的SPAであり、POSTリクエスト自体がCDN層でブロックされていたのです。APIは `editor.note.com` ではなく `note.com/api` を経由している。

### JSバンドル解析でエンドポイントを発見

Playwrightでエディタのページをロードし、JavaScriptのチャンクファイルを取得。「eyecatch」「upload」「image」などのキーワードで検索しました。

```python
# Playwrightでeditor.note.comを開き、JSチャンクを収集
page.on("response", handle_response)
await page.goto("https://editor.note.com")

# JSバンドル内を検索
for pattern in ['eyecatch', 'image_upload']:
    for match in re.finditer(pattern, js_content):
        # 前後のコンテキストを取得
```

そして見つけたのが、この2行でした。

```javascript
let s = await ey.post("/v1/image_upload/note_eyecatch", a);
// ...
let i = await ey.post("/v1/image_upload/text_note_picture", o);
```

さらにJSのコンテキストから、FormDataの構造も特定できました。

```javascript
a.append("note_id", String(t));
a.append("file", r);
a.append("width", String(1920));
a.append("height", String(1005));
```

**ベースURL:** JSバンドル内で `e9="https://note.com"`, `e7="/api"` と定義されていたため、完全なエンドポイントは `https://note.com/api/v1/image_upload/note_eyecatch` でした。

### 6. アイキャッチ画像の自動アップロード

発見したAPIをPythonで実装しました。

```python
async def _upload_eyecatch(self, client, cookies, note_id, ogp_path):
    image_data = Path(ogp_path).read_bytes()

    resp = await client.post(
        f"{self.BASE_URL}/api/v1/image_upload/note_eyecatch",
        headers=headers,
        files={"file": ("eyecatch.png", image_data, "image/png")},
        data={
            "note_id": str(note_id),
            "width": "1920",
            "height": "1005",
        },
        timeout=30.0,
    )

    if resp.status_code in (200, 201):
        return resp.json()["data"]["url"]
    return None
```

レスポンスには、Noteのアセットサーバーにアップロードされた画像のURLが返されます。

```json
{
  "data": {
    "url": "https://assets.st-note.com/production/uploads/images/.../rectangle_large_type_2_xxxxx.png"
  }
}
```

これで投稿フローは完全に自動化されました。

```
python -m src.cli publish articles/drafts/my-article.md
  │
  ├── OGP画像を自動生成
  ├── Noteドラフト作成
  ├── アイキャッチ画像をAPIでアップロード  ← NEW
  ├── Markdown→Note HTML変換（修正済み）  ← FIXED
  └── ドラフト保存
```

---

## 技術的なハマりポイントと解決策

### CloudFrontの403とCSRF認証の罠

最初は `editor.note.com` の403をCSRF認証の問題だと思い、Playwrightでブラウザコンテキストから認証済みfetchを実行するなど、様々な回避策を試みました。

しかし実際は、CloudFrontがPOSTリクエスト自体をブロックしていただけでした。エディタのSPAは `note.com/api` にリクエストを送っており、`editor.note.com` には一切のAPIエンドポイントが存在しません。

**教訓：** 403エラーが返ってきたら、レスポンスボディを必ず確認する。認証問題なのか、インフラ層のブロックなのかで対策は全く異なります。

### draft_saveのフィールド名の罠

NoteのAPIは、`draft_save` に未知のフィールドを送っても**エラーを返さずに201を返します**。つまり `key_visual_image_src` や `eyecatch` を送っても成功レスポンスが返り、実際には何も保存されていない。

サイレントフェイルは最も厄介なバグの温床です。

**教訓：** APIが成功を返しても、実際にデータが保存されたか必ず確認する。

---

## まとめ：今日からできるアクションプラン

この記事で解説した内容を整理すると：

1. **ProseMirror対応テスト**: 推測ではなく実際のテストで対応HTML形式を特定
2. **HTML変換器**: テスト結果に基づいた堅実な変換ロジック
3. **JSバンドル解析**: 非公開APIエンドポイントの発見手法
4. **アイキャッチAPI**: `POST /api/v1/image_upload/note_eyecatch` の実装

**今日からできる具体的なアクション：**

> NoteのAPIを自前で叩いている方は、まずこの記事のHTML対応表を参考にしてください。
> また、プラットフォームの非公開APIを探す際は、ブラウザの開発者ツールだけでなく、JSバンドルの静的解析も強力なツールになります。

---

## おわりに

最後まで読んでいただき、ありがとうございました。

「テストで仕様を特定する」「JSバンドルを読んでAPIを見つける」。どちらも泥臭い作業ですが、結果的にはこの地道なアプローチが最も確実でした。

特にアイキャッチ画像APIの発見は、CloudFrontの403に惑わされず、「APIはどこかに存在するはずだ」と粘り続けた結果でした。推測でコードを書く前に、まず仕様を確かめる。エンジニアとしての基本を改めて実感した体験でした。

質問や感想があれば、ぜひコメントやSNSでお知らせください。

---

## 参考リンク

- [ProseMirror 公式ドキュメント](https://prosemirror.net/docs/)
- [Playwright 公式ドキュメント](https://playwright.dev/python/)
- [httpx 公式ドキュメント](https://www.python-httpx.org/)

---

**この記事が役に立ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じ悩みを持つ誰かの助けになります。