---
title: "1ステップ19分の地獄。RTX 5090を自動化パイプライン3本で取り合うと何が起きるのか｜診断から恒久対策まで"
emoji: "🔥"
type: "tech"
topics: ["ollama", "comfyui", "gpu", "llm", "python"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/gpu-contention-ollama-gateway"
---

![OGP](/images/gpu-contention-ollama-gateway-ogp.png)


# 1ステップ19分の地獄。RTX 5090を自動化パイプライン3本で取り合うと何が起きるのか

---

## 📖 30秒でわかるこの記事のまとめ

むずかしい言葉を使わずに言うと、こういう話です。

> 私のパソコンの中では、**「絵を描くAI」「文章を書くAI」「調べ物をするAI」の3人が働いています**。ところが作業に必要な「机」(GPUという部品)は**1つしかありません**。
>
> ある日、3人が同時に机を使おうとして大混乱に。誰もエラー(悲鳴)を上げないまま、全員の仕事が**ほとんど止まってしまいました**。絵を1枚描くのに5時間半。7時間かけた作業が消えたこともありました。
>
> この記事は、**「誰が机を占領しているのか」を突き止める方法**と、**「順番待ちのルール」を作って二度と混乱しないようにした方法**の記録です。

「AIを2つ以上、同じパソコンで動かしている(動かしたい)人」に、いつか必ず役立つ話です。

---

## 📚 前提知識：この記事に出てくるもの

事前知識がなくても読めるように、登場するものを先に説明します。知っている方は読み飛ばしてください。

| 用語 | ざっくり説明 |
|------|-------------|
| **GPU** | AIの計算を高速にこなす部品。この記事では「作業机」に例えています。1台のPCに1枚が普通 |
| **VRAM** | GPUに直結した専用メモリ(=机の広さ)。私のGPU(RTX 5090)は32GB。**AIモデルはここに載らないと本来の速度が出ません** |
| **ローカルLLM** | ChatGPTのような文章AIを、クラウドではなく自分のPCで動かすもの。1つで19〜21GBものVRAMを使います |
| **Ollama(オラマ)** | ローカルLLMを動かすための定番ソフト。PCの中で「窓口(ポート11434)」を開いて、いろんなプログラムからの依頼を受け付けます |
| **ComfyUI** | 画像生成AIを動かすための定番ソフト。こちらも大量のVRAMを使います |
| **LoRA学習** | 画像AIに特定の絵柄や人物を覚えさせる追加学習。数時間かかる重い処理です |
| **パイプライン** | 「自動で動き続ける一連の処理」のこと。この記事では、記事の自動生成・データ収集・画像生成という3つの自動処理が登場します |
| **Docker** | プログラムを「コンテナ」という箱に入れて動かす仕組み。箱の中からもOllamaの窓口に依頼が飛んできます |
| **ゲートウェイ(プロキシ)** | 窓口の前に立つ「受付係」。依頼を中継しつつ、必要なら「今は順番待ちです」と待たせることができます。今回の解決策の主役です |

**押さえてほしいポイントは1つだけ**: VRAM(机の広さ)は32GBしかないのに、文章AI(21GB)と画像AI(20GB超)を同時に載せようとすると溢れる、ということです。溢れたときに何が起きるかが、この記事の本題です。

---

## 🗺️ この記事の構成

1. **何が起きたか** — 実際の事故の記録(数字つき)
2. **どう考えて解決したか** — 対策の発想と設計
3. **具体的な実装** — コードと手順(コピペ可能)
4. **明日から使えるTips** — 調査コマンド集と応急処置

---

### 事故1：画像生成が「1ステップ19分」になる

ある日、画像生成バッチの進捗が止まりました。エラーはゼロ。プロセスは生きている。でも12時間で生成枚数が1枚も増えていない。

ComfyUIのログを見て目を疑いました。

```
90%|████████▉ | 18/20 [5:30:56<37:46, 1133.31s/it]
```

**1ステップ1133秒（約19分）**。通常は1秒前後なので、およそ1100倍の遅さです。1枚の画像に5時間半かかっていました。

原因はVRAMの取り合いでした。LLMが19〜21GBを占有した状態で20Bクラスの画像モデルを動かすと、モデルの一部がVRAMから追い出され、システムメモリとの間でスワップが発生します。**クラッシュせず、エラーも出さず、ただ絶望的に遅くなる**。これがGPU競合の最も厄介な性質です。

### 事故2：7時間20分の学習が全損する

さらに手痛かったのはLoRA学習です。VRAM 31.9/32.6GBという限界張り付き状態で7時間20分学習を続けた末、**CUDAエラーで突然死**。途中保存を設定していなかったため、7時間分の計算が完全に消えました。

学習速度の劣化も顕著でした。競合なしで1.2it/s出ていた学習が、競合下では13.6s/itまで低下。**16倍遅い**うえに、いつクラッシュするか分からない爆弾を抱えて走り続けることになります。

### 決め手になった3つのポイント

1. **無改修**: クライアントは今まで通り11434を叩くだけ。新パイプラインも自動的に対象になる
2. **選択的**: 生成系エンドポイントだけ保留し、`ollama ps` 等の軽量APIは素通しにできる
3. **可観測**: 誰が何を要求し、何が保留されているかがゲートウェイのログ1箇所に集まる

**「N個のクライアントを直すより、1個のチョークポイントを作る」** —— これがこの経験で最も大きな学びでした。

---

## 🔧 具体的な実装方法

### 全体アーキテクチャ

```
heavy-worker / collector / chat UI / 今後の全パイプライン
        │  (全て :11434 のまま無改修)
        ▼
┌──────────────────────────────┐
│ Ollamaゲートウェイ (:11434)  │ ← 生成系のみロック中は保留
│  - /api/chat, /api/generate  │    /api/tags 等は素通し
│  - ロック: 期限付きJSON       │
└──────────────┬───────────────┘
               ▼
        Ollama本体 (:11435)

  画像生成/学習側は開始前にロックを acquire、終了後に release
```

### Step 1: 期限付きロックファイル

デッドロック対策として**ロックには必ず有効期限を入れます**。取得側がクラッシュしても、期限が切れれば自動的に無視されます。

```python
# gpu_lock.py (抜粋)
LOCK_PATH = Path(r"C:\Users\you\gpu_lock\exclusive.lock")

def is_locked() -> dict | None:
    if not LOCK_PATH.exists():
        return None
    d = json.loads(LOCK_PATH.read_text(encoding="utf-8"))
    if datetime.fromisoformat(d["expires_at"]) < datetime.now():
        return None  # 期限切れは無視 → デッドロックしない
    return d

def acquire(owner: str, reason: str, hours: float = 3.0) -> bool:
    LOCK_PATH.write_text(json.dumps({
        "owner": owner, "reason": reason,
        "expires_at": (datetime.now() + timedelta(hours=hours)).isoformat(),
    }), encoding="utf-8")
```

ポイントは**単一ファイルではなくディレクトリごと運用する**こと。Dockerコンテナに見せる場合、ファイル単体のbindマウントは削除・再作成でマウントが壊れるため、ディレクトリをread-onlyマウントします。

### Step 2: ゲートウェイ本体（aiohttp）

```python
# ollama_gateway.py (核心部の抜粋)
GOVERNED_PREFIXES = (
    "/api/generate", "/api/chat", "/api/embed", "/api/embeddings",
    "/v1/chat/completions", "/v1/completions", "/v1/embeddings",
)

async def handle(request: web.Request) -> web.StreamResponse:
    # 生成系だけロック中は保留(30秒ポーリング)
    if any(request.path.startswith(p) for p in GOVERNED_PREFIXES):
        while (d := lock_active()) is not None:
            await asyncio.sleep(30)

    # あとは素直にストリーミング転送
    body = await request.read()
    async with session.request(
        request.method, UPSTREAM + request.path_qs, data=body or None,
        headers=filtered_headers(request),
    ) as up:
        out = web.StreamResponse(status=up.status, headers=filtered_headers(up))
        await out.prepare(request)
        async for chunk in up.content.iter_chunked(8192):
            await out.write(chunk)
        await out.write_eof()
        return out
```

注意点は2つ。**タイムアウトを外す**こと（LLM生成は分単位、`ClientTimeout(total=None)`）と、**ストリーミングをそのまま中継する**こと（チャットUIの逐次表示が壊れないように）。

### Step 3: Ollama本体の退避とカットオーバー

```powershell
# Ollama本体を11435へ (ユーザー環境変数)
setx OLLAMA_HOST "127.0.0.1:11435"
# Ollamaを再起動 → ゲートウェイを11434で起動
```

これで全クライアントは無改修のままゲートウェイ経由になります。切り戻したい場合は環境変数を戻すだけです。

### Step 4: 運用

```bash
# 画像生成・学習の前に専有宣言(所要時間+余裕で期限を切る)
python gpu_lock.py acquire image-pipeline "LoRA学習" 3

# 終わったら解放
python gpu_lock.py release image-pipeline

# 状態確認
curl http://127.0.0.1:11434/gateway/status
```

---

## 💡 実践Tips・よくあるエラーと解決法

<!-- qiita-section -->

### 症状: GPUを使う処理がエラーなしで極端に遅くなる

VRAM競合のサイレント劣化を疑ってください。確認手順:

```bash
# 1. VRAM使用量と使用率
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv

# 2. Ollamaのモデル常駐状況(keep-aliveが更新され続けていないか)
ollama ps
# UNTIL が「4 minutes from now」のまま長時間変わらない
# → 誰かが継続的にリクエストしている証拠
```

### Tips: Ollama(11434)を使っているプロセスを特定する

Ollamaのログにはクライアント情報が残らないため、TCP接続から逆引きします。

```powershell
# 11434への接続元PIDを列挙
netstat -ano | findstr :11434 | findstr ESTABLISHED

# PIDからプロセスとコマンドラインを特定
Get-CimInstance Win32_Process -Filter "ProcessId = <PID>" |
  Select-Object ProcessId, Name, CommandLine
```

- 接続元が `com.docker.backend.exe` → **Dockerコンテナ内のクライアント**。`docker ps` で稼働コンテナを確認
- 接続元が `python.exe` → CommandLine でどの自動化ジョブか判別

### Tips: モデルのロード履歴を確認する

```powershell
# Ollamaサーバーログからランナー起動時刻を抽出
Select-String -Path "$env:LOCALAPPDATA\Ollama\server.log" -Pattern "starting llama-server" |
  Select-Object -Last 5
```

いつからモデルが常駐しているか(=占有時間)が分かります。

### Tips: 応急処置コマンド

```bash
# LLMを即アンロード(VRAM解放)
ollama stop <model名>

# Dockerコンテナを一時停止/再開(プロセスは保持される)
docker pause <container>
docker unpause <container>
```

`docker pause` はプロセス状態を保ったまま凍結するので、キューやジョブの状態を壊さずにGPUを空けられます。

### Tips: 32GB VRAMで20Bクラスの学習を安定させる(musubi-tuner)

```
--fp8_base --fp8_scaled --blocks_to_swap 16
--save_every_n_steps 100 --save_last_n_steps 300
```

- `fp8_scaled` + `blocks_to_swap` でVRAM約30GB→24GBに(限界張り付きを回避)
- 途中保存がないと外因クラッシュで全損する(実体験: 7時間20分が消えました)

### 恒久対策: Ollamaゲートウェイ(全クライアント無改修)

```
クライアント群(:11434のまま) → ゲートウェイ(:11434) → Ollama本体(:11435)
```

1. `setx OLLAMA_HOST "127.0.0.1:11435"` でOllama本体を退避
2. aiohttpの薄いプロキシを11434に常駐(生成系パスのみ、ロックファイル存在中は保留)
3. GPU専有したい処理は期限付きロックを書く(期限切れ自動無視でデッドロックなし)

ポイント:
- プロキシは `ClientTimeout(total=None)` (LLM生成は分単位)
- ストリーミングは `iter_chunked` でそのまま中継
- `/api/tags` `/api/ps` は素通しにして運用コマンドを生かす
- Dockerに見せるロックはファイル単体でなく**ディレクトリを**read-onlyマウント

<!-- /qiita-section -->

---

## ❓ よくある質問（FAQ）

### Q1: ゲートウェイが単一障害点になりませんか？

A: なります。対策として (1) 異常終了時に自動再起動するランナー経由で常駐、(2) 各パイプライン側にもロック確認をフォールバックとして残す、(3) 切り戻し手順(環境変数を戻すだけ)を文書化、の3点を入れています。

### Q2: vLLMやLM Studioでも使えますか？

A: 考え方はそのまま使えます。要は「全クライアントが通る唯一のポートの前段に、保留機能つきプロキシを置く」だけなので、OpenAI互換APIを話すサーバーなら同じ構成が組めます。

### Q3: Kubernetesやジョブスケジューラを入れるべきでは？

A: 正攻法はそうです。ただ個人の1台マシンに対しては過剰で、「今日動いている自動化を無改修で調停できる」ことを優先しました。パイプラインがさらに増えたら本格的なスケジューラへの移行を検討します。

---

## 📝 まとめ：今日からできるアクションプラン

1. **まず現状把握**: `nvidia-smi` と `ollama ps` で、誰がVRAMを使っているか確認する
2. **犯人特定の手順を手元に**: `netstat -ano | findstr :11434` → PID逆引きをブックマーク
3. **長時間処理に途中保存を入れる**: 学習は `save_every_n_steps`、バッチは逐次保存+冪等化
4. **競合が2回起きたら恒久対策**: ゲートウェイ+期限付きロックの構成を検討する

> 📌 まずは `ollama ps` を実行して、UNTILの表示が更新され続けていないか見てみてください。所要時間は10秒です。

---

## 📚 参考リンク

- [Ollama公式ドキュメント](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [aiohttp Server ドキュメント](https://docs.aiohttp.org/en/stable/web.html)
- [musubi-tuner](https://github.com/kohya-ss/musubi-tuner)


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/gpu-contention-ollama-gateway)**
