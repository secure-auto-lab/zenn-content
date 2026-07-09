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

Cloudflareが `530` を返している。ここから、**5層——後日談まで含めれば8層——に積み重なった根本原因を1枚ずつ剥がしていく**長い旅が始まりました。

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

## ⚡ 後日談：勝利宣言の3日後、サイトはまた落ちた

……ここまでが、当日の私が書いた「解決編」です。

**この記事をこのまま公開していたら、嘘になるところでした。**

`sleep infinity` は確かにあの日の犯人でした。でも「なぜWSLはそもそも不安定なのか」という一番下の問いには、実は答えていなかった。その代償は3日後、**本番サイトの全断**という形で返ってきます。ここからは、5層の下に隠れていた**残り3層**を剥がしきるまでの続編です。

### 第6層：WSLハングの真因は「ClickHouseの診断ログ135GB」

keepalive復旧後もWSLは1日数回ハングしました。`echo` すら8〜30秒返らない、カーネルレベルの無応答です。

仕方なく、2分ごとにサイトを監視して異常なら `wsl --shutdown` で自動復旧する**watchdogスクリプト**を書きました（これが後に牙を剥きます）。並行してハングの瞬間を計測すると——

- メモリ: PSI memory 0・swap 0 → **メモリは無罪**
- ディスク: **PSI io some 58% / full 32%**、読込20MB/sが数分継続 → **I/O飽和が真犯人**

犯人はClickHouseでした。`system.trace_log` **61GB**、`system.processors_profile_log` **58GB**——**TTL未設定の診断ログが計135GB**まで無限成長し、起動・マージのたびにWSL全体をI/O待ちに沈めていたのです。

```sql
-- 診断ログはTTLを設定しないと無限に成長する（これが135GBの正体）
ALTER TABLE system.trace_log MODIFY TTL event_date + INTERVAL 3 DAY;
```

TRUNCATE＋プロファイラ停止＋TTL設定で、**CH起動は数分→10秒、定常I/Oは58%→3%**。「WSLが不安定」の正体は、WSLでもDockerでもなく**その中の1コンテナのログ肥大**でした。

### 第7層：CPU 763%——真犯人は検索ボット（と、私の誤診断）

それでもまだ、マシンが重い。`docker stats` を見て目を疑いました。

```text
nexus-clickhouse   CPU 763.57%   ← 8コア中7.6コアを占有
load average: 33   ← 適正値は8以下
```

同じ重いクエリ（商品一覧、`FINAL`付き）が**10分間に2,050回**、完了より速く積み上がって11本同時実行。ここで私は**恥ずかしい誤診断**をします。

`docker logs nexus-lb --since 2m` が **0件** を返したため、「外部アクセスはゼロ。Next.jsが内部で勝手にページを再生成している」と結論づけたのです。ISR、instrumentation、cron、キャッシュハンドラ……アプリの内部を何時間も探しました。**全部シロでした。**

転機は `--since` を `--tail` に変えた瞬間です。

```bash
docker logs nexus-lb --tail 1000   # クロックに依存しない
```

```text
GET /ja/products/NHDTA-267  200  bingbot/2.0  40.77.167.47
GET /en/products/PBD-446    200  bingbot/2.0  40.77.167.47
GET /zh-TW/products/GIGP-32 200  bingbot/2.0  40.77.167.7
...
```

**直近1,000リクエスト中、bingbotが565件（64%）。** Microsoftの検索クローラーが3言語の商品ページを毎秒約0.6ページで総なめクロールしており、その1ページごとに `FINAL` 付きの重いクエリが9本走っていたのです。`--since` はコンテナとホストのクロック差で平気で取りこぼします。**私は「ログが0件」という嘘の証拠で、存在しない内部犯を追っていた。**

対処も1つ失敗しました。CHの同時実行数を32→4に絞ったら、**実ユーザーのページ描画まで `TOO_MANY_SIMULTANEOUS_QUERIES` で死んだ**のです（1ページの描画自体が並列クエリを11本撃つため）。即座に戻し、正解は「**ボットのUAだけ**レート制限」でした。

```nginx
# ボットUAだけ15req/分に制限。実ユーザーはキーが空になり無制限
map $http_user_agent $rl_crawler_key {
    default          "";
    "~*bingbot"      "crawler";
    "~*googlebot"    "crawler";
}
limit_req_zone $rl_crawler_key zone=crawler_zone:10m rate=15r/m;
limit_req_status 429;
server {
    limit_req zone=crawler_zone burst=8 nodelay;
    ...
}
```

検証結果：bingbot→429、人間のUA→全ページ200。load averageは33→8まで低下しました。SEOを守りながら（ブロックではなく減速）、CHへの砲撃だけを止められます。

### 第8層（最終ボス）：自動復旧watchdogが、本番を殺し続けていた

そして3日後の朝。**サイト全断。** 調べると、恐ろしいループが回っていました。

```text
CH起動に数分かかる（第6層の後遺症でマージ掃除が重い）
 → その間サイトは502 ＋ WSLも高負荷でコマンドが20秒返らない
 → watchdog「サイトダウン＋WSL無応答＝カーネルハングだ！」
 → wsl --shutdown 発動
 → 全コンテナ再起動 → CHまた数分の起動 → 最初に戻る（無限ループ）
```

**第6層の対策として自分で書いた自動復旧が、第8層の障害そのものになっていた**のです。watchdogを止めた瞬間、CHは一度で起動しきり、サイトは何事もなく200を返しました。

修正の鍵は「本物のカーネルハング」と「ただ遅いだけ」の判別です。

```powershell
# shutdownの前に docker inspect を60秒待って深層判定する
# → dockerが応答する = カーネルは生きている = shutdownしてはいけない
# → CHがhealth:startingなら「起動中」なので、何もせず待つ
$deep = Get-ChStateDeep   # docker inspect nexus-clickhouse (timeout 60s)
if ($null -ne $deep) {
    if ($deep -match 'starting') { Write-Log 'CH_STARTING_GRACE' '起動中。待つ。' }
    else { Write-Log 'WSL_SLOW_BUT_ALIVE' '遅いだけ。shutdownしない。' }
} else {
    # 60秒待ってもdockerが無応答 → ここで初めて本物のハングと認定
    wsl --shutdown
}
```

**自動復旧は「復旧対象の起動が遅い」場合、必ずグレース期間を持たないと自己増幅ループになります。**

### そして恒久策：`FINAL` を「消す」のではなく「不要にする」

最後に残った根本問題が、ページ描画のたびに走る `ReplacingMergeTree` の `FINAL` でした。このテーブルは更新をINSERTで行う設計なので、`FINAL`（クエリ時の重複統合）を単純に外すと**古いデータが混ざった嘘の結果**が返ります。外せない。

答えは ClickHouse の **Refreshable Materialized View** でした。

```sql
-- FINALは10分に1回だけ、MVのリフレッシュ時に実行する
-- ページ描画はFINALなしで最新版スナップショットを読む
CREATE MATERIALIZED VIEW nexus.products_master_latest
REFRESH EVERY 10 MINUTE
ENGINE = MergeTree ORDER BY product_id
AS SELECT * FROM nexus.products_master FINAL;
```

アプリ側は `FROM products_master FINAL` → `FROM products_master_latest` に置換するだけ（約40箇所）。鮮度は10分遅れになりますが、もともとアプリのキャッシュTTLが600秒だったので**体感は何も変わりません**。

デプロイ後の結果がこれです。

| 項目 | 対策前 | 対策後 |
|------|--------|--------|
| ClickHouse CPU | **763%**（8コア中7.6を占有） | **11〜16%** |
| 商品ページの描画 | 4.75秒＋積み上がり | **0.17秒** |
| `FINAL`付きクエリ | 数百回/分 | **0回**（10分に1回のMVリフレッシュのみ） |
| サイト | 断続的に502 | 200で安定 |

bingbotのクロールが来ても、もう何も溶けません。

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

### Tips 6: `docker logs --since` が0件を返す罠

`--since` はホストとコンテナのクロック差（UTC/JST等）で実在するログを取りこぼすことがある。「リクエスト0件」を根拠に判断する前に、クロック非依存の `--tail` で必ず裏を取る。

```bash
docker logs nexus-lb --since 2m 2>&1 | wc -l   # 0件でも信じない
docker logs nexus-lb --tail 1000               # こちらが真実
```

### Tips 7: 検索ボットだけレート制限するnginx設定

ボットのクロールでDBが飽和する場合、完全ブロックはSEOを壊す（deindexされる）。UAマップ＋`limit_req`で**減速だけ**させる。キーが空文字のリクエストは`limit_req`の対象外になる仕様を使い、実ユーザーは無制限のまま。

```nginx
map $http_user_agent $rl_crawler_key {
    default          "";        # 人間 → 空キー → 制限なし
    "~*bingbot"      "crawler";
    "~*googlebot"    "crawler";
    "~*semrushbot"   "crawler";
}
limit_req_zone $rl_crawler_key zone=crawler_zone:10m rate=15r/m;
limit_req_status 429;
server {
    limit_req zone=crawler_zone burst=8 nodelay;
    ...
}
```

注意: `return` で即応答するlocation（ヘルスチェック等）は `limit_req`（preaccessフェーズ）を通らないため、検証は**プロキシされるパス**で行うこと。

### Tips 8: ReplacingMergeTree の FINAL を Refreshable MV で除去する

`FINAL` は単純に外せない（重複と古い版が混ざる）。ClickHouse 24.1+ の Refreshable Materialized View で「10分に1回だけFINALを実行した最新版スナップショット」を持ち、読み側はそれを参照する。

```sql
CREATE MATERIALIZED VIEW db.products_latest
REFRESH EVERY 10 MINUTE
ENGINE = MergeTree ORDER BY product_id
AS SELECT * FROM db.products FINAL;
-- アプリは FROM products FINAL → FROM products_latest に置換
```

リフレッシュ状況は `system.view_refreshes` で確認できる。行数が `SELECT count() FROM products FINAL` と一致すれば正しく動いている。

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
5. **「解決した」と思った日から3日間は疑う**: 症状の消失と真因の解決は別物
6. **自動復旧を書いたら「対象が遅いだけの時」の挙動を必ずテストする**: グレースなき自動復旧は障害製造機になる

> 📌 まずは、あなたの障害の「1つ下の階層」を見に行ってください。
> ログを"時系列で"読むだけで、見えていなかった真因が浮かびます。


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/cloudflare-530-wsl-keepalive)**
