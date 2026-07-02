---
title: "Cloudflareの530を追ったら真犯人はWSLの sleep infinity だった｜多層障害デバッグ全記録"
emoji: "🕵️"
type: "tech"
topics: ["wsl", "cloudflare", "docker", "trouble", "cloudflared"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/cloudflare-530-wsl-keepalive"
---

![OGP](/images/cloudflare-530-wsl-keepalive-ogp.png)


# Cloudflareの530を追ったら、真犯人はWSLの `sleep infinity` だった

---

## 😰 あなたもこんな「多層障害」に溺れたことはありませんか？

- 「エラーメッセージを直しても直しても、別の場所から次のエラーが出てくる…」
- 「ログでは『正常に接続しました』と出ているのに、ユーザーには繋がらない…」
- 「原因を1つ潰すたびに、実はその下にもう1つ原因が隠れていた…」

**私はこの日、まさにこの沼に何時間も沈みました。**

きっかけは、たった一言の報告でした。

> 「本番サイトに繋がらないです。Cloudflare tunnelに繋がっていません」

Cloudflareが `530` を返している。ここから、**5層に積み重なった根本原因を1枚ずつ剥がしていく**長い旅が始まりました。

---

## 💭 なぜ「トンネルの再起動」では直らなかったのか

このインシデントで最も大きな学びは、**「動いている証拠」を鵜呑みにしなかったこと**でした。

### `/ready = 4 connections` という"嘘"

`cloudflared` のメトリクスエンドポイントはこう言っていました。

```json
{"status":200,"readyConnections":4,"connectorId":"..."}
```

「4本のコネクションがready」。ローカルのログにも `Registered tunnel connection connIndex=0..3` と、成功が並んでいます。普通ならここで「トンネルは正常。原因は別だ」と判断してしまいます。

**でも、Cloudflareのダッシュボードは正反対のことを言っていました。**

- ステータス: **停止**
- アクティブなレプリカ: **0**

ローカルの「正常です」と、edge側の「停止しています」。この**矛盾こそが真実への扉**でした。片方の指標だけを信じていたら、永遠に origin側を疑い続けて迷宮入りしていたはずです。

### ユーザーの「ClickHouseが怪しい」を、あえて否定した

途中でこんな報告も来ました。

> 「ClickHouseと接続できていないようです。ページは開くがランキングが表示されない。IPが変わっていないか確認してください」

もっともらしい仮説です。実際、ghost attachment（コンテナがネットワークから外れてDNSで引けなくなる）は過去に起きていました。

でも、**推測で動かず証拠を取りに行きました**。

- ClickHouseのIP → `172.21.0.10` のまま、**変わっていない**
- コンテナのhealth → `healthy`
- origin（nginx LB経由）で `/ja` を叩く → **ランキングデータを含む56KBを正常に描画**

つまり、**ClickHouseもoriginも無実**。「ページは開くがランキングが出ない」の正体は、CHではなく **Cloudflareトンネルの530**——ページの枠だけがCDNキャッシュで開き、データ取得がトンネル経由で失敗していたのです。

**もっともらしい仮説ほど、証拠で殺す。** これが遠回りに見えて最短でした。

### 決め手：「なぜCloudflareはレプリカ0と言うのか」を問い直す

「cloudflaredは繋がっている。でもCloudflareは繋がっていないと言う」——この矛盾を放置せず、**cloudflaredのログを"時系列"で読み直した**のが決定打でした。

そこに、犯人の指紋がありました。

```text
15:06:12 INF Registered tunnel connection connIndex=2 ...
15:06:13 INF Registered tunnel connection connIndex=3 ...
15:06:44 INF Initiating graceful shutdown due to signal terminated ...  ← 31秒後にSIGTERM
15:07:21 INF Registered tunnel connection connIndex=0 ...                ← また登録
15:07:29 INF Initiating graceful shutdown due to signal terminated ...  ← また8秒後にSIGTERM
```

**登録 → 30〜60秒後にSIGTERM → 再登録** の無限ループ。トンネルは「繋がっては殺され」を繰り返していて、Cloudflareが安定したレプリカとしてカウントできる瞬間が一度も無かった。だから edge は `1033` を返し続けていたのです。

---

## 🔧 具体的な調査と修正の全手順

### 全体像：症状から真因へ剥がしていく流れ

```text
[530] サイト不通
  │
  ├─ WSL自体が応答なし (HCS_E_CONNECTION_TIMEOUT)
  │    → wsl --shutdown で復帰（が、ループ継続）
  │
  ├─ p9io poweroffループ (docker起動15秒後にpower off)
  │    → /etc/wsl.conf automount=false（85%減）
  │    → wsl --update --pre-release（2.9.3でVM安定）
  │
  ├─ ClickHouse ghost attachment / nginx stale upstream
  │    → docker network connect / nginx -s reload
  │
  └─ nexus-tunnel が30-60秒ごとにSIGTERM
       → 原因: keepalive(sleep infinity)喪失
       → Start-ScheduledTask WSLKeepAlive で復旧 → 200
```

### Step 1: WSLが「Running」でも死んでいることを見抜く

`wsl --list` は `Running` と表示するのに、中のコマンドが返らない。まずはここを疑います。

```bash
# 25秒待って返らなければ rc=124（ハング確定）
timeout 25 wsl.exe -d Ubuntu -e bash -lc 'echo ALIVE' ; echo "rc=$?"
```

`rc=124` ならディストロがハング。`wsl --shutdown` で一度クリーンにします。

### Step 2: p9io poweroffループを特定する

ジャーナルに「docker.service起動 → 約15秒後にpower off」のパターンが出ていれば `#13435`。

```bash
# 直近のpoweroff回数とp9ioエラーを数える
sudo journalctl --since '-20 min' | grep -cE 'power off now'
sudo journalctl --since '-20 min' | grep -cE 'AcceptAsync|p9io'
```

`p9io.cpp:258 (AcceptAsync) Operation canceled` が連発していれば確定です。

### Step 3: 9P境界を削り、WSLを更新する

`/etc/wsl.conf` でWindowsドライブの自動マウント（9P）を切ると、クラッシュ面が激減します。

```ini
[boot]
systemd=true

[automount]
enabled = false
mountFsTab = false
```

そして本命の修正はプラットフォーム更新です。

```powershell
wsl --update --pre-release   # #13435 修正を含む新ビルドへ
wsl --shutdown               # 適用
```

### Step 4: 「ローカルは正常、edgeは停止」の矛盾を確認する

`cloudflared` の `/ready` と、Cloudflareダッシュボードの**両方**を見ます。

```bash
# ローカル側（同一Dockerネットワークの別コンテナから）
docker exec nexus-lb sh -c "wget -qO- http://nexus-tunnel:2000/ready"
# → {"status":200,"readyConnections":4} でも安心しない
```

ダッシュボード（Networks → Tunnels）で **アクティブなレプリカ / ステータス**を確認。ここが `0 / 停止` なら、**ローカルの"ready"は信用してはいけません**。

### Step 5: 真犯人 —— SIGTERMループとkeepalive喪失

トンネルログに `graceful shutdown due to signal terminated` が周期的に出ていたら、コンテナが外部からSIGTERMされ続けています。今回の犯人は、WSLディストロを保持し続ける `sleep infinity` プロセスの消失でした。

```bash
# keepalive(sleep infinity)が生きているか
ps -ef | grep '[s]leep infinity'    # 0個なら黒
```

WSLは、ディストロを保持するプロセスが無いと数十秒周期でpoweroffを試み、その巻き添えで `docker.service` が停止し、トンネルがSIGTERMで死にます。復旧は、既存のkeepaliveタスクを起動するだけ。

```powershell
Start-ScheduledTask -TaskName 'WSLKeepAlive'
Start-ScheduledTask -TaskName 'WSL2-KeepAlive'
```

keepaliveが入ればループが止まり、ループが止まればkeepaliveも死ななくなる（**自己安定**）。トンネルが数分安定すると、Cloudflareがレプリカを認識し、`530` は `200` に変わります。

---

## 💡 実践Tips・よくあるエラーと解決法

<!-- qiita-section -->

WSL2 + Docker + cloudflared 構成で「サイトが落ちた」時に、上から順に叩くと切り分けが速いコマンド集です。

### Tips 1: WSLが「Running」でも死んでいるか判定する

`wsl --list` の表示は当てになりません。中で1コマンド叩いて、返らなければハングです。

```bash
timeout 25 wsl.exe -d Ubuntu -e bash -lc 'echo ALIVE' ; echo "rc=$?"
# rc=124 → ハング。wsl --shutdown でクリーン再起動
```

### Tips 2: WSLの p9io poweroffループ（microsoft/WSL #13435）

`docker.service` 起動の直後に `systemd-logind: The system will power off now!` が周期的に出るのが特徴。

```bash
# 直近20分のpoweroff回数（多発していればループ）
sudo journalctl --since '-20 min' | grep -cE 'power off now'
# p9ioエラー
sudo journalctl --since '-20 min' | grep -E 'AcceptAsync|p9io' | tail
```

対処は「9P境界を削る」＋「WSL更新」の合わせ技。

```ini
# /etc/wsl.conf … Windowsドライブの自動マウント(9P)を無効化
[automount]
enabled = false
mountFsTab = false
```

```powershell
wsl --update --pre-release   # #13435修正を含む新ビルドへ
wsl --shutdown
```

### Tips 3: cloudflared が「ローカルはready、edgeは停止」

トンネルの `/ready` が正常でも、Cloudflareダッシュボードで **アクティブなレプリカ=0 / ステータス=停止** なら、コンテナが**周期的にSIGTERMされている**可能性大。ログの時系列を見る。

```bash
# 登録直後にSIGTERMが出ていないか
docker logs nexus-tunnel --since 10m 2>&1 \
  | grep -E 'Registered|graceful shutdown|signal terminated'
```

`Registered → 数十秒後に graceful shutdown due to signal terminated` の反復が見えたら、外部からコンテナが殺され続けている。

### Tips 4: WSLディストロが数十秒周期で落ちる → keepaliveを確認

`wsl --shutdown` を繰り返した後などに起きがち。ディストロを保持する `sleep infinity` が消えると、WSLが idle poweroff を繰り返し、Docker→トンネルが巻き添えで落ちる。

```bash
# keepaliveが生きているか（0個なら黒）
ps -ef | grep '[s]leep infinity'
```

```powershell
# 既存のkeepaliveタスクを起動して復旧（Windows側）
Start-ScheduledTask -TaskName 'WSLKeepAlive'
Get-Process wsl | Measure-Object   # プロセスが増えれば保持できている
```

keepaliveを入れるとpoweroffループが止まり、Cloudflareがレプリカを再認識して `530→200` に戻ります。

### Tips 5: ClickHouse ghost attachment（DNSで引けない）

コンテナがネットワークから外れて、固定IPを失いDNS解決できなくなる現象。

```bash
# CHがネットワークに正しいIPで居るか
docker inspect nexus-clickhouse \
  --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}({{$v.IPAddress}}){{end}}'
# 空 or 想定外IPなら ghost。再接続する:
docker network disconnect -f nexus-network nexus-clickhouse
docker network connect --ip 172.21.0.10 nexus-network nexus-clickhouse
docker compose restart nexus-aggregator nexus-web-blue nexus-web-green
```

<!-- /qiita-section -->

---

## ❓ よくある質問（FAQ）

### Q1: `wsl --update` が効かないと聞きましたが？

A: バグ修正がまだそのビルドに載っていない時期だと効きません。今回は `--pre-release` で当日リリースの新ビルド（2.9.3）を掴んだのが効きました。**更新前に、その修正がどのバージョンで入ったかを確認**するのが確実です。

### Q2: `automount=false` にするとWSLから `/mnt/c` が使えなくなりませんか？

A: なります。ただしコンテナのバインドマウントが `/mnt/c` に依存していないなら実害は小さいです（プロジェクトをext4側に置いていれば問題なし）。必要な時だけ `\\wsl$` 経由で参照するか、設定を戻します。

### Q3: keepaliveって普段は意識しなくていいのでは？

A: 通常のPC再起動ならブート時にkeepaliveタスクが自動起動するので不要です。**問題になるのは「手動で `wsl --shutdown` を何度も打った後」**。この時だけは手動でkeepaliveを起動し直してください。

---

## 📝 まとめ：今日からできるアクションプラン

多層障害を最短で解くための順番です。

1. **表面のエラーを"影"だと疑う**: 530は結果。原因は数階層下にあると仮定する
2. **メトリクスは視点を確認**: 「ローカルからの正常」と「相手からの正常」は別物
3. **仮説は証拠で殺す**: IP・health・直接叩き、で容疑者を1つずつ外す
4. **自分の操作ログも時系列に並べる**: 真犯人は自分の副作用かもしれない

> 📌 まずは、あなたの障害の「1つ下の階層」を見に行ってください。
> ログを"時系列で"読むだけで、見えていなかった真因が浮かびます。


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/cloudflare-530-wsl-keepalive)**
