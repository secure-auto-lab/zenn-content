---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第6回】物理NFCチップと対戦システムの実装"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "nfc", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-06-battles"
---

![OGP](/images/wedding-it-06-battles-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

# 40枚のNFCチップが、デジタルとリアルを繋いだ

前回はチップ経済圏の設計──稼ぎ方、使い方、インフレ制御──を解説しました。

仕組みは整った。でも、足りないものがありました。

**「チップを賭けて対戦する」体験が、まだデジタルの中に閉じていたのです。**

画面の中だけでチップが動いても、二次会の「テーブルを囲んで勝負する」熱狂は生まれない。新婦のために作るカジノには、**手で触れられるチップ**が必要でした。

そこで、40枚の物理NFCチップを作ることにしました。

---

## 💰 物理NFCチップ：デジタルとリアルを繋ぐ鍵

### なぜ物理チップが必要だったのか

チップ経済圏の仕組み上、ゲストはスマホの画面で自分のチップ残高を確認できます。でも考えてみてください。

結婚式の二次会で、全員がスマホの画面を覗き込んでいる光景──**それ、楽しいですか？**

カジノの魅力は、チップを手に取り、テーブルに積み、「これを賭ける」と宣言する**身体的な体験**にあります。画面上の数字が増減するだけでは、その興奮は再現できません。

そこで発想を転換しました。

**「物理チップそのものが、デジタルの入り口になればいい」**

NFCタグを内蔵したカジノチップを作り、スマホでタッチするだけで対戦が始まる──デジタルとリアルが融合する体験を目指しました。

### 40枚のNFCチップの設計

物理チップは「デジタルIDの器」です。1枚1枚にユニークなコードが書き込まれたNFCタグが埋め込まれています。

```sql
-- 物理NFCチップのテーブル設計
CREATE TABLE chips (
  code TEXT PRIMARY KEY,                          -- NFCタグに書き込むコード
  owner_id UUID UNIQUE REFERENCES profiles(id),   -- 1ユーザー = 1チップ（UNIQUE制約）
  linked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

設計のポイントは **`owner_id` の `UNIQUE` 制約**です。

- 1人のゲストが持てる物理チップは**1枚だけ**
- 1枚のチップは**1人にしか紐づかない**
- これにより「他人のチップを奪う」「複数チップを集める」不正を構造的に防止

40枚のチップはマイグレーション時に自動生成されます。

```sql
-- 40枚の本番チップを生成
INSERT INTO chips (code) VALUES
  ('CHIP-001-' || gen_random_uuid()),
  ('CHIP-002-' || gen_random_uuid()),
  -- ... 40枚分
  ('CHIP-040-' || gen_random_uuid());
```

`CHIP-001-` のプレフィックスは人間が識別するため、末尾のUUIDは推測を防ぐため。**人にやさしく、セキュリティに厳しい**命名です。

### NFCタグのURL設計：`/c/[code]`

物理NFCチップをスマホにタッチすると、以下のURLが開きます。

```
https://wedding0406-kanami-jun.secure-auto-lab.com/c/CHIP-001-xxxx-xxxx
```

この `/c/[code]` ルートが、物理チップとデジタルシステムの**唯一の接点**です。

---

## 🔗 `/c/[code]` ── 1つのURLで4つのシナリオに対応

NFCチップをタッチしたとき、状況は4パターンあります。1つのページですべてを捌く設計にしました。

```
タッチ
  ├─ 未登録チップ × 未ログイン → 新規登録フロー（/onboarding/chip）
  ├─ 未登録チップ × ログイン済 → チップ自動紐付け → カジノへ
  ├─ 自分のチップ → カジノトップへリダイレクト
  └─ 他人のチップ → チップ送金ページ（/pay/{owner_id}）
```

**最後のパターンが、対戦の入口になります。**

「あいつのチップをスマホでタッチする」→「送金画面が開く」→「チップを賭けて対戦」

物理チップをタッチするという**身体的なアクション**が、デジタルの対戦を起動する。この体験設計がカジノの没入感を生みます。

### チップ紐付けのRPC関数

```sql
CREATE OR REPLACE FUNCTION claim_chip(p_code TEXT)
RETURNS JSONB AS $$
DECLARE
  v_user_id UUID;
  v_existing UUID;
  v_rows INT;
BEGIN
  v_user_id := auth.uid();

  -- 1. このユーザーが既に別のチップを持っていないかチェック
  SELECT owner_id INTO v_existing
  FROM chips WHERE owner_id = v_user_id;
  IF v_existing IS NOT NULL THEN
    RETURN jsonb_build_object('status', 'already_has_chip');
  END IF;

  -- 2. チップを紐付け（owner_id IS NULL のものだけ更新）
  UPDATE chips
  SET owner_id = v_user_id, linked_at = NOW()
  WHERE code = p_code AND owner_id IS NULL;

  GET DIAGNOSTICS v_rows = ROW_COUNT;

  IF v_rows = 0 THEN
    -- 更新できなかった = 既に誰かが持っている
    SELECT owner_id INTO v_existing FROM chips WHERE code = p_code;
    IF v_existing = v_user_id THEN
      RETURN jsonb_build_object('status', 'already_owned');
    ELSE
      RETURN jsonb_build_object('status', 'taken');
    END IF;
  END IF;

  RETURN jsonb_build_object('status', 'claimed');
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### レースコンディション対策

40人のゲストが同時にチップを手に取り、NFCをタッチする──最悪のケースでは、**同じチップを2人が同時にタッチ**します。

ここで `FOR UPDATE` は使っていません。代わりに `UPDATE ... WHERE owner_id IS NULL` のパターンを採用しています。

```
ユーザーA: UPDATE chips SET owner_id = 'A' WHERE code = 'CHIP-001' AND owner_id IS NULL;
  → 1行更新 → 成功

ユーザーB: UPDATE chips SET owner_id = 'B' WHERE code = 'CHIP-001' AND owner_id IS NULL;
  → 0行更新（既にAが所有）→ ROW_COUNTで検知 → "taken" を返す
```

`FOR UPDATE` でロックするよりもシンプルで、PostgreSQLのMVCC（Multi-Version Concurrency Control）に任せる方が安全です。ロック待ちでタイムアウトする心配もありません。

### フロントエンド：タッチしてからの3秒間

```typescript
// app/c/[code]/page.tsx（簡略版）
type ChipState = "loading" | "linking" | "linked" | "own" | "other" | "error";

export default function ChipPage({ params }: { params: { code: string } }) {
  const [state, setState] = useState<ChipState>("loading");

  useEffect(() => {
    (async () => {
      const { data: chipInfo } = await supabase.rpc("get_chip_info", {
        p_code: params.code,
      });

      if (!chipInfo.owner_id) {
        // 未登録チップ → 紐付け処理
        setState("linking");
        const { data } = await supabase.rpc("claim_chip", {
          p_code: params.code,
        });
        if (data.status === "claimed") {
          setState("linked");
          await delay(1500); // 演出用の待機
          router.push("/casino");
        }
      } else if (chipInfo.owner_id === currentUser.id) {
        setState("own");
        await delay(1000);
        router.push("/casino");
      } else {
        setState("other");
        await delay(1000);
        router.push(`/pay/${chipInfo.owner_id}`);
      }
    })();
  }, []);

  return (
    <div className="min-h-screen flex items-center justify-center">
      {state === "linking" && <p>🔗 チップを登録中...</p>}
      {state === "linked" && <p>✅ あなたのチップになりました！</p>}
      {state === "own" && <p>🎰 カジノへ移動します...</p>}
      {state === "other" && <p>💸 チップ送金画面へ...</p>}
    </div>
  );
}
```

NFCタッチから画面遷移まで**約1〜1.5秒**。短い待機と絵文字アニメーションで「何が起きたか」を直感的に伝えます。待機時間がゼロだと「本当に登録された？」と不安になるし、長すぎると離脱する。この微妙なバランスが体験の質を決めます。

---

## ⚔️ 1vs1対戦：相手のチップをタッチするだけ

### 対戦の始め方

1vs1対戦のフローは、驚くほどシンプルです。

```
1. 相手が持っている物理チップをスマホでタッチ（NFCスキャン）
2. → /c/[code] → 相手のチップと判定 → /pay/{owner_id} にリダイレクト
3. 賭けるチップ額を入力
4. 「送る」ボタンで完了
```

**「相手のチップをタッチする」という物理的なアクションが対戦の宣言になる。** これがリアルとデジタルの融合です。

### なぜ勝敗判定をシステムに入れないのか

「じゃんけん」「腕相撲」「ダーツ」「トランプ」──二次会では様々なゲームで対戦が発生します。これらすべてのルールをシステムに実装するのは非現実的です。

そこで割り切りました。

- システムは「財布」に徹する
- 勝敗判定は人間がリアルで行う
- **敗者が勝者にチップを送る**

この設計により、どんなゲームでもチップを賭けられるようになりました。「じゃんけんで100チップ」「ダーツで500チップ」──対戦の自由度が無限に広がります。

### チップ送金のアトミック性

前回解説した `transactions` テーブルを使い、送金処理は1つのINSERT文で完結します。

```sql
-- 送金処理（送り手と受け手を同時に記録）
INSERT INTO transactions (user_id, amount, game_type, source_ref)
VALUES
  (sender_id, -100, 'transfer_out', 'transfer:' || uuid || ':out'),
  (receiver_id, 100, 'transfer_in', 'transfer:' || uuid || ':in');
```

2つのレコードが1つのINSERT文で作成されるため、「送り手から引かれたのに受け手に入っていない」という不整合が発生しません。`source_ref` に共通のUUIDを含めることで、送金のペアを後から追跡できます。

---

## 🏟️ バトルロイヤル：The Coliseum

### コンセプト

1vs1が「テーブルでのカジュアルな賭け」なら、The Coliseumは**会場全体が熱狂するイベント**です。

最大10人が参加し、最後の1人になるまで戦うサバイバル形式。参加者がチップを賭け、勝者がポット（賭け金の合計）を総取りする**Winner-Takes-All方式**です。

### 対戦ルーム作成のフロー

```
1. 司会が「The Coliseum」の開始をアナウンス
2. 参加者がスマホからエントリー（賭け金を設定）
3. 定員（最大10人）に達したらゲーム開始
4. リアルで対戦（ゲーム内容は司会の裁量）
5. 敗者が脱落するたびに「脱落」ボタンを押す
6. 最後の1人が勝者 → ポット全額獲得
```

### ポット管理と分配ロジック

```typescript
// ポットの計算例（5人参加、各1000チップ）
const participants = 5;
const betPerPerson = 1000;
const totalPot = participants * betPerPerson; // 5000 chips

// 勝者がポット全額を獲得
// 勝者の収支: +4000 (5000 - 自分の1000)
// 敗者の収支: -1000 (各自)
```

負けた場合のリスクは「自分の賭け金だけ」ですが、勝った場合のリターンは「参加者数 - 1」倍。参加人数が多いほどリターンが大きくなる設計は、「もう1人参加して！」という自然なバイラルを生みます。

### 脱落処理とリアルタイム更新

脱落はプレイヤー自身が「脱落」ボタンを押す方式です。

```typescript
const handleElimination = async (roundId: string) => {
  await supabase
    .from("coliseum_participants")
    .update({ status: "eliminated", eliminated_at: new Date() })
    .eq("round_id", roundId)
    .eq("user_id", currentUser.id);
};
```

脱落するたびにSupabase Realtimeで全参加者の画面が更新され、「残り3人！」「残り2人！」とサバイバル感が演出されます。

---

## 📡 Supabase Realtimeによるリアルタイム同期

### Realtime + ポーリングのハイブリッド戦略

Supabase Realtimeは便利ですが、WebSocket接続は不安定になることがあります。二次会会場のWi-Fiは40人が同時接続する過酷な環境。接続が切れたときにゲームが止まるのは致命的です。

そこで、Realtimeとポーリングのハイブリッド戦略を採用しました。

```typescript
useEffect(() => {
  const channel = supabase
    .channel("ranking-updates")
    .on(
      "postgres_changes",
      { event: "*", schema: "public", table: "profiles" },
      () => fetchRankings()
    )
    .subscribe((status) => {
      if (status === "SUBSCRIBED") {
        setConnectionStatus("online");
      } else if (status === "CHANNEL_ERROR") {
        // Realtime失敗時はポーリングにフォールバック
        setConnectionStatus("polling");
        pollingInterval = setInterval(fetchRankings, 5000);
      }
    });

  return () => {
    channel.unsubscribe();
    if (pollingInterval) clearInterval(pollingInterval);
  };
}, []);
```

### なぜハイブリッドなのか

| 方式 | メリット | デメリット |
|------|---------|----------|
| Realtimeのみ | 即時反映 | 接続断で更新停止 |
| ポーリングのみ | 安定 | 最大5秒の遅延 |
| ハイブリッド | 即時反映 + 接続断に耐性 | 実装がやや複雑 |

Realtimeが正常なときは即時反映、接続が切れたら5秒間隔のポーリングにフォールバック。ユーザーには `connectionStatus` をUIに表示して、現在の接続状態を透明にしています。

---

## 🏆 リアルタイムランキングシステム

### チーム対抗戦：RED vs BLUE

二次会を通して、ゲストはRED / BLUEの2チームに分かれて競います。チーム分けは**NFCチップの紐付け時に自動**で行われます。

```sql
CREATE OR REPLACE FUNCTION assign_team_if_needed()
RETURNS VOID AS $$
DECLARE
  v_red_count INTEGER;
  v_blue_count INTEGER;
BEGIN
  -- 現在の各チームの人数を取得
  SELECT
    COUNT(*) FILTER (WHERE team_color = 'RED'),
    COUNT(*) FILTER (WHERE team_color = 'BLUE')
  INTO v_red_count, v_blue_count
  FROM profiles;

  -- 少ない方のチームに割り当て
  UPDATE profiles SET team_color =
    CASE
      WHEN v_red_count < v_blue_count THEN 'RED'
      WHEN v_blue_count < v_red_count THEN 'BLUE'
      ELSE CASE WHEN random() < 0.5 THEN 'RED' ELSE 'BLUE' END
    END
  WHERE id = auth.uid() AND team_color IS NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**常に少ない方のチームに割り当て**る設計で、人数バランスを自然に保ちます。

物理チップを受け取り、NFCをタッチした瞬間に「あなたはREDチーム！」と表示される──受付の時点からチーム対抗の意識が芽生えます。

### ランキング表示の工夫

ランキングはスマホとスクリーン（会場プロジェクター）の2つのビューを用意しました。

**スマホ版:** 自分の順位をハイライト、チーム別/全体の切り替え

**スクリーン版:** 上位10名を大きく表示、チーム合計チップの対抗グラフ、リアルタイム順位変動アニメーション

Realtimeでリアルタイムに更新されるため、ランキング表示中に対戦が行われると**順位がその場で入れ替わる**演出が可能です。

---

## 🍺 バー端末との連携

前回紹介したバー端末は、チップ経済圏のインフレ調整装置であると同時に、**対戦への参加を促すタッチポイント**でもあります。

バーでドリンクを受け取る → 500チップ獲得 → 「せっかくだから対戦しよう」

```typescript
const BarTerminal = ({ terminalId }: { terminalId: string }) => {
  const handleReward = async (userId: string) => {
    await supabase.rpc("give_bar_reward", {
      p_user_id: userId,
      p_terminal_id: terminalId,
    });
  };

  return <QRScanner onScan={(userId) => handleReward(userId)} />;
};
```

バーカウンターには専用のNFCカードを設置。ゲストがバーNFCにタッチするだけでチップがもらえる簡単オペレーションです。5分間のクールダウンで不正を防止しつつ、「バーに行くとチップがもらえる」という回遊導線を作りました。

---

## 🎫 スタンプラリー：NFCチップが「交流装置」になる

### 着想：水曜日のダウンタウン「部活カジノ」

このカジノシステムの原点は、2026年2月放送の「水曜日のダウンタウン」で登場した**部活カジノ**企画です。街行く人の学生時代の部活を体型や服装から予想してチップをBETする──この「人の属性を当てる面白さ」にピンときました。

結婚式の二次会でも、ゲスト同士が知らない人と交流するきっかけが欲しい。でも**クイズ形式は不公平**です。新郎新婦との関係が深い人ほど有利になり、二次会から参加した人や面識が薄い人が楽しめません。

そこで考えたのが**スタンプラリー方式**。知識ではなく「いろんな人と交流したか」で報酬が決まる仕組みです。

### 仕組み：チップを贈ると自動でスタンプ判定

オンボーディング時に、ゲストは**名前・所属（新郎友人、新婦同僚など）・部活**を登録します。部活は水曜日のダウンタウンにインスパイアされた要素で、「テニス部」「サッカー部」「帰宅部」などから選びます。

**チップを贈った相手の「所属」や「部活」がまだコレクションにない場合、新しいスタンプとして獲得。ボーナスで+500チップ。**

```typescript
// チップ送金時のスタンプ判定ロジック（抜粋）

// 1. 送信者の過去の送金相手の所属・部活を取得
const pastInteractions = await getPastTransferPartners(senderId);
const collectedAffiliations = new Set(pastInteractions.map(p => p.affiliation));
const collectedClubs = new Set(pastInteractions.map(p => p.club));

// 2. 受取人の所属・部活が未収集かチェック
const receiver = await getProfile(receiverId);

if (!collectedAffiliations.has(receiver.affiliation)) {
  // 新しい所属スタンプ → +500チップ
  await grantStampReward(senderId, "affiliation", receiver.affiliation);
}
if (receiver.club && !collectedClubs.has(receiver.club)) {
  // 新しい部活スタンプ → +500チップ
  await grantStampReward(senderId, "club", receiver.club);
}

// 3. 受取人側にも同様にスタンプ判定（双方向）
```

報酬は `source_ref` の冪等性で二重付与を防止。前回解説した `transactions` テーブルのパターンがここでも活きています。

### なぜスタンプラリーがNFCチップと相性がいいのか

スタンプを集めるには「知らない人にチップを贈る」必要があります。でも、知らない人にいきなり「チップください」とは言いにくい。

ここで**物理NFCチップが効いてきます**。

```
1. 「じゃんけんで勝負しませんか？」（対戦のきっかけ）
2. じゃんけん → 勝者のチップをタッチ → チップ送金
3. → 送金時に自動でスタンプ判定
4. 「あ、新しいスタンプ獲得！+500チップ」
5. 「あなた何部だったんですか？」（会話が生まれる）
```

**物理チップのタッチが対戦のきっかけを作り、対戦がスタンプラリーの進捗を生み、スタンプラリーが次の交流への動機を生む。** このループが、新郎側・新婦側の垣根を超えた交流を自然に促します。

### スタンプラリー画面

```
┌─────────────────────────────────────┐
│  スタンプラリー  進捗: 7/12          │
│  ████████░░░░░░░░░ 58%              │
│                                     │
│  【所属】                           │
│  ✅ 新郎友人  ✅ 新婦友人            │
│  ✅ 新郎同僚  ？ 新婦同僚            │
│  ✅ 新郎親族  ？ 新婦親族            │
│                                     │
│  【部活】                           │
│  ✅ サッカー部  ✅ テニス部           │
│  ✅ 帰宅部    ？ 吹奏楽部           │
│  ？ バスケ部   ？ 美術部             │
│                                     │
│  💡 新婦同僚のゲストを探して         │
│     チップを贈ろう！                 │
└─────────────────────────────────────┘
```

未収集のスタンプが「？」で表示されるため、「新婦同僚の人、まだ話してないな」と**次に話しかけるべき相手が可視化**されます。

---

## 📐 NFCとWebアプリの統合で意識したこと

### 1. 物理タッチを「体験の起点」にする

NFCタッチ → URLオープン → アプリ起動。この3ステップをユーザーに意識させない速度で処理する。1.5秒以内にフィードバックを返すことで、「タッチした→何かが起きた」の因果関係を体感させる。

### 2. 同じチップを何度タッチしても壊れない

冪等性（Idempotency）は物理デバイス連携の生命線。「自分のチップを何度タッチしてもOK」「紐付け済みチップをタッチしてもエラーにならない」──ユーザーが不安にならない設計が重要。

### 3. QRコードを保険として用意する

NFCタグは読み取り失敗することがあります。スマホのNFC位置がわかりにくい、タグの性能差、ケースとの干渉──本番で「読めない！」は致命的。

そこで全チップにQRコードも印刷し、同じURLにアクセスできるようにしました。**NFC + QRのデュアルインターフェース**が安心感を生みます。

<!-- qiita-section -->
## NFC × Webアプリの実装パターン

### パターン1: NFCタグ → URL → Webアプリの一気通貫
物理NFCタグにURLを書き込み、タッチでWebアプリを起動する。URLのパスパラメータにユニークコードを含め、1つのページで複数シナリオに対応する。

### パターン2: `UPDATE ... WHERE ... IS NULL` でのレースコンディション防止
`FOR UPDATE` ロックの代わりに、WHERE条件で状態チェックと更新を1ステートメントで行う。`ROW_COUNT` で成功/失敗を判定。PostgreSQLのMVCCに任せることでデッドロックリスクを回避。

### パターン3: Realtime + ポーリングのハイブリッド
Supabase Realtimeの `subscribe` コールバックで接続状態を監視し、`CHANNEL_ERROR` 時にポーリングにフォールバック。不安定なネットワーク環境（イベント会場等）では必須のパターン。

### パターン4: 冪等なNFCタッチ処理
同じNFCタグを複数回タッチしても安全に処理する設計。`ON CONFLICT DO NOTHING` や状態チェック＋既存レコード返却で、ユーザーが不安にならないUXを実現。

### パターン5: NFC + QRのデュアルインターフェース
NFC読み取り失敗時の保険としてQRコードを併用。同一URLにアクセスするため、バックエンドの実装は共通。物理デバイス連携では信頼性のフォールバックが必須。

### パターン6: トランザクション副作用としてのコレクション判定
チップ送金のトランザクション処理内で、送金相手の属性を自動判定し、未収集であればボーナスを付与する。`source_ref` の冪等性で二重付与を防止。メイン処理（送金）に副作用（スタンプ報酬）を組み込むことで、ユーザーに追加操作を求めずにゲーム性を実現。
<!-- /qiita-section -->

## 次回予告

物理NFCチップで対戦の入口を作り、スタンプラリーで交流の動機を生み、1vs1とバトルロイヤルで「自由対戦」の仕組みが揃いました。

次回は少し時間を巻き戻して、**披露宴の話**をします。席次カードにNFCを仕込み、受付で手渡すだけでゲスト登録が完了する仕組み。そして披露宴で作られたセッションが、二次会のカジノにそのまま引き継がれる認証設計を解説します。

**次回：【連載第7回】席次カードにNFCを仕込んだら、受付が感動体験になった**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-06-battles)**
