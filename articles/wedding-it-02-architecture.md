---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第2回】「1回限り」のシステムを設計する"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "typescript", "アーキテクチャ"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-02-architecture"
---

![OGP](/images/wedding-it-02-architecture-ogp.png)


# 「1回限り」のシステムを設計する

## 〜40人を60秒で認証し、披露宴と二次会で姿を変えるアーキテクチャ〜

---

[前回の記事](https://note.com/tinou/n/wedding-it-01-prologue)では、なぜ結婚式にITで貢献しようと思ったのか、3つのコンセプトと技術選定について書きました。

今回は、そのコンセプトを実現するための**アーキテクチャ設計**に踏み込みます。「1回限りのイベント」だからこそ求められる設計判断、40人が同時にアクセスする認証フロー、披露宴と二次会で切り替わるデュアルモードUI、そして72のマイグレーションを積み重ねたDB設計の全貌です。

---

## 🎯 「1回限り」が意味するもの

業務システムとは異なり、このシステムは**2026年4月5日のたった1日のために**作られます。

「1回限り」のシステム設計には、独特の判断基準があります。

- **スケーラビリティは不要**：ユーザーは最大40人。水平スケールを考える必要がない
- **メンテナンス性より確実性**：長期運用しないから、多少のコード重複は許容する
- **失敗が許されない**：「明日直します」が通用しない。当日動かなければ終わり

この「絶対に当日動く」という制約が、すべての技術選定を支配しました。

---

## 🔐 40人を60秒で認証する：匿名認証の設計

結婚式のゲストに「アカウント登録してください」とは言えません。メールアドレスの入力もパスワードの設定も、お祝いの場にはふさわしくない。

### Supabase Anonymous Auth

Supabase の匿名認証を採用しました。ゲストがやることはたった3ステップです。

1. NFCタグをスマホにかざす（URLが開く）
2. 名前と所属を入力
3. 顔写真を撮影（オプション）

```typescript
// 匿名認証 → プロフィール作成の一連のフロー
const handleSubmit = async () => {
  // 1. Supabase匿名認証（メアドもパスワードも不要）
  const { data: authData } = await supabase.auth.signInAnonymously();
  const userId = authData.user.id;

  // 2. プロフィール作成（UPSERTで冪等性を確保）
  await supabase.from("profiles").upsert({
    id: userId,
    supabase_user_id: userId,
    display_name: displayName,
    affiliation: affiliation,   // 新郎友人、新婦同僚、etc.
    club: club,                 // 部活（スタンプラリー用）
    current_chips: 1000,        // 初期チップ
    is_pin_verified: true,
  }, { onConflict: "id" });

  // 3. 顔写真があれば登録（後でAI顔認識に使用）
  if (faceBase64) {
    await supabase.storage.from("avatars").upload(`${userId}.jpg`, blob);
    await supabase.functions.invoke("register-face", {
      body: { profileId: userId, avatarPath: `${userId}.jpg` },
    });
  }
};
```

会場PINの検証はEdge Functionで行い、IPベースのレート制限（10分間に5回まで）を設けて**部外者のアクセスを防ぎつつ、ゲストには最小限の操作で参加してもらう**設計です。

### 2つのオンボーディング経路

実は、披露宴と二次会では**参加の入り口が異なります**。

- **披露宴ゲスト**：受付でNFCタグをタッチ → 名前・所属・顔写真を登録 → `/reception` へ
- **二次会参加者**：NFCチップをタッチ → チップが紐付けられ、チームに自動振り分け → `/casino` へ

二次会では物理的なNFCチップが「入場券」になります。チップをタッチすると RPC 関数 `claim_chip` でチップが紐付けられ、`assign_team_if_needed` で RED か BLUE チームに自動的に振り分けられます。

```sql
-- チーム自動振り分け：少ない方のチームに配属
CREATE FUNCTION assign_team_if_needed(p_user_id UUID)
RETURNS TEXT AS $$
DECLARE
  v_red_count INT;
  v_blue_count INT;
BEGIN
  SELECT
    COUNT(*) FILTER (WHERE team_color = 'RED'),
    COUNT(*) FILTER (WHERE team_color = 'BLUE')
  INTO v_red_count, v_blue_count
  FROM profiles
  INNER JOIN chips ON chips.owner_id = profiles.id
  WHERE team_color IS NOT NULL;

  -- 少ない方に割り当て、同数ならランダム
  IF v_red_count < v_blue_count THEN
    UPDATE profiles SET team_color = 'RED' WHERE id = p_user_id;
  ELSIF v_blue_count < v_red_count THEN
    UPDATE profiles SET team_color = 'BLUE' WHERE id = p_user_id;
  ELSE
    UPDATE profiles SET team_color =
      CASE WHEN random() < 0.5 THEN 'RED' ELSE 'BLUE' END
    WHERE id = p_user_id;
  END IF;
END;
$$;
```

このバランス調整ロジックのおかげで、チーム人数が片方に偏ることなく、公平な対戦が実現できます。

---

## 🎨 デュアルモードUI：白とゴールドの切り替え

このシステムの特徴的な設計の一つが、**披露宴と二次会でUIが完全に切り替わる**ことです。

### 3つのフェーズ

システムには3つのフェーズがあり、`system_config` テーブルの `event_status` と、ユーザーの `team_color` の2軸で制御しています。

```
EVENT PHASES:
┌─────────────────────────────────────────────────┐
│ PHASE 1: RECEPTION（披露宴）                     │
│ event_status: "active" / team_color: null        │
│ UI: 白を基調としたエレガントなデザイン             │
│ 機能: 写真アップロード、マイアルバム、クエスト       │
└─────────────────────────────────────────────────┘
                        ↓
            [管理画面: チップ配布 → team_color 設定]
                        ↓
┌─────────────────────────────────────────────────┐
│ PHASE 2: CASINO（二次会）                        │
│ event_status: "active" / team_color: RED | BLUE  │
│ UI: 黒×ゴールドのカジノデザイン                    │
│ 機能: バトル、コロシアム、ランキング、スタンプラリー  │
└─────────────────────────────────────────────────┘
                        ↓
         [管理画面: "エピローグモードON" ボタン]
                        ↓
┌─────────────────────────────────────────────────┐
│ PHASE 3: EPILOGUE（エピローグ）                   │
│ event_status: "epilogue"                         │
│ ミドルウェアが全ルートを /thank-you にリダイレクト   │
│ 最終成績、チーム戦結果、称号を表示                   │
└─────────────────────────────────────────────────┘
```

### 披露宴モード（白を基調としたエレガントなUI）

披露宴は感動の場です。ゲームの存在を意識させません。チップが裏で貯まっていることも見せません。

```typescript
// app/reception/page.tsx（サーバーコンポーネント）
// 白×ローズの落ち着いた配色。写真とクエストに集中
<div className="bg-gradient-to-b from-rose-50 via-white to-amber-50">
  <p className="text-gray-600">
    二次会が始まると、特別なモードが解放されます
  </p>
</div>
```

### 二次会モード（黒 x ゴールドのカジノUI）

二次会に切り替わった瞬間、画面が黒とゴールドに変わり、「あなたのチップは○○です！」と表示される。

```typescript
// app/casino/page.tsx（クライアントコンポーネント）
// Supabase Realtimeでチップ増減をリアルタイム反映
const channel = supabase
  .channel("chips-updates")
  .on("postgres_changes", {
    event: "INSERT",
    schema: "public",
    table: "transactions",
  }, async (payload) => {
    // チップ増減アニメーション！
    setChipAnimation(payload.new.amount);
  })
  .subscribe();
```

カジノのメニューは `system_config` テーブルの `casino_menu_buttons` で**完全に設定駆動**。管理画面からボタンの表示/非表示、ラベル、並び順をリアルタイムに変更できます。当日の進行に合わせて柔軟に対応できる設計です。

### エピローグモード：一斉リダイレクトの仕組み

イベント終了時、管理画面のボタン一つで全ユーザーを `/thank-you` にリダイレクトします。

```typescript
// middleware.ts — エピローグモード時のリダイレクト
const EPILOGUE_EXEMPT_ROUTES = [
  "/thank-you", "/admin", "/finale",
  "/ranking/display", "/slideshow", "/api",
];

if (eventStatus === "epilogue" && !isExempt) {
  return NextResponse.redirect(new URL("/thank-you", request.url));
}
```

Next.js のミドルウェア層で制御するため、**どのページにいても瞬時にエピローグ画面に遷移**します。管理画面とディスプレイ表示用のルートだけは除外し、運営が引き続き操作できるようにしています。

---

## 🗄️ 72のマイグレーションが語るDB設計

開発を始めて6週間で、マイグレーションファイルは72個に達しました。これは「最初から完璧な設計はできない」ことの証拠であり、同時に「段階的に改善し続けた」証でもあります。

### ER図（主要テーブル）

25テーブルから成るデータベースの中核部分です。

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   profiles   │     │    photos    │     │  photo_tags  │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │◄────│ user_id (FK) │     │ id (PK)      │
│ display_name │     │ id (PK)      │◄────│ photo_id(FK) │
│ affiliation  │     │ storage_path │     │ user_id (FK) │
│ club         │     │ labels       │     │ confidence   │
│ team_color   │     │ is_blocked   │     │ reward_done  │
│ current_chips│     │ last_shown   │     └──────────────┘
│ avatar_url   │     └──────────────┘
│ is_pin_verify│
└──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ transactions │     │coliseum_match│     │ coliseum_bets│
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │     │ id (PK)      │◄────│ match_id(FK) │
│ user_id (FK) │     │ title        │     │ user_id (FK) │
│ amount       │     │ options      │     │ option_id    │
│ game_type    │     │ source_ref   │     │ amount       │
│ source_ref   │     │ status       │     │ payout       │
└──────────────┘     └──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   battles    │     │    chips     │     │  blessings   │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │     │ id (PK)      │     │ id (PK)      │
│ challenger_id│     │ code         │     │ from_user(FK)│
│ defender_id  │     │ owner_id(FK) │     │ to_user (FK) │
│ bet_amount   │     │ nfc_token_id │     │ message      │
│ winner_id    │     └──────────────┘     │ chips_amount │
└──────────────┘                          └──────────────┘
```

### 設計の要点：transactions テーブルの冪等性

チップ経済圏の根幹は `transactions` テーブルです。ここで最も重要なのが `source_ref` による**冪等性の保証**です。

```sql
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id),
  amount INTEGER NOT NULL,
  game_type TEXT NOT NULL,
  description TEXT,
  source_ref TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- source_ref のユニーク制約で二重付与を防ぐ
CREATE UNIQUE INDEX idx_tx_idempotency ON transactions(source_ref);
```

`source_ref` は `'photo:abc123:tagger'` や `'stamp:uuid:sender:テニス部'` のような形式で、「どのイベントに対するチップか」を一意に特定します。ネットワークが不安定な会場で同じリクエストが2回飛んでも、`ON CONFLICT DO NOTHING` で二重付与を防げます。

### game_type で全チップ移動を分類

`transactions` テーブルの `game_type` は、チップがどの機能で発生したかを示します。

| game_type | 内容 |
|-----------|------|
| `initial` | 初期チップ付与 |
| `photo_upload` | 写真アップロード報酬 |
| `photo_tagged` | 写真に写っていた報酬 |
| `battle_win` / `battle_lose` | P2Pバトル |
| `coliseum_bet` / `coliseum_payout` | コロシアム賭け |
| `transfer_out` / `transfer_in` | チップ送付 |
| `stamp_reward` | スタンプラリー報酬 |
| `blessing` | 祝福メッセージ報酬 |
| `bar_reward` | バーカウンター報酬 |
| `admin_adjustment` | 管理者による調整 |

このフラットな設計のおかげで、**ランキング集計も履歴表示もすべて1テーブルのクエリ**で完結します。

### RLS（Row Level Security）の方針

Supabase の RLS は、セキュリティの最後の砦です。40人のゲスト全員が同じアプリを使うため、「他人のデータを書き換えられない」保証が必要です。

基本方針はシンプルです。

- **SELECT**: 全ゲストが閲覧可能（ランキングや写真は共有リソース）
- **INSERT**: 自分のデータのみ作成可能
- **UPDATE**: 自分のプロフィールのみ更新可能
- **DELETE**: 基本的に禁止（データは残す）

チップの移動や対戦結果など、複数ユーザーに影響する操作は**すべてサービスロールクライアント（Server Actions）経由**で行い、クライアントから直接テーブルを操作させない設計にしています。

---

## 🚀 Vercel + Supabase の本番構成

### ディレクトリ構成

```
casino-interaction-system/
├── app/
│   ├── c/[code]/           # NFC/QRエントリー（チップ紐付け）
│   ├── onboarding/         # 披露宴・二次会オンボーディング
│   ├── reception/          # 披露宴モード
│   │   ├── upload/         # 写真アップロード
│   │   └── my-album/       # マイアルバム
│   ├── casino/             # 二次会モード（設定駆動メニュー）
│   ├── battle/             # P2Pバトル
│   ├── coliseum/           # The Coliseum（賭けイベント）
│   ├── ranking/            # ランキング
│   ├── stamp-rally/        # スタンプラリー
│   ├── pay/[code]/         # チップ送付
│   ├── my-qr/              # QRコード表示
│   ├── scan/               # QRスキャン
│   ├── blessing/           # 祝福メッセージ
│   ├── bar/                # バーカウンター
│   ├── quest/              # Royal Trinityクエスト
│   ├── thank-you/          # エピローグ画面
│   ├── admin/              # 管理画面
│   └── ...                 # その他（gallery, profile, etc.）
├── supabase/
│   ├── migrations/         # 72 マイグレーション
│   └── functions/          # 12 Edge Functions
├── lib/
│   ├── supabase/           # クライアント設定
│   └── config/             # 所属・部活などの設定
└── middleware.ts            # 認証・エピローグ制御
```

26以上のルート、12のEdge Function、25のテーブル。個人開発としては大規模ですが、Next.js 15 の App Router のおかげでルーティングは直感的に管理できています。

### 主要な技術スタック

| 技術 | 用途 |
|------|------|
| Next.js 15 (App Router) | フロントエンド + Server Actions |
| React 19 | UIライブラリ |
| Supabase (PostgreSQL) | DB + 認証 + Realtime + Storage |
| Supabase Edge Functions | PIN検証、顔認識 API |
| Vercel | ホスティング + CDN |
| Tailwind CSS | スタイリング |
| Framer Motion | アニメーション |
| heic2any | iPhone写真(HEIC)変換 |
| html5-qrcode | QRコードスキャン |

:::message-only

## 🔧 有料セクション：設計の深掘り

ここからは、実装で特に工夫した点を掘り下げます。

### system_config テーブル：柔軟な設定管理

このシステムの柔軟性の鍵は、`system_config` テーブルです。key-value 形式で、**管理画面からリアルタイムに変更可能な設定**を格納しています。

```sql
CREATE TABLE system_config (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

| key | value の例 | 用途 |
|-----|-----------|------|
| `event_status` | `"active"` / `"epilogue"` | イベントフェーズ制御 |
| `venue_pin` | `"1234"` | 会場PIN |
| `casino_menu_buttons` | `[{ id, label, emoji, href, enabled }]` | カジノメニュー構成 |
| `chip_rewards` | `{ photo_upload: 500, blessing: 500 }` | 報酬額設定 |
| `guest_affiliations` | `[{ value, label, color }]` | 所属の選択肢 |
| `guest_clubs` | `[{ value, label, emoji }]` | 部活の選択肢 |
| `game_types` | `{ battle_win: { label, emoji } }` | ゲーム種別定義 |

JSONB型を使うことで、**スキーマ変更なしに新しい設定を追加**できます。

### Edge Function のアーキテクチャ

12のEdge Functionは大きく3カテゴリに分かれます。

**認証・セキュリティ（3）**
- `verify-pin` — 会場PIN検証 + IPレート制限
- `verify-access-token` — URL トークン検証
- `verify-admin` — 管理者パスワード検証

**顔認識 AI（6）**
- `search-face` — 顔画像で既存ユーザー検索
- `register-face` — 顔を Face Collection に登録
- `analyze-face` — 顔分析
- `retag-photo` — 写真の顔タグ付け直し
- `reanalyze-photos` — バッチ再解析
- その他デバッグ用

**プロフィール管理（1）**
- `link-profile` — 既存プロフィールと新規匿名ユーザーの紐付け

### トリガーによるチップ自動計算

`current_chips` は `transactions` テーブルへのINSERT時にトリガーで自動更新されます。

```sql
CREATE OR REPLACE FUNCTION update_current_chips()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE profiles
  SET current_chips = current_chips + NEW.amount
  WHERE id = NEW.user_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_chips
AFTER INSERT ON transactions
FOR EACH ROW EXECUTE FUNCTION update_current_chips();
```

この設計により、**アプリケーション層でチップ残高を計算する必要がありません**。トランザクションを INSERT するだけで、残高が自動的に更新されます。

### スタンプラリー：双方向の報酬ロジック

スタンプラリーは、チップを贈った際に相手の「所属」と「部活」を自動収集する仕組みです。面白いのは**送り手にも受け手にも報酬が発生する**双方向設計です。

```typescript
// 送り手が新しい所属/部活のスタンプを獲得した場合
// → 送り手に500チップ報酬
const senderRef = `stamp:${transferId}:sender:${stamp}`;
await serviceClient.from("transactions").insert({
  user_id: senderId,
  amount: STAMP_REWARD_AMOUNT,  // 500
  game_type: "stamp_reward",
  source_ref: senderRef,  // 冪等性保証
});

// 受け手も同様に新しいスタンプを獲得
const recipientRef = `stamp:${transferId}:recipient:${stamp}`;
```

`source_ref` に送り手/受け手とスタンプの種類を含めることで、同じ組み合わせでの二重報酬を防いでいます。

:::

<!-- qiita-section -->

### DB設計の実践Tips

**1. 冪等性を `source_ref` で保証する**

イベント系システムでは、ネットワーク不安定によるリトライが頻繁に発生します。`source_ref` にユニーク制約をつけ、`ON CONFLICT DO NOTHING` で二重処理を防ぐパターンは、決済システムでも使われる定石です。

```sql
-- 冪等なチップ付与
INSERT INTO transactions (user_id, amount, game_type, source_ref)
VALUES ($1, $2, $3, $4)
ON CONFLICT (source_ref) DO NOTHING;
```

**2. チーム振り分けはDB関数で制御する**

RED/BLUE のチーム分けを均等にするため、PostgreSQL の関数で人数カウント → 少ない方に配属する処理を実装しています。クライアント側でランダムに決めると偏りが出ます。

**3. RLSは「全部禁止」から始める**

Supabase の RLS は、まずすべてのテーブルで `ENABLE ROW LEVEL SECURITY` を設定し、ポリシーを一切書かない状態（全アクセス拒否）から始めます。必要なアクセスだけをポリシーで開放していくホワイトリスト方式が安全です。

**4. イベント設定は key-value テーブルに集約する**

`system_config` テーブルに JSONB 型で設定を格納すると、マイグレーション不要で新しい設定を追加できます。イベント当日の臨機応変な対応には、この柔軟性が不可欠です。

```sql
-- 設定値の取得
SELECT value FROM system_config WHERE key = 'chip_rewards';
-- → { "photo_upload": 500, "blessing": 500, "bar": 500 }
```

**5. トリガーで残高を自動更新する**

`transactions` テーブルに INSERT するだけで `profiles.current_chips` が自動更新されるトリガーを設定しています。アプリ層の計算ミスを防ぎ、**データの一貫性をDB層で保証**できます。

<!-- /qiita-section -->

---

## 📝 次回予告

アーキテクチャの全体像が見えたところで、次回は**写真アップロードとAI顔認識**の実装に踏み込みます。

iPhoneのHEIC形式をどう処理するか、AWS Rekognition の Face Collection で「この写真に誰が写っているか」をどう判定するか、そして不正画像をどう検出するか。

新婦のために作り込んだ、写真システムの裏側をお見せします。

**次回：「結婚式の写真をAI顔認識で自動タグ付けする仕組み」**

---

**この記事が参考になったら、ぜひシェアをお願いします！**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-02-architecture)**
