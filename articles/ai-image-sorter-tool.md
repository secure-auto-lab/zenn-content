---
title: "AI画像の選別が面倒すぎたので、Python 200行で生産性を3倍にした話"
emoji: "🖼️"
type: "tech"
topics: ["python", "ai", "comfyui", "automation"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/ai-image-sorter-tool"
---

![OGP](/images/ai-image-sorter-tool-ogp.png)


# AI画像の選別が面倒すぎたので、Python 200行で生産性を3倍にした話

**100枚の画像。そのうち使えるのは20枚。**

AI画像生成を使ったことがある人なら、この「選別地獄」に覚えがあるはずです。私はComfyUIでバッチ生成した画像を毎回エクスプローラで1枚ずつ開いて、右クリックして、フォルダに移動して……を繰り返していました。

**キーボード1つ叩くだけで、画像が一瞬で振り分けられたら？**

そんなツールをPython 200行で作りました。

---

## 💭 なぜ「キーボード駆動」にこだわったのか

実は、最初から今の設計を選んだわけではありません。

### 最初に考えた方法とその限界

まず考えたのは、GUIのボタンでの操作でした。「SAVEボタン」「NGボタン」のように画面に配置して、クリックで仕分けるパターンです。

しかし、すぐに問題に気づきました。

- **マウスを使う時点で遅い**: 画像を見る → マウスを移動 → ボタンをクリック → 次の画像を見る。この視線の往復が無駄
- **判断は一瞬なのに操作が追いつかない**: 人間が「この画像は使える/使えない」を判断するのは0.5秒。しかしマウス操作に2〜3秒かかる
- **誤クリックのリスク**: 隣のボタンを押し間違えた場合の取り消しが面倒

「画面を見る目線を一切動かさず、指だけで完結させたい」

この要件を満たすのは、キーボードショートカットしかありませんでした。

### 決め手になった3つの設計判断

1. **ホームポジションから動かないキー配置**: `S`（SAVE）、`N`（保留）、`D`（削除）——左手だけで完結
2. **即時フィードバック**: キーを押した瞬間にステータスバーの色が変わり、次の画像が表示される。操作の結果を目で確認する必要がない
3. **Undoは必須機能**: 高速操作では必ず間違えます。`Z`キーで直前の操作を元に戻せる安心感が、さらに振り分け速度を上げてくれます

**「速度を上げるために必要なのは、高速な処理ではなく、迷わないUI」** ——これが今回の最大の学びでした。

---

## 🔧 具体的な実装方法

### 全体アーキテクチャ

```
入力ディレクトリ（ComfyUI出力先など）
├── image_001.png      ← 未振り分け画像
├── image_002.png
├── ...
├── save/              ← Sキーで移動（採用）
├── hold/              ← Nキーで移動（保留）
└── trash/             ← Dキーで移動（削除）
```

設計はシンプルです。指定ディレクトリ内の画像を1枚ずつ全画面表示し、キー入力に応じてサブフォルダに `shutil.move` するだけ。出力フォルダは入力ディレクトリ内に自動作成されます。

### Step 1: 基本構造

```python
class ImageSorter:
    def __init__(self, input_dir: Path, extensions: set):
        self.input_dir = input_dir
        self.extensions = extensions

        # 出力フォルダを入力パス内に作成
        self.save_dir = input_dir / "save"
        self.hold_dir = input_dir / "hold"
        self.trash_dir = input_dir / "trash"
        for d in (self.save_dir, self.hold_dir, self.trash_dir):
            d.mkdir(exist_ok=True)

        self.images: list[Path] = []
        self.index = 0
        self._undo_stack: list[tuple] = []  # Undo用の操作履歴
```

ポイントは `_undo_stack` です。移動操作を記録するスタックで、`Z`キーでPOPして元に戻します。

### Step 2: キーボード駆動の画像表示

```python
# キーバインド
self.root.bind("s", lambda e: self._move_to("save"))
self.root.bind("n", lambda e: self._move_to("hold"))
self.root.bind("d", lambda e: self._move_to("trash"))
self.root.bind("z", lambda e: self._undo())
self.root.bind("<Right>", lambda e: self._next())
self.root.bind("<Left>", lambda e: self._prev())
self.root.bind("r", lambda e: self._refresh())
```

tkinterの `bind` でキーイベントをハンドリングするだけ。大文字・小文字の両方をバインドして、CapsLock状態でも動作するようにしています。

### Step 3: Undo機能の実装

これが最も重要な機能です。

```python
def _move_to(self, dest: str):
    src = self.images[self.index]
    dest_dir = {"save": self.save_dir, "hold": self.hold_dir, "trash": self.trash_dir}[dest]
    dst = dest_dir / src.name

    shutil.move(str(src), str(dst))

    # 操作履歴をスタックに記録
    self._undo_stack.append((src, dst, dest, self.index))

    self.images.pop(self.index)
    self._show_current()

def _undo(self):
    if not self._undo_stack:
        return

    original_path, moved_path, dest_key, original_index = self._undo_stack.pop()
    shutil.move(str(moved_path), str(original_path))

    # 元の位置にリストへ再挿入
    self.images.insert(original_index, original_path)
    self.index = original_index
    self._show_current()
```

**スタック方式にした理由**: `Ctrl+Z`のように、連続して複数回Undoできることが重要です。「3枚前に仕分けた画像が気になる」というケースは頻繁に起きます。スタックなら `Z` を3回押すだけで戻れます。

操作履歴には「元のパス」「移動先パス」「カテゴリ」「リスト内の位置」を記録しているため、ファイルの物理移動もリスト上の位置も完全に復元できます。

### Step 4: ホットリロード

```python
def _refresh(self):
    """入力ディレクトリから新しい画像を検出して追加"""
    existing_set = set(self.images)
    new_images = []
    for f in sorted(self.input_dir.iterdir()):
        if (f.is_file()
            and f.suffix.lower() in self.extensions
            and f not in existing_set
            and f.parent not in self._exclude_dirs):
            new_images.append(f)
    self.images.extend(new_images)
    return len(new_images)
```

AI画像生成では「バッチ生成しながら並行して振り分ける」ワークフローが一般的です。`R`キーで新しく生成された画像をその場で取り込めるので、生成完了を待つ必要がありません。

`_exclude_dirs` で `save/`、`hold/`、`trash/` を除外しているのがポイントです。振り分け済みの画像を再度読み込んでしまうのを防ぎます。

---

<!-- qiita-section -->

## 💡 実践Tips：AI画像振り分けツールの実装ポイント

### 完成コードの使い方

```bash
# インストール（Pillow のみ）
pip install Pillow

# 実行
python image_sorter.py /path/to/images

# 拡張子を限定
python image_sorter.py /path/to/images --extensions png webp
```

### キー操作一覧

| キー | 動作 |
|------|------|
| `S` | SAVE（採用）に移動 |
| `N` | HOLD（保留）に移動 |
| `D` | ゴミ箱（trash）に移動 |
| `Z` | 直前の操作をUndo |
| `→` / `Space` | スキップ |
| `←` | 前の画像に戻る |
| `R` | 新規画像を取り込む |
| `Q` / `Esc` | 終了 |

### Tips: 同名ファイルの衝突回避

ComfyUIなど一部のツールでは、異なるバッチで同じファイル名が生成されることがあります。

```python
# 同名ファイル対策
if dst.exists():
    stem = src.stem
    suffix = src.suffix
    i = 1
    while dst.exists():
        dst = dest_dir / f"{stem}_{i}{suffix}"
        i += 1
```

### Tips: tkinterで画像をキャンバスにフィットさせる

```python
ratio = min(canvas_width / img.width, canvas_height / img.height)
new_w = int(img.width * ratio)
new_h = int(img.height * ratio)
img = img.resize((new_w, new_h), Image.LANCZOS)
```

アスペクト比を維持しつつ、キャンバスの幅・高さの**小さい方**に合わせてリサイズします。`LANCZOS`フィルタで縮小時のジャギーを防止。

### Tips: PhotoImageのGC防止

tkinterのよくあるハマりポイントです。

```python
# NG: ローカル変数のPhotoImageはGCされて画像が表示されない
photo = ImageTk.PhotoImage(img)
canvas.create_image(0, 0, image=photo)

# OK: インスタンス変数に保持してGCを防ぐ
self._photo_ref = ImageTk.PhotoImage(img)
canvas.create_image(0, 0, image=self._photo_ref)
```

### カテゴリのカスタマイズ

コード中の `save` / `hold` / `trash` はあくまでデフォルトの命名です。用途に応じて自由に変更できます。

```python
# 例: 写真のコンテスト応募用
self.save_dir = input_dir / "finalist"   # S → 最終候補
self.hold_dir = input_dir / "maybe"      # N → 保留
self.trash_dir = input_dir / "reject"    # D → 不採用
```

<!-- /qiita-section -->

---

## 🚀 応用編：さらに上を目指すために

### 応用1: カテゴリの追加

現在は3カテゴリ（S/N/D）ですが、数字キーで5段階評価にすることも可能です。

```python
# 1〜5の数字キーでスター評価
for i in range(1, 6):
    self.root.bind(str(i), lambda e, star=i: self._move_to(f"star{star}"))
```

### 応用2: ファイル監視による完全自動化

`R`キーの手動リロードの代わりに、`watchdog`ライブラリでディレクトリを監視すれば、新しい画像が生成された瞬間に自動で取り込めます。

### 応用3: メタデータ連携

ComfyUIは画像のEXIFにワークフロー情報を埋め込みます。振り分け結果とEXIFメタデータを組み合わせれば、「どのプロンプト・モデル・シード値の組み合わせが良い画像を生むか」の分析にも使えます。

---

## ❓ よくある質問（FAQ）

### Q1: Windows以外でも動きますか？

A: Python + tkinter + Pillowが動く環境なら、macOS・Linuxでも動作します。tkinterはPythonに標準で含まれています。

### Q2: 大きな画像（4K以上）でも重くなりませんか？

A: 表示前にキャンバスサイズに合わせてリサイズしているため、元画像のサイズに関わらず軽快に動作します。

### Q3: 画像以外のファイル（動画など）がフォルダにあっても大丈夫ですか？

A: 拡張子フィルタ（デフォルト: png, jpg, jpeg, webp, bmp）に一致するファイルのみ読み込むため、他のファイルは無視されます。`--extensions` で対象を変更可能です。

---

## 📝 まとめ：今日からできるアクションプラン

この記事で解説した内容をまとめます：

1. **道具を作る**: AI画像の振り分けは汎用ツールではなく、専用ツールで3倍速になる
2. **キーボード駆動**: 視線を動かさず指だけで完結するUIが最速
3. **Undoは必須**: 速度を上げるなら、リカバリも同じ速度で

**今日からできる具体的なアクション：**

> 📌 Pillowをインストールして、普段使っているAI画像出力フォルダでツールを起動してみてください。
>
> ```bash
> pip install Pillow
> python image_sorter.py /path/to/your/ai-images
> ```
>
> 所要時間は約1分。次のバッチ生成から世界が変わります。

---

## 📚 参考リンク

- [ソースコード（GitHub）](https://github.com/secure-auto-lab)
- [Pillow公式ドキュメント](https://pillow.readthedocs.io/)
- [tkinter公式ドキュメント](https://docs.python.org/ja/3/library/tkinter.html)
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/ai-image-sorter-tool)**
