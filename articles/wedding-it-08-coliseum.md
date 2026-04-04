---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第8回】40人が一斉参加する「みんなで大予想」の設計と実装"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "postgresql", "gamedesign"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-08-coliseum"
---

![OGP](/images/wedding-it-08-coliseum-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

# 「ケーキ入刀、どっちが先に食べさせる？」──40人が一斉にチップを乗せた瞬間

前回は披露宴の席次カードにNFCを仕込み、受付からプロフィール登録までを1タッチで繋げる設計を解説しました。

さて、披露宴の話はここまで。いよいよ二次会の**最大の盛り上がりポイント**に踏み込みます。

二次会には「ケーキ入刀」「ファーストバイト」「ブーケトス」といった定番イベントがあります。でも、普通はゲストは**見ているだけ**です。写真を撮って、拍手して、終わり。

ここに**「大予想」**を組み込んだら何が起きるか。

「ケーキ入刀でどちらが先に食べさせるか」に自分のチップを乗せた瞬間、ゲストはもう**観客ではなくプレイヤー**になります。結果がどちらに転ぶかで自分のチップが変わる。だから叫ぶし、応援するし、結果発表の瞬間に歓声が上がる。

**受動的な「見ているだけ」の時間を、能動的な「予想している」時間に変える。** これが「みんなで大予想」の設計思想です。

---

## 💭 なぜ「パリミュチュエル方式」を選んだのか

### そもそもパリミュチュエル方式とは？

聞き慣れない言葉ですが、実は日本人のほとんどが体験したことがある仕組みです。**競馬の馬券**がまさにこれ。

全員のチップをひとつの「プール」に集めて、当たった人に山分けする方式です。人気の馬は配当が低く、穴馬は配当が高い。この仕組みを結婚式の大予想に転用しました。

### ブックメーカー方式との違い

予想システムには大きく2つの方式があります。

| | ブックメーカー（固定オッズ） | パリミュチュエル（変動オッズ） |
|--|--|--|
| オッズ決定 | 運営が事前に設定 | ベット総額から自動計算 |
| 運営の負担 | オッズ設定が必要 | 不要（市場原理で決まる） |
| 運営の取り分 | 必要 | 不要 |
| ベット中のオッズ | 変わらない | リアルタイムで変動する |

**パリミュチュエル方式を選んだ理由は3つ：**

1. **運営コストゼロ**: 新郎新婦がオッズを考える必要がない。結婚式の準備で忙しいのに「ファーストバイトのオッズは2.5倍でいいかな…」なんて考えたくない
2. **運営の取り分が不要**: 結婚式で運営が儲ける必要はない。全てのチップをゲスト間で分配する
3. **リアルタイムの興奮**: ベットが集まるほどオッズが変わる。スクリーンに映るオッズが動くたびに「今が狙い目だ！」と駆け込みベットが起きる

### パリミュチュエルの基本式

```
オッズ = プール総額 ÷ 勝利選択肢のベット総額
```

```
例：プール総額 130,000 chips
├── 新郎を選んだ人: 100,000 chips
└── 新婦を選んだ人:  30,000 chips

新婦が勝った場合：
オッズ = 130,000 ÷ 30,000 = 4.33x
→ 1,000 chips 乗せた人は 4,330 chips を獲得（利益3,330）

新郎が勝った場合：
オッズ = 130,000 ÷ 100,000 = 1.30x
→ 1,000 chips 乗せた人は 1,300 chips を獲得（利益300）
```

**少数派を選んだ人ほど大きなリターンを得る。** この逆転要素がゲームを盛り上げます。

---

## 🌱 odds_seed：最初の1人目の問題を解決する

### 実装して気づいた致命的な問題

パリミュチュエル方式をそのまま実装してテストしたとき、問題に気づきました。

**誰もチップを乗せていない状態のオッズが表示できない。**

スクリーンに「新郎: ?x / 新婦: ?x」と表示されても、誰も最初にベットしたくありません。最初の1人はオッズが不明で、2人目以降のベットでオッズが乱高下する。

さらに、最初のベットが1つの選択肢に集中すると、反対側のオッズが極端に高くなり、バランスが崩壊します。

### odds_seed（シード金）という解決策

そこで導入したのが **odds_seed** です。各選択肢に仮想的なベット額を加算し、初期状態から安定したオッズを表示できるようにしました。

```sql
-- coliseum_matches テーブル
odds_seed INT DEFAULT 5000  -- 各選択肢に仮想5000chipsを加算
```

```
【odds_seed なし】最初の1ベット
  新郎: 1000 chips → オッズ計算不能（新婦側が0）
  新婦: 0 chips

【odds_seed = 5000 あり】最初の1ベット
  新郎: 1000 + 5000 = 6000 chips
  新婦: 0 + 5000 = 5000 chips
  合計: 11000

  新郎オッズ = 11000 / 6000 = 1.83x
  新婦オッズ = 11000 / 5000 = 2.20x  ← 1人もいないが表示可能
```

**odds_seedは画面表示とオッズ計算にのみ使い、実際のチップ配布には影響しません。** あくまで「見えないバランサー」として機能します。ベットが増えるほどseedの影響は薄まり、最終的にはプレイヤーの意思がオッズを決定します。

管理画面からイベントごとにodds_seedを調整できるため、参加者が少ないイベントではseedを大きく（安定重視）、参加者が多いイベントではseedを小さく（市場原理重視）と柔軟に対応できます。

---

## 🔒 最低オッズ保証：1.5x

圧倒的にベットが偏った場合──例えば新郎に9割、新婦に1割──多数派を選んで当たってもオッズが1.1xでは面白くありません。「当たったのに100チップしか増えなかった」ではガッカリです。

そこで**最低オッズ1.5x**を保証しました。

```sql
v_raw_odds := v_virtual_total::FLOAT / v_virtual_winning::FLOAT;
v_final_odds := GREATEST(v_raw_odds, v_match.min_odds);
```

通常のパリミュチュエルでは、配当総額がプール総額を超えることはありません。しかし最低オッズ保証があると超える可能性があります。

**結婚式ではこれでOK。** チップはシステム内通貨なので、配当がプールを超えても経済は回ります。「勝ったら嬉しい」を保証する体験設計を、正確な数学より優先しました。

---

## 🔄 試合の状態遷移：lockedが生む「ドキドキの間」

```
prepared → open → locked → finished
    ↓                ↓
 cancelled       cancelled
```

| 状態 | 画面表示 | ゲストのスマホ |
|------|---------|--------------|
| prepared | ─（非表示） | ─ |
| open | 🔥 ベット受付中 | ベットモーダルが自動表示 |
| locked | 🔒 締切 | ベット不可、オッズ確認のみ |
| finished | ✅ 結果発表！ | 結果 + 配当表示 |
| cancelled | ─ | 返金通知 |

### なぜ locked 状態が必要なのか

`open` から直接 `finished` にすると2つの問題があります。

1. **滑り込みベット**: 結果が見えた瞬間にベットする不正の余地が生まれる
2. **演出の喪失**: 「もうベットできない、でも結果はまだわからない」というドキドキの時間がない

`locked` を挟むことで、締切後〜結果発表までの**数十秒の緊張感**を演出できます。

```
新郎新婦の操作フロー：
1. 管理画面で「受付開始」→ open
2. 2〜3分待つ（スクリーンでオッズが動く）
3. 「受付締切」→ locked（ドキドキの間）
4. イベント実行（ケーキ入刀など）
5. 「結果確定」→ 正解を選択 → finished
```

全てワンクリック。新郎新婦が会場を盛り上げながら、スマホ片手で操作できます。

---

## ⚛️ ベット処理のDB関数：40人同時アクセスに耐える設計

ここからは技術的な実装に踏み込みます。

Supabaseでは**RPC関数**（Remote Procedure Call＝データベース内で直接実行される関数）を使うことで、「残高チェック→チップ減算→ベット記録」を**1つの処理として丸ごと実行**できます。途中で別の処理が割り込む隙がないため、40人が同時にベットしてもデータが壊れません。

```sql
CREATE OR REPLACE FUNCTION place_coliseum_bet(
  p_match_id UUID,
  p_user_id UUID,
  p_option_id INT,
  p_amount INT
) RETURNS JSONB AS $$
DECLARE
  v_current_chips INT;
  v_match RECORD;
BEGIN
  -- 1. 試合取得 + ステータス確認
  SELECT * INTO v_match
  FROM coliseum_matches WHERE id = p_match_id;

  IF v_match IS NULL THEN
    RETURN jsonb_build_object('success', false, 'error', 'match_not_found');
  END IF;

  IF v_match.status != 'open' THEN
    RETURN jsonb_build_object('success', false, 'error', 'betting_closed');
  END IF;

  -- 2. 既存ベットチェック（1試合1ベット制）
  IF EXISTS (
    SELECT 1 FROM coliseum_bets
    WHERE match_id = p_match_id AND user_id = p_user_id
  ) THEN
    RETURN jsonb_build_object('success', false, 'error', 'already_bet');
  END IF;

  -- 3. 残高取得 + 排他ロック
  SELECT current_chips INTO v_current_chips
  FROM profiles WHERE id = p_user_id FOR UPDATE;

  IF v_current_chips < p_amount THEN
    RETURN jsonb_build_object('success', false, 'error', 'insufficient_chips');
  END IF;

  -- 4. ベット記録
  INSERT INTO coliseum_bets (match_id, user_id, option_id, amount)
  VALUES (p_match_id, p_user_id, p_option_id, p_amount);

  -- 5. トランザクション記録（トリガーでchips減算）
  INSERT INTO transactions (user_id, amount, game_type, description)
  VALUES (p_user_id, -p_amount, 'coliseum_bet', '大予想ベット');

  -- 6. プール更新
  UPDATE coliseum_matches
  SET total_pool = total_pool + p_amount
  WHERE id = p_match_id;

  RETURN jsonb_build_object('success', true);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
-- SECURITY DEFINER = この関数は「管理者権限」で実行される
-- → 一般ユーザーはこの関数経由でしかベットできない（直接DBを触れない）
```

### 設計のポイント

**`FOR UPDATE` による残高ロック**: `FOR UPDATE` は「この行を今から書き換えるから、他の処理は待って」とデータベースに宣言する仕組みです。40人が同時にベットしても、Aさんの残高チェック中にBさんが同じ残高を読んでしまう事故が起きません。ロック対象は個人の行だけなので、他のユーザーのベットはブロックしません。

**1試合1ベット制**: テスト時に「少額で様子見 → 大額追加」のオッズ操作が発生したため、1ベットに制限。一発勝負の緊張感のほうが盛り上がる。

**全チェックがDB関数内で完結**: アプリ側でチェックしてからDBに書くと、チェックと書き込みの間に他の処理が割り込む隙が生まれます。DB関数内で「確認→実行」を一息で行うことで、この隙を完全に排除しています。

---

## 🏆 配当計算：正解が決まった瞬間に何が起きるか

```sql
CREATE OR REPLACE FUNCTION resolve_coliseum_match(
  p_match_id UUID,
  p_winner_option_id INT
) RETURNS JSONB AS $$
DECLARE
  v_match RECORD;
  v_num_options INT;
  v_winning_pool BIGINT;
  v_virtual_total BIGINT;
  v_virtual_winning BIGINT;
  v_raw_odds FLOAT;
  v_final_odds FLOAT;
  v_bet RECORD;
  v_payout INT;
  v_total_payout BIGINT := 0;
  v_winner_count INT := 0;
BEGIN
  SELECT * INTO v_match FROM coliseum_matches
  WHERE id = p_match_id AND status = 'locked'
  FOR UPDATE;

  -- 選択肢の数を取得
  v_num_options := jsonb_array_length(v_match.options);

  -- 勝利選択肢のベット合計
  SELECT COALESCE(SUM(amount), 0) INTO v_winning_pool
  FROM coliseum_bets
  WHERE match_id = p_match_id AND option_id = p_winner_option_id;

  -- 的中者がいない場合
  IF v_winning_pool = 0 THEN
    UPDATE coliseum_matches
    SET status = 'finished',
        winner_option_id = p_winner_option_id,
        final_odds = 0
    WHERE id = p_match_id;
    RETURN jsonb_build_object(
      'success', true, 'final_odds', 0,
      'winner_count', 0, 'total_payout', 0
    );
  END IF;

  -- odds_seed を加味したオッズ計算
  v_virtual_total := v_match.total_pool
    + (v_match.odds_seed * v_num_options);
  v_virtual_winning := v_winning_pool + v_match.odds_seed;

  v_raw_odds := v_virtual_total::FLOAT / v_virtual_winning::FLOAT;
  v_final_odds := GREATEST(v_raw_odds, v_match.min_odds);

  -- 各的中者にペイアウト
  FOR v_bet IN
    SELECT * FROM coliseum_bets
    WHERE match_id = p_match_id AND option_id = p_winner_option_id
  LOOP
    v_payout := FLOOR(v_bet.amount * v_final_odds);

    UPDATE coliseum_bets SET payout = v_payout
    WHERE id = v_bet.id;

    INSERT INTO transactions (user_id, amount, game_type, description)
    VALUES (v_bet.user_id, v_payout, 'coliseum_win', '大予想 勝利');

    v_total_payout := v_total_payout + v_payout;
    v_winner_count := v_winner_count + 1;
  END LOOP;

  UPDATE coliseum_matches
  SET status = 'finished',
      winner_option_id = p_winner_option_id,
      final_odds = v_final_odds
  WHERE id = p_match_id;

  RETURN jsonb_build_object(
    'success', true,
    'final_odds', v_final_odds,
    'winner_count', v_winner_count,
    'total_payout', v_total_payout,
    'total_pool', v_match.total_pool
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

`FLOOR`（小数点以下切り捨て）で端数を処理しているのは意図的です。切り上げにすると配当総額がプール+odds_seed分をさらに超えてしまい、インフレが加速します。

---

## 📺 スクリーン表示：会場を巻き込む演出装置

会場のプロジェクターに投影する `/coliseum/screen` は、大予想の演出の要です。

```
┌──────────────────────────────────────────────────┐
│  🔥 ベット受付中                                   │
│  「ケーキ入刀：どちらが先に食べさせる？」            │
├──────────────────────────────────────────────────┤
│                                                    │
│    🔴 新郎              🔵 新婦                    │
│     1.50x                4.33x                     │
│   100,000 chips         30,000 chips               │
│    (25名)                (10名)                    │
│                                                    │
├──────────────────────────────────────────────────┤
│  [█████████████████░░░░░░░░] 77% ← → 23%          │
│  総プール: 130,000 chips                            │
├──────────────────────────────────────────────────┤
│  🏆 ハイローラー                                    │
│  🥇 山田太郎 5,000  🥈 鈴木花子 3,000  🥉 ...      │
└──────────────────────────────────────────────────┘
```

### ハイローラー表示の意図

上位10名の高額ベッターを名前付きで表示しています。これは**意図的な仕掛け**です。

自分の名前が大画面に映ることで「もっとチップを乗せたい」という心理が生まれる。高額ベットが増えるとプール総額が膨らみ、オッズ変動が活発になり、ゲーム全体の盛り上がりが加速する──**自己強化ループ**です。

### Realtime + ポーリングのハイブリッド

スクリーン表示は `coliseum_matches` と `coliseum_bets` の両テーブルをRealtimeで購読し、変更があるたびにデータを再取得します。

```typescript
const channel = supabase
  .channel("coliseum-screen")
  .on("postgres_changes",
    { event: "*", schema: "public", table: "coliseum_matches" },
    () => fetchData()
  )
  .on("postgres_changes",
    { event: "*", schema: "public", table: "coliseum_bets" },
    () => fetchData()
  )
  .subscribe();

// フォールバック：5秒間隔ポーリング
const interval = setInterval(fetchData, 5000);
```

第6回で解説したハイブリッド戦略がここでも活きています。

### 結果発表のアニメーション

`locked → finished` への遷移を検知すると、5秒間の結果発表アニメーションが表示されます。

```typescript
// 前回のステータスと比較して遷移を検知
if (prevStatus === "locked" && match.status === "finished") {
  setShowCelebration(true);
  setTimeout(() => setShowCelebration(false), 5000);
}
```

🏆 結果発表！→ 正解の選択肢名 → 最終オッズ → 自動で通常表示に戻る。この5秒のアニメーションが、結果発表の興奮を視覚的に増幅します。

---

## 📱 ゲストのスマホ：ベットモーダル

大予想が始まると、ゲストのスマホ（カジノページ）にベットモーダルが**自動で浮上**します。

```typescript
// coliseum-bet-modal.tsx
// open状態のマッチがあり、まだベットしていなければモーダル表示
const shouldShowModal = openMatch && !existingBet;
```

### モーダルのUI構成

```
┌─────────────────────────────┐
│ ケーキ入刀：どちらが先？      │
├─────────────────────────────┤
│                              │
│  🔴 新郎    🔵 新婦          │
│   1.50x      4.33x          │
│  [選択]     [選択]           │
│                              │
│  ベット額: [       ] chips   │
│  [100] [500] [1000] [2000]   │
│                              │
│  残高: 8,500 chips           │
│                              │
│  [ ベットする ]              │
└─────────────────────────────┘
```

**プリセットボタン**を用意したのは、酔っ払った状態で数字を入力するのが困難だからです。ワンタップでベットできる設計が参加率を大きく上げます。残高を超えるプリセットは自動で非活性になります。

### ベット成功後の演出

```typescript
if (result.success) {
  setBetSuccess(true);
  setTimeout(() => {
    setBetSuccess(false);
    setShowModal(false);
  }, 1500);
  onChipsUpdated(); // 親コンポーネントの残高を更新
}
```

🎰「ベット完了！」の表示を1.5秒見せてからモーダルを閉じる。即座に閉じると「本当にベットできた？」と不安になるため、フィードバックの間を取ります。

ベット済みの場合は、画面下部にステータスバーが常時表示されます。

```
[大予想] ケーキ入刀 → 新婦に1,000 chips ベット済 ✅
```

---

## 🖥️ 管理画面：新郎新婦2人で完結する運営

### イベント作成

管理画面からイベントを作成します。選択肢は2〜6個まで自由に設定可能。

```
イベント名: ケーキ入刀：どちらが先に食べさせる？
選択肢1: 新郎
選択肢2: 新婦
最低オッズ: 1.5x
odds_seed: 5000
```

「ブーケトスで誰がキャッチする？」のような3択以上のイベントも作れます。選択肢の数に応じてスクリーンのレイアウトが自動調整されます。

### ワンクリック状態遷移

| ボタン | 遷移 | 効果 |
|--------|------|------|
| 受付開始 | prepared → open | ゲストのスマホにモーダル表示 |
| 受付締切 | open → locked | モーダルが閉じ、ベット不可に |
| 結果確定 | locked → finished | 正解選択ダイアログ → 配当自動配布 |
| 中止 | any → cancelled | 全ベット返金 |

「結果確定」ボタンを押すと、正解選択ダイアログが開きます。ここで各選択肢の**事前計算されたオッズ・ベット額・人数**が表示されるため、新郎新婦は結果を確認してからワンクリックで配当を実行できます。

### キャンセル（全額返金）のDB関数

```sql
CREATE OR REPLACE FUNCTION cancel_coliseum_match(p_match_id UUID)
RETURNS JSONB AS $$
DECLARE
  v_bet RECORD;
  v_count INT := 0;
BEGIN
  -- 全ベットを返金
  FOR v_bet IN
    SELECT * FROM coliseum_bets WHERE match_id = p_match_id
  LOOP
    INSERT INTO transactions (user_id, amount, game_type, description)
    VALUES (v_bet.user_id, v_bet.amount, 'coliseum_refund', '大予想 返金');
    v_count := v_count + 1;
  END LOOP;

  UPDATE coliseum_matches SET status = 'cancelled'
  WHERE id = p_match_id;

  RETURN jsonb_build_object('success', true, 'refund_count', v_count);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

何か問題が起きたときのセーフティネット。全額返金して「なかったこと」にできます。

---

## 🧱 設計で悩んだこと、そして割り切ったこと

### 1試合1ベット vs 複数ベット

最初は「追加ベット可能」にしていました。でもテストで、少額で様子を見てからオッズが有利な方に大額を突っ込む──いわゆる**オッズ操作**が発生。

**1イベント1回限りの予想**に変更したら、ゲーム性が劇的に向上しました。「どっちを選ぶ？ いくら乗せる？」を一度に決める一発勝負の緊張感。これがゲームを面白くしています。

### 2択 vs 多選択肢

大半のイベントは2択で足ります。でも「ブーケトス」のように候補が3人以上になるケースもある。そこで選択肢を**2〜6個**まで可変にしました。スクリーンのグリッドレイアウトも選択肢数に応じて2列/3列に自動調整されます。

### 的中者が0人の場合

全員が外した場合（誰も正解の選択肢を選んでいない場合）、配当は0です。チップはプールに留まり、事実上消滅します。

結婚式ではまず起きない（2択なら50%は当たる）ですが、エッジケースとして処理を入れています。

---

## パリミュチュエル方式（変動オッズ）の予想システム実装パターン

### パターン1: odds_seedによる初期オッズ安定化
各選択肢に仮想的なシード金を加算し、ベットがゼロの状態でも安定したオッズを表示する。ベットが増えるほどseedの影響は薄まり、自然に市場原理へ移行する。

```sql
v_virtual_total := total_pool + (odds_seed * num_options);
v_virtual_winning := winning_pool + odds_seed;
v_odds := v_virtual_total / v_virtual_winning;
```

### パターン2: 最低オッズ保証
`GREATEST(raw_odds, min_odds)` で最低倍率を保証。ゲーム内通貨の場合、配当がプールを超えても問題ないケースに有効。

### パターン3: 状態遷移による演出制御
`prepared → open → locked → finished` の4状態でイベントのタイミングを制御。`locked` でベット締切〜結果発表の「間」を作り、演出効果を最大化。

### パターン4: 1試合1ベットによるオッズ操作防止
`UNIQUE(match_id, user_id)` 制約で1人1ベットを強制。少額で様子見→大額追加のオッズ操作を構造的に防止し、一発勝負の緊張感を実現。

### パターン5: Realtime購読による全端末同期
ベットのたびにオッズが変わるパリミュチュエル方式では、Realtimeによる即時反映が体験の核。スクリーン・管理画面・プレイヤースマホの3種類のクライアントが同一チャンネルを購読。
<!-- /qiita-section -->

---

## 次回予告

大予想で会場全体が熱狂する仕組みが完成しました。

次回は、祝福メッセージの仕組みを**「チップ消費」から「チップ報酬」に反転**させた話。なぜ「お金を払ってメッセージを送る」設計がうまくいかなかったのか、テスト段階で気づいた問題と改善のプロセスを解説します。

**次回：【連載第9回】チップ消費を報酬に反転させた話 ── 設計改善の記録**

---

**この記事が面白いと思ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じような「面白いことやりたいエンジニア」に届くかもしれません。

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-08-coliseum)**
