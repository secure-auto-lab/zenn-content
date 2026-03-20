---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第5回】ゼロサムにしないゲーム設計"
emoji: "💒"
type: "tech"
topics: ["supabase", "postgresql", "gamedesign", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-05-chip-economy"
---

![OGP](/images/wedding-it-05-chip-economy-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

## はじめに

初期チップ1,000。対戦で負けたら0。ゲームオーバー -- 結婚式二次会で、それは最悪のシナリオです。

[前回の記事](https://blog.secure-auto-lab.com/articles/wedding-it-04-slideshow)では、披露宴の写真をリアルタイムにスクリーンへ映し出すスライドショーシステムを作りました。今回からは**二次会篇**。カジノゲームの心臓部、「チップ経済圏」の設計に入ります。

この記事を読むと、以下のことがわかります。

- ゼロサムゲームを**ポジティブサム**に変える経済設計の考え方
- PostgreSQLトリガーによる**アトミックなチップ移動**の実装
- `source_ref` を使った**冪等性保証**で二重送金を防ぐ方法
- 酔っ払いでも使える**PayPay風送金UI**の設計判断

## 🎯 設計思想：「ゼロサムゲーム」にしない

### なぜゼロサムが問題なのか

新婦のために「全員が楽しめる場」を作りたい。でもカジノゲームは本質的にゼロサム -- 誰かが勝てば誰かが負ける。

対戦で負けた人のチップが勝者に移動するだけの仕組みだと、以下の問題が発生します。

1. **富の集中** -- 強いプレイヤーにチップが集まり続ける
2. **参加障壁** -- チップが減った人は賭けられなくなる
3. **心理的離脱** -- 「もう無理」と感じた人がゲーム自体をやめる

結婚式二次会で「もうチップないからゲームできない...」という人が出たら最悪です。新婦のために作ったシステムが場の空気を壊してしまう。

### 💭 なぜ「お金」ではなく「行動」に報酬を与えるのか

最初は「対戦に勝てばチップが増える」というシンプルな設計を考えていました。でも少し考えると、それは**ゲームが上手い人だけが楽しめる仕組み**でしかない。

結婚式二次会の本質は「みんなが交流して楽しむ場」です。だから報酬設計のゴールも「ゲームの勝者を決める」ことではなく、**「二次会を楽しむ行為そのものにインセンティブを与える」**ことだと気づきました。

写真を撮る、ドリンクを飲む、新郎新婦にメッセージを送る -- これらはすべて「二次会を楽しんでいる行為」です。であればこれら全てにチップ報酬をつけてしまえばいい。

### 解決策：複数のチップ獲得手段を用意する

対戦以外にもチップを得る方法を設計しました。

| 手段 | 獲得量 | 条件 |
|------|--------|------|
| 写真アップロード | +500 | 写真を撮ってアップロード |
| 写真に写る | +500 | AIが自分の顔を検出した写真 |
| 祝福メッセージ | +500 | 新郎新婦へメッセージを送信 |
| バー端末 | +500 | ドリンク注文時にQRスキャン |
| 対戦勝利 | +賭け金 | 相手のチップから移動 |
| スタンプ収集 | +500 | 新しい所属の人にチップを贈る |
| クエスト達成 | +500,000 | Royal Trinity 完了時 |

ポイントは、**対戦に参加しなくてもチップが増える**こと。二次会を楽しむ行為自体がチップ獲得に繋がります。

![チップの稼ぎ方を説明するルール画面](/images/chip-economy-how-to-earn.png)

これにより「対戦で負けてもチップは回復できる」という安心感が生まれ、ゲームへの参加ハードルが大幅に下がります。

### 🧱 インフレとの戦い：「消費」の設計

チップが増えすぎると価値が薄れます。最初のテストプレイでは、写真報酬だけで全員のチップが膨れ上がり、対戦の賭け金が相対的に無意味になりました。

獲得手段だけでなく、以下の「消費」も設計しました。

| 手段 | 消費量 | 効果 |
|------|--------|------|
| 対戦敗北 | -賭け金 | 相手に移動 |
| The Coliseum | -ベット額 | バトルロイヤル。負けるとベット額没収 |

特に面白いのが「祝福メッセージ」の設計です。当初は「チップを消費して新郎新婦に寄付する」案でしたが、最終的に**「メッセージを送ると+500チップ報酬」**という逆のアプローチに変えました。なぜか？

「チップを払ってメッセージを送る」だと、チップが少ない人がメッセージを躊躇してしまう。でも新郎新婦へのメッセージは全員に送ってほしい。だから**報酬にした**のです。インフレ対策はThe Coliseumの没収で担保し、祝福メッセージは「みんなに使ってもらう機能」として開放しました。

## 💸 PayPay方式のチップ送金UI

### 現場の制約を受け入れる

二次会の現場は「薄暗い」「ガヤガヤ」「片手にドリンク」「酔っ払っている」状況です。

複雑なUIは絶対に使えません。

そこで参考にしたのがPayPayの送金UIです。設計方針は以下の通り。

- システムは「財布」に徹する
- 勝敗判定は人間がリアルで行う
- 敗者が勝者にチップを送るだけ

```
┌─────────────────────────────────┐
│        [アバター]                │
│    山田 太郎 さんにチップを贈る    │
│                                 │
│          1000 chips             │
│                                 │
│ [500] [1000] [2000] [5000]      │
│                  [ All In ]     │
│                                 │
│     [1] [2] [3]                 │
│     [4] [5] [6]                 │
│     [7] [8] [9]                 │
│     [C] [0] [←]                 │
│                                 │
│     [ 1000 chips を送る ]        │
└─────────────────────────────────┘
```

QRコードで相手を特定し、テンキーで金額を入力して送信。酔っ払っていても片手で操作できるUIを目指しました。

### プリセットボタンの設計意図

テンキーに加えて `[500] [1000] [2000] [5000]` のプリセットボタンを用意しています。

```typescript
const PRESET_AMOUNTS = [500, 1000, 2000, 5000];
```

二次会の「じゃんけんで1000チップ！」のような場面では、プリセットをタップするだけで完了します。`[All In]` は赤いボタンで目立たせ、一発逆転の盛り上がりを演出します。

テンキーには**残高上限チェック**を組み込みました。所持チップを超える金額は入力できないため、「送りすぎ」事故を防げます。

```typescript
const handleNumberPad = (num: string) => {
  if (num === "C") {
    setAmount("");
  } else if (num === "←") {
    setAmount((prev) => prev.slice(0, -1));
  } else {
    const newAmount = amount + num;
    const numValue = parseInt(newAmount, 10);
    // 所持チップを超える入力はブロック
    if (numValue <= (sender?.current_chips || 0)) {
      setAmount(newAmount);
    }
  }
};
```

## 🔐 アトミックなチップ送金のDB設計

### transactionsテーブルによる一元管理

チップの増減はすべて `transactions` テーブルを経由します。直接 `profiles.current_chips` を UPDATE することはありません。

```sql
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id),
  amount INTEGER NOT NULL,
  game_type TEXT,
  source_ref TEXT,  -- 冪等性保証用
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 冪等性のためのユニークインデックス（NULLは許容）
CREATE UNIQUE INDEX idx_transactions_source_ref
  ON transactions(source_ref)
  WHERE source_ref IS NOT NULL;
```

`transactions` にINSERTすると、トリガーが自動的に `profiles.current_chips` を更新します。

```sql
CREATE OR REPLACE FUNCTION update_chips_after_transaction()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE profiles
  SET current_chips = current_chips + NEW.amount
  WHERE id = NEW.user_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_transaction_insert
  AFTER INSERT ON transactions
  FOR EACH ROW
  EXECUTE FUNCTION update_chips_after_transaction();
```

この設計には3つのメリットがあります。

1. **監査証跡** -- すべてのチップ移動が `game_type` 付きで記録される
2. **整合性** -- トリガーで自動計算するため残高の不整合が起きない
3. **拡張性** -- 新しいチップ獲得手段を追加するとき、`transactions` にINSERTするだけ

### なぜ直接UPDATEしないのか

「`profiles.current_chips` を直接 `+= 100` すればいいのでは？」と思うかもしれません。しかし以下の問題があります。

- 同時更新でレースコンディションが発生する
- 「なぜチップが減ったか」の履歴が残らない
- 不正操作の検出ができない

`transactions` テーブル経由にすることで、チップの流れが完全に追跡可能になります。実際、本番でも `game_type` でフィルタするだけで「写真報酬でいくら発行されたか」「対戦でいくら動いたか」が即座にわかりました。

### source_refによる冪等性保証

二次会の現場では、通信が不安定な場面もあります。送金ボタンを二度押しされたとき、チップが二重に移動してしまったら大問題です。

これを防ぐのが `source_ref` カラムです。

```typescript
// Server Action: チップ送金処理
const transferId = crypto.randomUUID();

// 送信側トランザクション（マイナス）
await serviceClient.from("transactions").insert({
  user_id: senderProfile.id,
  amount: -amount,
  game_type: "transfer_out",
  source_ref: `transfer:${transferId}:out`,
});

// 受信側トランザクション（プラス）
await serviceClient.from("transactions").insert({
  user_id: recipientId,
  amount: amount,
  game_type: "transfer_in",
  source_ref: `transfer:${transferId}:in`,
});
```

`source_ref` にはユニークインデックスがかかっているため、同じ `transferId` で2回INSERTしようとするとDBレベルでエラーになります。アプリケーション側のリトライロジックに頼らず、**DBが二重送金を根本的に防ぐ**設計です。

`source_ref` の命名規則は `{context}:{uuid}:{suffix}` で統一しており、ログを見れば「この送金がいつ・誰から・誰に」をすぐに特定できます。

## 🍹 バー端末システム

物理的な「チップ獲得ポイント」を会場に設置しました。

### 運用フロー

1. バーカウンターにタブレット端末を設置
2. ゲストがドリンクを注文
3. バーテンダーが端末でゲストのQRをスキャン
4. 500チップが自動付与
5. クールダウン画面が表示（5分間再取得不可）

### クールダウン方式のレート制限

「何杯も飲んでチップを稼ぐ」のを防ぐ必要があります。

最初は `UNIQUE(user_id, terminal_id)` 制約で「1人1端末1回」にする案を考えました。しかしテストプレイで気づいたのは、2時間の二次会で1回しか報酬がもらえないとドリンクに行くインセンティブが弱いということ。

そこで**クールダウン方式**を採用しました。5分間隔で何度でも報酬を受け取れます。

```typescript
// /bar/[terminalId]/page.tsx（Server Component）
const barRewards = await getSystemConfigValue(
  "bar_rewards",
  DEFAULT_VALUES.bar_rewards
);
const COOLDOWN_MINUTES = barRewards.cooldown_minutes; // 5分
const REWARD_AMOUNT = barRewards.reward_amount;       // 500チップ

// クールダウン判定：最後の報酬から5分経過しているか
const cooldownTime = new Date();
cooldownTime.setMinutes(cooldownTime.getMinutes() - COOLDOWN_MINUTES);

const { data: recentReward } = await supabase
  .from("bar_rewards")
  .select("created_at")
  .eq("user_id", userId)
  .gte("created_at", cooldownTime.toISOString())
  .order("created_at", { ascending: false })
  .limit(1)
  .single();

if (recentReward) {
  // クールダウン中 → 残り時間を表示
  return <CooldownScreen remainingTime={...} />;
}

// チップ付与
await supabase.from("bar_rewards").insert({
  user_id: userId,
  terminal_id: terminalId,
  amount: REWARD_AMOUNT,
});

await supabase.from("transactions").insert({
  user_id: userId,
  amount: REWARD_AMOUNT,
  game_type: "bar_reward",
  description: `バー端末 (${terminalId}) でボーナス獲得`,
});
```

このバー端末ページはNext.jsの**Server Component**で実装しています。ページにアクセスした瞬間にサーバーサイドでクールダウン判定→チップ付与→結果表示が完了するため、クライアント側でのAPIコールが不要。ネットワークが不安定な会場でもワンステップで処理が完結します。

### system_configによる柔軟な設定管理

報酬額やクールダウン時間はコード内にハードコードせず、`system_config` テーブルで管理しています。

```sql
-- system_config テーブルでバー報酬を管理
INSERT INTO system_config (key, value) VALUES
  ('bar_rewards', '{"cooldown_minutes": 5, "reward_amount": 500}');
```

これにより、本番中でも管理画面から報酬額を調整できます。「チップが足りなさそうだから報酬を上げよう」といった判断をデプロイなしで即座に反映可能です。

## 🗺️ スタンプ収集システム

チップ送金に**コレクション要素**を組み合わせた仕組みです。

新しい所属（「新婦同僚」「新郎大学友人」など）のゲストに初めてチップを贈ると、その所属の「スタンプ」を獲得し、+500チップのボーナスがもらえます。

```typescript
// 送金時にスタンプ判定（/app/pay/[code]/actions.ts）
const recipientData = await serviceClient
  .from("profiles")
  .select("affiliation, club")
  .eq("id", recipientId)
  .single();

// 過去の送金先から既に収集済みの所属を算出
const collectedAffiliations = new Set<string>();
for (const tx of pastTransfers) {
  // ... 過去の送金先プロフィールをチェック
}

// 新しい所属なら報酬付与
if (!collectedAffiliations.has(recipientData.affiliation)) {
  newStamps.push(recipientData.affiliation);
  await serviceClient.from("transactions").insert({
    user_id: senderProfile.id,
    amount: STAMP_REWARD_AMOUNT, // 500
    game_type: "stamp_reward",
    source_ref: `stamp:${transferId}:sender:${recipientData.affiliation}`,
    description: `新スタンプ「${recipientData.affiliation}」獲得ボーナス`,
  });
}
```

この仕組みの狙いは**グループを超えた交流の促進**です。「新郎の友人」と「新婦の同僚」は普通なら話す機会がありません。でもスタンプを集めたい人は自然と知らない人に話しかけることになります。

## 📊 経済圏のバランス調整

### リアルタイムで見えるチップの流れ

チップ経済のバランスは、ランキングダッシュボードでリアルタイムに可視化しています。

![リアルタイム更新されるランキングダッシュボード](/images/chip-economy-ranking-display.png)

個人チップランキング、所属別ランキング、チーム別スコアが一画面で確認でき、「チップが偏りすぎていないか」を運営側がリアルタイムで監視できます。

### シミュレーションで検証

本番前に40人規模のシミュレーションを行い、2時間後のチップ分布を確認しました。

チェックしたのは以下の指標です。

- **ジニ係数** -- チップ格差の度合い（0.4以下を目標）
- **最低チップ保有者** -- ゲームに参加不能になっていないか
- **最高チップ保有者** -- 一人勝ち状態になっていないか

写真撮影やバー端末の報酬額を調整し、「負けても2〜3回の行動で回復できる」バランスを目指しました。初期チップ1,000に対して各報酬が500なので、2回行動すれば元に戻れる計算です。

## イベント向けゲーム経済設計のパターン集

結婚式に限らず、イベント向けゲーム経済を設計する際に使えるパターンをまとめます。

### パターン1: マルチソース獲得

単一の獲得手段（対戦勝利のみ）ではなく、複数の獲得手段を用意する。ゲームに参加しなくても経済圏に参加できるようにする。

```typescript
// game_type で獲得手段を分類
type GameType =
  | "transfer_in"      // 対戦勝利
  | "photo_upload"     // 写真アップロード
  | "photo_tagged"     // 写真に写る
  | "bar_reward"       // バードリンク報酬
  | "blessing_reward"  // 祝福メッセージ
  | "stamp_reward"     // スタンプ収集
  | "quest_reward";    // クエスト達成
```

### パターン2: トランザクションログ方式

残高を直接UPDATEせず、増減をすべてログとして記録し、トリガーで残高を自動計算する。

```sql
-- トリガー関数：INSERTだけで残高が自動更新される
CREATE OR REPLACE FUNCTION update_chips_after_transaction()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE profiles
  SET current_chips = current_chips + NEW.amount
  WHERE id = NEW.user_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

メリット：監査・デバッグ・不正検出が容易。`game_type` でフィルタすれば「どの機能でいくらチップが動いたか」が一発でわかる。

### パターン3: source_refによる冪等性保証

ネットワーク不安定な環境での二重処理を防ぐ。`source_ref` カラムにユニークインデックスを設定し、同一リクエストの再実行をDBレベルで拒否する。

```sql
-- NULLは許容しつつ、値がある場合はユニークを保証
CREATE UNIQUE INDEX idx_transactions_source_ref
  ON transactions(source_ref)
  WHERE source_ref IS NOT NULL;
```

命名規則: `{context}:{uuid}:{suffix}`
- 送金: `transfer:{uuid}:out` / `transfer:{uuid}:in`
- スタンプ: `stamp:{uuid}:sender:{affiliation}`
- クエスト: `royal_trinity:{user_id}`

### パターン4: クールダウン方式のレート制限

`UNIQUE` 制約（1回限り）よりもクールダウン方式（時間間隔）の方が、繰り返し参加のインセンティブを維持できる。

```sql
-- クールダウン判定
SELECT created_at FROM bar_rewards
WHERE user_id = ? AND created_at >= NOW() - INTERVAL '5 minutes'
ORDER BY created_at DESC LIMIT 1;
```

### パターン5: system_configによるランタイム設定

報酬額・クールダウン時間をDBテーブルで管理し、デプロイなしで調整可能にする。

```sql
CREATE TABLE system_config (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  description TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 例：バー報酬の設定
INSERT INTO system_config (key, value) VALUES
  ('bar_rewards', '{"cooldown_minutes": 5, "reward_amount": 500}');
```

TypeScript側ではフォールバック値付きで取得：

```typescript
const barRewards = await getSystemConfigValue<BarRewardsConfig>(
  "bar_rewards",
  { cooldown_minutes: 5, reward_amount: 500 } // フォールバック
);
```
</qiita-section>

## 次回予告

チップ経済圏ができたところで、次はこのチップを使った**対戦システム**の実装に入ります。1vs1のQRスキャン対戦から、最大10人のバトルロイヤルまで、Supabase Realtimeを使ったリアルタイム同期の仕組みを解説します。

**次回：[【連載第6回】1vs1からバトルロイヤルまで - リアルタイム対戦システム](https://blog.secure-auto-lab.com/articles/wedding-it-06-battles)**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-05-chip-economy)**
