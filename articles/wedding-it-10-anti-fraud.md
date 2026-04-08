---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第10回】ゲームを壊さないための多層防御設計"
emoji: "💒"
type: "tech"
topics: ["supabase", "postgresql", "security", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-10-anti-fraud"
---

![OGP](/images/wedding-it-10-anti-fraud-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

# ゲームを壊さないための多層防御設計

## 〜クライマックスルール、冪等性、レート制限、不正画像検出〜

---

> **連載第10回。今回は、ゲームバランスを守るための「不正対策」を解説します。**
>
> 結婚式の二次会でゲームをやる以上、「ズルをしたい」人は必ず現れます。友人同士でチップを集約する、PINコードを総当たりする、ネットで拾った写真をアップロードする。これらを**ゲスト体験を損なわずに**防ぐための多層防御の設計記録です。

---

## 🎯 不正対策の設計原則

結婚式の不正対策は、一般的なWebサービスとは異なる制約があります。

1. **ゲストを「犯罪者扱い」しない** -- 過剰なセキュリティはお祝いの場にそぐわない
2. **ゲームの公平性は守る** -- 不正で1位を取られると他のゲストが白ける
3. **事前に防ぐ、事後に検出する** -- リアルタイムイベントだから、起きてからでは遅い

この3つのバランスを取りながら、**多層的な防御**を設計しました。

---

## ⏰ クライマックスルール：ラスト30分の結託防止

### なぜ「ラスト30分」を制限するのか

二次会の終盤は、ランキング争いが最も白熱する時間帯です。しかし、ここで**「仲間内でチップを集約して1位を取る」**という不正が起こりえます。

例えば：
- 新郎友人グループが結託
- 全員のチップを1人に贈る
- その1人がぶっちぎりで1位

これでは、他のグループから見て不公平ですし、何より**盛り上がりが冷めます**。

### 実装：同所属間チップ送付の禁止

```typescript
// クライマックスルール設定
const EVENT_END_TIME = new Date("2026-04-06T20:00:00+09:00");
const CLIMAX_MINUTES = 30;

const now = new Date();
const minutesLeft = (EVENT_END_TIME.getTime() - now.getTime()) / 1000 / 60;

if (minutesLeft <= CLIMAX_MINUTES && minutesLeft > 0) {
  const { data: recipientProfile } = await supabase
    .from("profiles")
    .select("affiliation")
    .eq("id", recipientId)
    .single();

  if (
    senderProfile.affiliation &&
    recipientProfile?.affiliation &&
    senderProfile.affiliation === recipientProfile.affiliation
  ) {
    return {
      success: false,
      error: "climax_rule_violation",
    };
  }
}
```

**設計判断のポイント：**

- **全面禁止ではなく、同所属間のみ禁止** -- 異なるグループ間の送付は許可することで、ゲームの自由度を維持
- **ラスト30分だけ発動** -- 序盤・中盤の自由な送付はゲームの楽しさの一部
- **affiliation（所属）ベースの判定** -- 「新郎友人」「新婦友人」「会社関係」などの属性で判定

### ゲーム演出としてのルール発動

クライマックスルールは**技術的に防ぐだけでなく、ゲーム演出としても機能**させています。

新郎新婦がスクリーンにルールスライドを表示し、こう伝えます。

> **「ここからラスト30分！クライマックスルールが発動します！」**
> **「同じチーム内でのチップ送付はできなくなりました！」**
> **「他チームからチップを奪うか、自力で稼ぐしかありません！」**

制限を「ルール発動」として伝えることで、むしろ盛り上がりを生む仕掛けです。

---

## 🔑 冪等性の保証：二重付与を防ぐ

### ネットワーク不安定環境の現実

結婚式会場のWiFi環境は、40人が同時接続すると不安定になります。写真アップロードが成功したがレスポンスがタイムアウトし、ユーザーがリトライ -- すると同じ写真に対してチップが2回付与される可能性があります。

### 解決策：source_ref によるユニーク制約

```sql
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  amount INTEGER NOT NULL,
  game_type TEXT NOT NULL,
  source_ref TEXT NOT NULL,  -- 冪等性キー
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_tx_idempotency ON transactions(source_ref);
```

`source_ref` は「このトランザクションの発生源」を一意に識別する文字列です。

```sql
-- ON CONFLICT で二重挿入を防止
INSERT INTO transactions (user_id, amount, game_type, source_ref)
VALUES (user_id, 500, 'photo_reward', 'photo:' || photo_id)
ON CONFLICT (source_ref) DO NOTHING;
```

**`source_ref` の命名規則：**

| ゲーム種別 | source_ref 形式 | 例 |
|-----------|----------------|-----|
| 写真報酬 | `photo:{photo_id}` | `photo:abc123` |
| 大予想配当 | `coliseum_payout:{match_id}:{user_id}` | `coliseum_payout:m1:u1` |
| 祝福報酬 | `blessing:{blessing_id}` | `blessing:xyz789` |
| バー端末 | `bar:{timestamp}:{user_id}` | `bar:1712345:u1` |

`ON CONFLICT ... DO NOTHING` により、同じ `source_ref` のINSERTは2回目以降すべて無視されます。アプリケーション側でリトライ検出をする必要がなく、**DBレベルで冪等性が保証**されます。

<!-- qiita-section -->

### 冪等性設計パターンの比較

冪等性を保証するアプローチはいくつかあります。今回採用した「DBユニーク制約」方式と他のパターンを比較します。

**パターン1：クライアント生成の冪等キー**

```
Client → POST /api/bet { idempotency_key: "uuid-v4" }
Server → INSERT ... ON CONFLICT(idempotency_key) DO NOTHING
```

クライアントがリクエストごとにUUIDを生成。同じキーでの再送は無視される。
利点：汎用性が高い。欠点：クライアント実装が必要。

**パターン2：サーバー生成のソースリファレンス（今回の方式）**

```
Server → source_ref = "photo:" + photo_id
Server → INSERT ... ON CONFLICT(source_ref) DO NOTHING
```

サーバーがビジネスロジックに基づいてキーを生成。同じ操作に対して常に同じキーが生成される。
利点：クライアントの協力不要、ビジネスルールと紐付く。欠点：source_ref設計が必要。

**パターン3：UPSERT方式**

```
INSERT INTO ... ON CONFLICT DO UPDATE SET updated_at = NOW()
```

利点：最後の状態が常に反映される。欠点：「更新」が意図しない副作用を持つ場合がある。

結婚式システムでは「同じ写真へのチップ付与は1回だけ」というビジネスルールが明確なため、パターン2が最適でした。

<!-- /qiita-section -->

---

## 🚦 レート制限：PIN認証への攻撃対策

### 会場PINの総当たり防止

ゲストは会場到着時にPINコードを入力して認証します。4桁の数字PINを総当たりで試されることを防ぎます。

```sql
CREATE TABLE venue_pin_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ip_address INET NOT NULL,
  client_id TEXT NOT NULL,
  url_token TEXT NOT NULL,
  success BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 10分間で5回まで
CREATE OR REPLACE FUNCTION check_pin_rate_limit(
  p_ip_address INET,
  p_client_id TEXT,
  p_url_token TEXT
) RETURNS BOOLEAN AS $$
DECLARE
  attempt_count INTEGER;
BEGIN
  SELECT COUNT(*) INTO attempt_count
  FROM venue_pin_logs
  WHERE ip_address = p_ip_address
    AND client_id = p_client_id
    AND created_at > NOW() - INTERVAL '10 minutes';

  RETURN attempt_count < 5;
END;
$$ LANGUAGE plpgsql;
```

**なぜDBレベルで実装したのか？**

Vercelのサーバーレス環境では、メモリ上のカウンターは関数の実行ごとにリセットされます。Redisを導入するほどの規模ではないため、PostgreSQLのテーブルで試行回数を管理しています。

**3つの識別子を組み合わせた理由：**

- `ip_address` -- 同一ネットワーク（会場WiFi）では全員同じIPになる場合がある
- `client_id` -- ブラウザのlocalStorageに保存するUUID。端末を識別
- `url_token` -- 招待URLに含まれるトークン。ゲストを識別

会場WiFiでは全員が同じIPアドレスになるため、IPだけでは不十分です。`client_id` と `url_token` を組み合わせることで、端末レベルでのレート制限を実現しています。

---

## 📸 不正画像の検出：Rekognition DetectModerationLabels

写真アップロードでチップを獲得できる仕組みがあるため、不適切な画像のアップロードを検出する必要があります。

AWS Rekognition の **DetectModerationLabels API**（コンテンツモデレーション専用API）を使って、アップロードされた画像を自動判定します。

```typescript
// Supabase Edge Function: analyze-face/index.ts
const moderationResult = await fetch(
  `https://rekognition.${region}.amazonaws.com/`,
  {
    method: "POST",
    headers: {
      "X-Amz-Target": "RekognitionService.DetectModerationLabels",
      "Content-Type": "application/x-amz-json-1.1",
    },
    body: JSON.stringify({
      Image: { Bytes: imageBase64 },
      MinConfidence: 70,
    }),
  }
);
```

検出結果は写真レコードにJSONBとして保存されます。

```sql
-- 写真テーブルのモデレーション関連カラム
ALTER TABLE photos ADD COLUMN moderation_labels JSONB DEFAULT '[]';
ALTER TABLE photos ADD COLUMN block_reason TEXT;
ALTER TABLE photos ADD COLUMN is_blocked BOOLEAN DEFAULT false;
```

**設計判断：「削除」ではなく「ブロックフラグ」**

不正と判定された画像を即座に削除するのではなく、`is_blocked: true` としてフラグを立てるだけにしています。理由は2つ：

1. **誤検出のリスク** -- AIの判定は100%ではない。管理画面から手動で解除できる必要がある
2. **ゲスト体験の保護** -- アップロードした写真が無言で消えると不信感を与える

ブロックされた写真はギャラリーに表示されず、チップ報酬も付与されません。しかしデータは残っているので、管理画面から確認してブロックを解除できます。`moderation_labels` にAIの判定理由がJSON形式で記録されるため、「なぜブロックされたか」が一目瞭然です。

---

## 🔐 RLS（Row Level Security）の設計原則

Supabase の RLS は、本システムのセキュリティの根幹です。

**基本方針：**

1. **全テーブルでRLSを有効化** -- 例外なし
2. **読み取りは広く、書き込みは狭く** -- ランキングは全員見えるが、チップの操作は本人のみ
3. **重要な操作はRPC（SECURITY DEFINER）経由** -- ベット処理、チップ送付など、複数テーブルにまたがる操作はDBファンクションに閉じ込める

RPC関数を `SECURITY DEFINER` で実行することで、RLSをバイパスしつつ関数内部でバリデーションを行います。アプリケーション層では「この関数を呼ぶ」だけでよく、セキュリティロジックがDB内に集約されるため、フロントエンドとサーバーの両方で同じチェックを実装する必要がありません。

---

## 🎤 ルール告知との連携

不正対策は技術だけでなく、**会場でのルール告知**と組み合わせることで効果を発揮します。新郎新婦がスクリーンにルールスライドを投影したり、口頭でアナウンスしたりします。

| 対策 | 告知例 |
|------|--------|
| クライマックスルール | 「ラスト30分！同チーム内のチップ送付が禁止されました！自力で稼ぎましょう！」 |
| 写真報酬 | 「写真を撮るとチップがもらえます！ただし、スクリーンショットはAIが見破ります！」 |
| 大予想締切 | 「締め切りました！結果をお待ちください！」 |

技術的な制限を「ゲームのルール」として自然に伝えることで、不正を抑止しつつ盛り上がりも演出できます。ルール説明スライドは事前に画像として用意し、管理画面からスクリーンに表示する仕組みにしました。

---

## 📊 多層防御のまとめ

| レイヤー | 対策 | 防ぐもの |
|---------|------|---------|
| ゲームルール | クライマックスルール | チップ集約による不正1位 |
| DB制約 | source_ref ユニーク制約 | 二重付与・二重ベット |
| レート制限 | PIN認証の試行回数制限 | 総当たり攻撃 |
| AI検出 | Rekognition DetectLabels | 不正画像のチップ詐取 |
| RLS | 行レベルセキュリティ | 不正なデータ操作 |
| RPC | SECURITY DEFINER関数 | トランザクション不整合 |

どの対策も単独では完璧ではありません。しかし、**複数のレイヤーを組み合わせることで、実用上十分な防御**を実現しています。

重要なのは、これらの対策が**ゲスト体験を損なわない**こと。結婚式の二次会で「あなたのリクエストはブロックされました」と表示されたら興ざめです。クライマックスルールを「ゲーム演出」として伝え、不正画像を「静かに報酬対象外」にする。技術と運用の両面から、お祝いの場にふさわしい不正対策を実現しました。

---

## 🔜 次回予告

ゲームの熱狂が終わった後、ゲストの手元にはNFCタグが残ります。

翌日、二日酔いの頭でふとNFCタグをスマホにかざすと、そこには昨夜の写真と自分の戦績が蘇る。「ゲーム道具」を「思い出の鍵」に変える -- **「祭りのあと」の余韻を設計する話**です。

**次回、第11回「『祭りのあと』を設計する」をお楽しみに。**

---

**この記事が面白いと思ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じような「面白いことやりたいエンジニア」に届くかもしれません。

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-10-anti-fraud)**
