---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第11回】「祭りのあと」を設計する"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "nfc", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-11-epilogue"
---

![OGP](/images/wedding-it-11-epilogue-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

# 「祭りのあと」を設計する

## 🌙 「祭りのあと」の余韻を設計する

カジノの熱狂が終わり、ゲストたちが帰路につく。手元には、二次会で使った**NFCタグ**が残っています。

ここで一つ問いかけます。

> **イベントの「終わり方」を設計していますか？**

多くのイベントシステムは、イベント当日の体験設計に全力を注ぎます。でも、**翌日以降の体験**まで設計しているものは少ない。

このプロジェクトでは、発想を転換しました。

> **「ゲーム道具」を「思い出の鍵」に変える**

翌日の朝、二日酔いの頭でふとNFCタグをスマホにかざすと、そこには**昨夜の写真と自分の戦績**が表示される。この体験設計が、単なる「ゲームシステム」を「思い出の装置」へと昇華させます。

---

## 🏗 サンキューページの設計

### 全体構成

NFCタグを読み取ると表示される `/thank-you` ページは、**ゲストの参加状況に応じてテーマが切り替わります**。

- **二次会参加者（team_colorあり）**: ウォームなアンバー/ホワイト基調。称号、ランキング、チーム結果が表示される
- **披露宴のみ参加者（team_colorなし）**: エレガントなローズ/ホワイト基調。席次カードメッセージと写真ギャラリーを中心に表示

二次会に参加していないゲストに「チップランキング3位！」と表示されても意味がありません。参加状況に応じて**見せるべき情報を変える**のが、体験を壊さないための配慮です。

以下は二次会参加者向けの構成です。

```
+---------------------------------------------+
|              Hero Message                    |
|    「本日はありがとうございました！」          |
|    素敵な時間を一緒に過ごせて幸せでした        |
+---------------------------------------------+
|           席次カードのメッセージ               |
|    （第7回で設定した個別メッセージを再表示）    |
+---------------------------------------------+
|              Your Result                     |
|    [称号のemoji]                              |
|    👑 KING OF CASINO  📸 伝説のカメラマン      |
|                                              |
|    あなたは Team RED でした  (勝利チーム!)     |
|                                              |
|    最終チップ   |   順位    |  撮影枚数       |
|    54,000      |  3/40    |  12枚           |
+---------------------------------------------+
|      💌 新郎新婦からの返信                    |
|    あなたのメッセージ: 「ハネムーンどこ行く？」 |
|    返信: 「モルディブに行くよ！楽しみ！」      |
+---------------------------------------------+
|           チーム対抗戦結果                     |
|    Team RED     vs     Team BLUE             |
|    12,500 avg         9,800 avg              |
+---------------------------------------------+
|              Photo Gallery                   |
|    [写真] [写真] [写真] [写真]                |
|    [写真] [写真] [写真] [写真]                |
+---------------------------------------------+
|    NFCチップを大切に保管してください            |
|    いつでもこのページで思い出を振り返れます      |
+---------------------------------------------+
```

### なぜこの順序なのか

1. **Hero Message** -- まず感謝を伝える。これが最も大切
2. **席次カードのメッセージ** -- 披露宴で受け取った個別メッセージを再表示。原点に立ち返る
3. **称号** -- 数字より先に「あなたは何者だったか」を見せる。感情的な体験
4. **統計** -- 称号で感情を動かした後に、具体的な数字で裏付ける
5. **新郎新婦からの返信** -- 自分のメッセージへの返信。「読んでくれたんだ」という感動
6. **チーム結果** -- 個人の結果を見た後に、チームの結果。帰属意識
7. **フォトギャラリー** -- 最後に写真。思い出に浸る時間を提供する

**数字より先に感情を動かす**。これがUXの鍵です。

---

## 🏆 称号システム

### 「数字」だけでは味気ない

「最終資産: 54,000チップ」と表示されても、それが凄いのかどうか分かりません。40人中の相対的な位置がなければ、数字は意味を持たない。

そこで**称号（タイトル）**を付与して、感情的な満足感を提供します。

### 表彰式の構成

表彰式は3つのパートで構成しています。

**1. 特別賞（5部門）── 景品付き**

| 称号名 | 判定基準 | 景品 |
|--------|----------|------|
| 🍺 バーの主 | バー来店回数1位 | 日本酒 |
| 📸 伝説のカメラマン | 写真投稿数1位 | スマホ用拡大レンズ |
| ✨ みんなのアイドル | 写真タグ付け回数1位 | 高級フェイスマスクセット |
| 🗺️ スタンプマスター | スタンプ収集数1位 | 名湯巡り入浴剤セット |
| 💸 名誉スポンサー | チップ損失額1位 | 豚の貯金箱 |

**2. 個人ランキング トップ3**

| 順位 | 称号名 | 景品 |
|------|--------|------|
| 👑 1位 | KING OF CASINO | バルミューダ 電気ケトル |
| 🥈 2位 | 個人ランキング 2位 | お菓子の詰め合わせセット |
| 🥉 3位 | 個人ランキング 3位 | 新婦の職場製品詰め合わせ |

**3. チーム賞**

チーム平均チップで比較し、勝利チームを発表。

### 設計のポイント

**1. 「負け」も称号にする**

**名誉スポンサー**は、最も多くのチップを失った人に贈られます。景品が**豚の貯金箱**なのが最大の笑いポイント。普通なら残念な結果ですが、「あなたがチップをばら撒いたから経済が回った」と表現し、景品で笑いに変える。

**2. 景品を称号の世界観に合わせる**

「バーの主」に日本酒、「伝説のカメラマン」にスマホレンズ──称号と景品がリンクしていると、受け取った瞬間に笑いと納得が生まれます。景品選びも体験設計の一部です。

**3. 管理画面から発表順を制御**

```typescript
// 表彰式の定義（発表順に配列で管理）
const SPECIAL_AWARD_DEFS = [
  { id: "special_bar", title: "バーの主", sortKey: "bar_visit_count", ... },
  { id: "special_camera", title: "伝説のカメラマン", sortKey: "upload_count", ... },
  // ...
];
```

特別賞 → 個人ランキング3位→2位→1位 → チーム賞の順で管理画面のボタンを押すだけで表彰が進行します。`user_stats_view` から自動で受賞者を計算するため、新郎新婦は結果を事前に知る必要もありません。

### サンキューページでの称号（補足）

表彰式とは別に、サンキューページではより細かい称号（`percent_rank()` ベースで上位20%に該当する全ゲストに付与）が表示されます。表彰式は「各部門1位」に景品を渡す場、サンキューページは「全員が何かしらの称号をもらえる」場──この役割分担で、40人全員を肯定する設計にしています。

### 統計分析View

称号判定のベースとなるデータは、PostgreSQLのViewで事前計算しています。

```sql
CREATE OR REPLACE VIEW user_stats_view AS
WITH stats AS (
  SELECT
    p.id AS user_id,
    p.display_name,
    p.affiliation,
    p.current_chips,
    (SELECT COUNT(*) FROM photos WHERE uploader_id = p.id) AS upload_count,
    (SELECT COUNT(*) FROM photo_tags WHERE user_id = p.id) AS tagged_count,
    (SELECT COUNT(*) FROM transactions WHERE user_id = p.id
      AND game_type = 'bar_reward') AS bar_visit_count,
    (SELECT COUNT(*) FROM blessings WHERE sender_id = p.id) AS blessing_count,
    (SELECT COALESCE(SUM(amount), 0) FROM coliseum_bets
      WHERE user_id = p.id) AS total_bet_amount,
    (SELECT COUNT(DISTINCT receiver_id) FROM transactions
      WHERE user_id = p.id AND game_type = 'transfer_out') AS transfer_count
  FROM profiles p
)
SELECT
  *,
  percent_rank() OVER (ORDER BY current_chips DESC) AS rank_chips,
  percent_rank() OVER (ORDER BY upload_count DESC) AS rank_upload,
  percent_rank() OVER (ORDER BY tagged_count DESC) AS rank_tagged,
  percent_rank() OVER (ORDER BY bar_visit_count DESC) AS rank_bar_visit,
  percent_rank() OVER (ORDER BY blessing_count DESC) AS rank_blessing,
  percent_rank() OVER (ORDER BY total_bet_amount DESC) AS rank_total_bet,
  percent_rank() OVER (ORDER BY transfer_count DESC) AS rank_transfer
FROM stats;
```

`percent_rank()` は、0.0（最上位）から1.0（最下位）の値を返すPostgreSQLのウィンドウ関数です。これを使うことで、**参加者数に依存しない**相対的なランク付けが可能になります。

40人でも100人でも、「上位20%」は常に `rank <= 0.2` で判定できる。

ランキングの軸が7つあるため、**どんな遊び方をしたゲストでも何かしらの上位に入る可能性がある**のがポイントです。

### 称号判定ロジック（Server Actions）

```typescript
// app/admin/ceremony/actions.ts（抜粋）

/** 特別賞 定義（発表順） */
const SPECIAL_AWARD_DEFS = [
  {
    id: "special_bar",
    title: "バーの主",
    emoji: "🍺",
    description: "誰よりもバーに通った人",
    prize: "景品：日本酒",
    sortKey: "bar_visit_count",
    formatValue: (v: number) => `${v} 回来店`,
  },
  {
    id: "special_camera",
    title: "伝説のカメラマン",
    emoji: "📸",
    description: "最も多く写真を撮った人",
    prize: "景品：スマホ用拡大レンズ",
    sortKey: "upload_count",
    formatValue: (v: number) => `${v} 枚投稿`,
  },
  // ... みんなのアイドル、スタンプマスター、名誉スポンサー
];

/** 個人賞 1位 */
const INDIVIDUAL_1ST = {
  title: "KING OF CASINO",
  emoji: "👑",
  prize: "景品：バルミューダ 電気ケトル",
};
```

表彰式の受賞者は `user_stats_view` から**自動計算**されます。新郎新婦は管理画面で「次の賞を発表」ボタンを押すだけ。受賞者の名前、記録、景品がスクリーンに表示されます。

---

## 📷 フォトギャラリーと一括ダウンロード

### 全員の視点を共有する

サンキューページの下部には、当日の写真がグリッドレイアウトで表示されます。

```typescript
// Server Action: 写真一覧取得
export async function getPhotos() {
  const serviceClient = createServiceClient();

  const { data: photos } = await serviceClient
    .from("photos")
    .select(`
      id,
      storage_path,
      created_at,
      uploader:profiles!photos_uploader_id_fkey(display_name)
    `)
    .eq("is_valid_for_reward", true)
    .order("created_at", { ascending: false })
    .limit(100);

  return {
    success: true,
    photos: photos?.map((p: any) => ({
      id: p.id,
      storage_path: p.storage_path,
      uploader_name: p.uploader?.display_name || "Unknown",
      created_at: p.created_at,
    })),
  };
}
```

ポイントは `is_valid_for_reward = true` のフィルタです。不正画像やスクリーンショットとして検出された写真は除外され、**本物の思い出だけ**が表示されます。

### なぜ「全員の写真」なのか

自分が撮った写真だけを表示する選択肢もありました。でも、**全員の写真を共有する**ことを選びました。

理由は3つ。

1. **「あ、こんな瞬間あったんだ！」という発見**がある。自分が見ていなかった角度の写真が見つかる
2. **参加者全員の視点を共有**できる。40人の40通りの結婚式体験が集まる
3. **写真が少ない人でも楽しめる**。撮影が苦手な人も、他の人の写真で思い出を追体験できる

---

## 📱 NFCタグを「思い出のトークン」にする

### 物理から始まり、物理に還る

このシステムの入口と出口は、どちらも**NFCタグ**です。

- **入口**: NFCタグを読み取って認証、ゲームに参加
- **出口**: 同じNFCタグを読み取って、思い出ページにアクセス

イベント中は「ゲーム道具」だったNFCタグが、イベント後は「思い出の鍵」になる。物理的なモノが意味を変えて手元に残り続ける。

何ヶ月後、何年後でもNFCタグを読み取れば思い出が蘇ります。デジタルデータは消えやすいけれど、**物理トークンは「捨てられないもの」**になる。

### 新郎新婦からの返信が表示される

第9回で紹介した返信機能がここで効いてきます。ゲストがサンキューページを開くと、自分が送ったメッセージと**新郎新婦からの返信**が一緒に表示されます。

また、自分が「いいね」したメッセージに返信がついている場合もここに表示されます。

```typescript
// 返信付きメッセージの取得
const repliedBlessings = await supabase
  .from("blessings")
  .select(`*, blessing_replies(*)`)
  .or(`sender_id.eq.${userId},id.in.(${likedBlessingIds})`)
  .not("blessing_replies", "is", null);
```

**イベント後にゲストがページを開いて、新郎新婦からの返信を見つける。** 席次カードの個別メッセージ（第7回）と同じく、「自分だけへの言葉」が心に残ります。

### 最後のメッセージ

二次会の終わりに、新郎新婦からゲストにこう伝えます。

> **「お手元のNFCチップは、捨てずに記念にお持ち帰りください！」**
>
> **「家に帰ってからスマホで読み取ると、今日撮った全員の写真が見れます！」**
>
> **「明日の朝、二日酔いの頭で眺めてニヤニヤしてください！」**

この一言が大切です。**「捨てないでね」と伝える設計**まで含めて、エピローグの設計です。

---

## 💡 なぜこれが最高のエピローグなのか

3つの理由があります。

### 1. 物理トークンからデジタル体験への橋渡し

NFCタグという物理的なモノが、デジタルな思い出への入口になる。URLをブックマークするより、**手元にモノがある方が思い出しやすい**。引き出しの奥でNFCタグを見つけたとき、自然とスマホをかざしたくなる。

### 2. 参加者全員の視点が集まる場所

結婚式の写真は、普通なら個人のスマホに散らばります。LINEグループで共有しても、いつの間にか流れてしまう。

このサンキューページは、**40人の視点が一箇所に集まる永続的な場所**です。

### 3. ゲームの「結果」が物語になる

単なる写真アルバムではなく、「自分の戦績」が残る。「KING OF CASINO」の称号はスクリーンショットしてSNSに上げたくなるデザインにしています。

**イベントが終わった後も、SNSで話題にしてもらえる**。これがエピローグの力です。

<!-- qiita-section -->

## NFC実装Tips

結婚式でNFCタグを活用する際の実装ポイントをまとめます。

### NFCタグの選定

NTAG215またはNTAG216を推奨します。URLを書き込むだけなら容量は十分です。カジノチップ型のケースに埋め込むことで、見た目の体験も向上します。

### URL設計

```
https://example.com/c/{CHIP_CODE}
```

`CHIP_CODE` にはUUIDを含めて推測困難にします。

```sql
-- 40枚分のチップを生成
INSERT INTO chips (code)
SELECT 'CHIP-' || LPAD(n::TEXT, 3, '0') || '-' || gen_random_uuid()
FROM generate_series(1, 40) AS n
ON CONFLICT (code) DO NOTHING;
```

### イベント後のルーティング

同じNFCタグを、イベントのフェーズによって異なるページにルーティングできます。

- イベント前: `/onboarding/chip`（登録ページ）
- イベント中: `/casino`（ゲームページ）
- イベント後: `/thank-you`（サンキューページ）

サーバー側でイベントフェーズを管理し、同一URLでもリダイレクト先を切り替える設計が有効です。

### セキュリティ

- 1ユーザー1チップの制約を設ける（同一ユーザーが複数チップを紐付けることを防止）
- チップ紐付け処理はレースコンディション対策を実施（`SELECT FOR UPDATE` を使用）
- UUIDを含むコードで推測を防止

<!-- /qiita-section -->

---

## 🎬 次回予告

ここまで11回にわたって、結婚式カジノシステムの設計と実装を解説してきました。

そして、いよいよ**本番を迎えます**。

次回は**2026年4月5日の結婚式当日レポート**。2時間半の運用で何が起きたのか、リアルな記録をお届けします。

計画通りに動くのか。トラブルは起きるのか。ゲストは本当に楽しんでくれるのか。

**すべての答えが出る日が、もうすぐやってきます。**

---

**この記事が面白いと思ったら、ぜひシェアをお願いします！**

あなたのシェアが、同じような「面白いことやりたいエンジニア」に届くかもしれません。

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-11-epilogue)**
