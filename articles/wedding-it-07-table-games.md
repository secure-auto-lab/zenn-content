---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第7回】席次カードにNFCを仕込んだら、受付が感動体験になった"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "nfc", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-07-table-games"
---

![OGP](/images/wedding-it-07-table-games-ogp.png)


> [【第1回】から読む](https://blog.secure-auto-lab.com/articles/wedding-it-01-prologue)

# 席次カードにNFCを仕込んだら、受付が「感動体験」になった

前回は物理NFCチップ（`/c/[code]`）で1vs1対戦とバトルロイヤルを実装しました。

今回は少し時間を巻き戻して、**二次会の前──披露宴の話**をします。

二次会カジノのシステムを作る中で、ずっと引っかかっていた問題がありました。

**「ゲストをどうやってシステムに登録するか」**

40人のゲストに、1人ずつアプリのURLを送って、名前を入力してもらって……？ 結婚式の当日にそんな手間をかけさせるのは論外です。

そこで目をつけたのが、披露宴で**必ず全員に渡すもの**──**席次カード**でした。

---

## 😰 「受付でゲスト全員をシステムに登録する」という難問

### 結婚式のIT化で最も難しいのは「登録」

技術的に一番むずかしいのは、リアルタイム同期でもAI顔認証でもありません。**「ゲスト全員にシステムを使ってもらう最初の1歩」**です。

普通のWebサービスなら「アプリをDLしてアカウント作って」と言えます。でも結婚式の受付で、着飾ったゲストにスマホの操作を求めるのは野暮です。手にはご祝儀袋、周りには久しぶりに会う友人。「まずこのURLを開いて、名前を入力して……」なんて、誰もやりたくありません。

しかし、システムに登録してもらわなければ、写真投稿も、チップも、ランキングも、何も始まらない。

### 「既にある導線」に乗せる

発想を転換しました。**ゲストが自然にやる行動の中に、登録を埋め込めばいい。**

披露宴の受付では、全員が必ず**席次カードを受け取ります**。名前が書かれた、自分の席を案内するカード。

**この席次カードにNFCタグを仕込んだら？**

受付係が「おめでとうございます、こちらが席次カードです。スマホにかざしてみてください」と渡す。ゲストがかざした瞬間、新郎新婦からの個別メッセージが表示される──**感動の瞬間がそのままシステム登録の入口になる**。

---

## 📖 席次カードが開くまでの物語

### ゲストの体験

```
1. 受付で席次カードを受け取る
       ↓
2. スマホにかざす → /m/[token]
       ↓
3. 自分の名前と、新郎新婦からの個別メッセージが表示される ✨
       ↓
4. 「写真を投稿する」ボタンをタップ
       ↓
5. 顔写真を撮影 → 顔認証で本人確認 or 新規登録
       ↓
6. プロフィール完成 → 披露宴の写真投稿・閲覧が可能に
       ↓
（そして二次会では、NFCチップをタッチするだけでカジノに入れる）
```

ゲストにとっては「席次カードをかざしたら素敵なメッセージが出てきた」という**感動体験**。でもその裏で、認証・プロフィール作成・顔登録が全て完了している。

**テクノロジーの存在を意識させない。** これが結婚式というハレの場にふさわしいUXです。

---

## 💭 なぜ「個別メッセージ」が設計の鍵なのか

NFC席次カードの最大の工夫は、タッチした瞬間に表示される**新郎新婦からの個別メッセージ**です。

### 「登録してください」ではなく「あなたへのメッセージがあります」

一般的なイベント受付アプリの画面を想像してください。

```
❌ よくあるパターン
┌─────────────────────────┐
│  イベント受付システム      │
│                         │
│  名前を入力してください    │
│  [          ]           │
│  [登録する]              │
└─────────────────────────┘
```

これでは「手続き」です。ゲストは面倒に感じる。

席次カードNFCでは、最初に表示されるのがこれです。

```
⭕ 席次カードNFCのパターン
┌─────────────────────────┐
│     Jun & Kanami        │
│      Wedding            │
│                         │
│    田中 太郎 様          │
│                         │
│  ┌───────────────────┐  │
│  │ 太郎くん、今日は来て │  │
│  │ くれて本当にありが  │  │
│  │ とう！大学時代から  │  │
│  │ ずっと支えてくれた  │  │
│  │ 太郎くんに、一番に  │  │
│  │ 伝えたかった。      │  │
│  │ 最高の1日にしよう！  │  │
│  └───────────────────┘  │
│                         │
│  [📸 写真を投稿する]     │
└─────────────────────────┘
```

**「あなたのために用意したメッセージがある」** という体験が先に来る。感動した流れで「写真を投稿する」ボタンを押す。そこから自然にプロフィール登録が始まる。

**システム登録が「手続き」ではなく「体験の続き」になる。** これが個別メッセージを最初に表示する理由です。

---

## 🗄️ データベース設計

### seat_cards テーブル

```sql
CREATE TABLE seat_cards (
  token TEXT PRIMARY KEY,                    -- NFCタグに書き込むトークン（UUID）
  profile_id UUID NOT NULL
    REFERENCES profiles(id) ON DELETE CASCADE,
  guest_name TEXT NOT NULL,                  -- 席次カードに印字される名前
  message TEXT NOT NULL DEFAULT '',          -- 新郎新婦からの個別メッセージ
  scanned_at TIMESTAMPTZ,                    -- 初回スキャン日時
  created_at TIMESTAMPTZ DEFAULT now()
);
```

設計のポイント：

- **`token`**: UUIDを使用。連番禁止──推測されると他人のメッセージが見えてしまう
- **`profile_id`**: 事前にプロフィールを作成し、席次カードと紐付けておく。ゲストがスキャンしたとき、既に「あなたは田中太郎さんです」とわかっている状態
- **`message`**: 新郎新婦が管理画面から1人ずつ入力する個別メッセージ
- **`scanned_at`**: 初回スキャンを記録。管理画面で「誰がまだスキャンしていないか」を把握できる

### RLS設計──誰でも読めるが、書けるのは管理者だけ

```sql
-- トークンを知っていれば誰でも閲覧可能（NFCスキャン時の表示に必要）
CREATE POLICY "Anyone can read by token"
  ON seat_cards FOR SELECT
  USING (true);

-- 作成・更新・削除は管理者のみ
-- （Service Role経由でのみ操作）
```

「トークンを知っている = 物理カードを手にしている」なので、閲覧を全開放しても問題ありません。

---

## 🔧 実装：3つのフェーズ

### Phase 1：個別メッセージの表示（`/m/[token]`）

ゲストがNFCをタッチすると、このページが開きます。

```typescript
// app/m/[token]/page.tsx（サーバーコンポーネント）
export default async function SeatCardPage({
  params,
}: {
  params: { token: string };
}) {
  const supabase = await createClient();

  const { data: seatCard } = await supabase
    .from("seat_cards")
    .select("token, guest_name, message, scanned_at, profile_id")
    .eq("token", params.token)
    .single();

  if (!seatCard) {
    notFound(); // 無効なトークン → 404
  }

  return <SeatCardClient seatCard={seatCard} />;
}
```

クライアント側では、framer-motionのアニメーションで**金とベージュ基調のエレガントなUI**を表示します。

```typescript
// app/m/[token]/seat-card-client.tsx（抜粋）
<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  className="bg-gradient-to-b from-[#f9f5ef] to-[#f0e8da]"
>
  {/* ゲスト名 */}
  <h2 className="text-2xl font-serif text-[#5a5550]">
    {seatCard.guest_name}
    <span className="text-lg ml-1">様</span>
  </h2>

  {/* 個別メッセージ */}
  {seatCard.message && (
    <div className="border border-[#d4c5a9] rounded-lg p-4">
      <p className="text-[#5a5550] leading-relaxed whitespace-pre-wrap">
        {seatCard.message}
      </p>
    </div>
  )}

  {/* アクションボタン */}
  <button onClick={handleProceed}>
    📸 写真を投稿する
  </button>
</motion.div>
```

`whitespace-pre-wrap` で改行を保持しているのがポイントです。新郎新婦が管理画面で入力した改行がそのまま表示されます。

### Phase 2：認証とセッション作成

「写真を投稿する」ボタンを押すと、Server Actionで認証処理が走ります。

```typescript
// app/m/[token]/actions.ts
"use server";

export async function authenticateGuest(token: string) {
  const supabase = await createClient();

  // 1. 席次カードからプロフィールIDを取得
  const { data: seatCard } = await supabase
    .from("seat_cards")
    .select("profile_id")
    .eq("token", token)
    .single();

  // 2. 既存セッションがあるか確認
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    // 3. なければ匿名認証でセッション作成
    await supabase.auth.signInAnonymously();
  }

  // 4. プロフィールに supabase_user_id を紐付け
  await supabase
    .from("profiles")
    .update({ supabase_user_id: user.id })
    .eq("id", seatCard.profile_id);

  // 5. 初回スキャンを記録
  if (!seatCard.scanned_at) {
    await supabase
      .from("seat_cards")
      .update({ scanned_at: new Date().toISOString() })
      .eq("token", token);
  }

  return { success: true };
}
```

ここで重要なのは **Supabase Anonymous Auth** を使っている点です。ゲストにメールアドレスやパスワードの入力は一切求めません。匿名セッションが自動で作られ、ブラウザを閉じても維持されます。

### Phase 3：顔認証 → プロフィール完成

認証後、オンボーディングフォームに遷移します。

```
撮影画面
  ↓ 顔写真を撮影
AWS Rekognition で顔検索
  ├─ 一致あり → 「○○さんですね？」→ 既存プロフィールでログイン
  └─ 一致なし → 名前・所属を入力 → 新規プロフィール作成
                 → 顔をRekognitionコレクションに登録
```

```typescript
// 顔検索の呼び出し（抜粋）
const searchFace = async (imageBase64: string) => {
  const { data } = await supabase.functions.invoke("search-face", {
    body: { image: imageBase64 },
  });

  if (data?.matched) {
    // 既存ゲストが見つかった → プロフィール表示
    setStep("face-found");
    setMatchedProfile(data.profile);
  } else {
    // 新規ゲスト → 登録フォーム表示
    setStep("form");
  }
};
```

席次カード経由の場合、`profile_id` が既に紐付いているため、**名前と所属は事前に入っています**。ゲストは顔写真を撮るだけでプロフィールが完成します。

---

## 🌉 披露宴から二次会へのセッション引き継ぎ

ここが席次カードNFCの**真骨頂**です。

披露宴でNFC席次カードをスキャンしたゲストは、匿名セッションとプロフィールが既に存在しています。二次会でNFCチップ（`/c/[code]`）をタッチしたとき、システムは**既にこのゲストを知っている**のです。

```
【披露宴】
  席次カード（/m/[token]）
    → 匿名セッション作成
    → プロフィール作成（名前、顔写真）
    → 写真投稿、My Album を楽しむ

【二次会】
  チップNFC（/c/[code]）
    → 既にセッションあり！
    → チップ紐付けだけで完了
    → 即カジノへ
```

もし席次カードをスキャンしていないゲスト（二次会からの参加者など）がチップをタッチした場合は、その場でオンボーディングが始まります。**どちらのルートからでも、最終的に同じシステムに合流する**設計です。

```
披露宴組：席次カード → プロフィール済み → チップタッチ → 即カジノ
二次会組：チップタッチ → プロフィール登録 → カジノ
```

この「2つの入口、1つの出口」設計が、披露宴と二次会をシームレスに繋ぎます。

---

## 🖥️ 管理画面：新郎新婦2人で40人分を準備する

### ゲスト管理ダッシュボード

席次カードの準備は全て管理画面（`/admin/seat-cards`）から行います。新郎新婦がスマホまたはPCで操作します。

```
┌─────────────────────────────────────┐
│  席次カード管理                       │
│                                     │
│  全40名 | カード作成済: 38 | スキャン済: 0  │
│                                     │
│  ┌───────────────────────────────┐  │
│  │ 田中太郎  [新郎友人]           │  │
│  │ カード: ✅ | スキャン: ─       │  │
│  │ メッセージ: 太郎くん、今日は…   │  │
│  │ [編集] [URLコピー] [削除]      │  │
│  ├───────────────────────────────┤  │
│  │ 鈴木花子  [新婦友人]           │  │
│  │ カード: ✅ | スキャン: ─       │  │
│  │ メッセージ: 花子ちゃん、大学…   │  │
│  │ [編集] [URLコピー] [削除]      │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### メッセージの入力と更新

```typescript
// app/admin/seat-cards/actions.ts
"use server";

export async function createSeatCard(
  profileId: string,
  guestName: string,
  message: string
) {
  const supabase = await createClient();

  const token = crypto.randomUUID(); // 推測不可能なトークン

  await supabase.from("seat_cards").insert({
    token,
    profile_id: profileId,
    guest_name: guestName,
    message,
  });

  return { success: true, token };
}

export async function updateSeatCardMessage(
  token: string,
  message: string
) {
  const supabase = await createClient();

  await supabase
    .from("seat_cards")
    .update({ message })
    .eq("token", token);

  return { success: true };
}
```

管理画面ではメッセージをインライン編集できます。ゲスト名をタップ → テキストエリアが開く → 保存。40人分のメッセージを、結婚式の準備の合間に少しずつ書き溜めていけます。

### 当日の進捗モニタリング

`scanned_at` が入っているかどうかで、**当日の受付進捗がリアルタイムにわかります**。

```
スキャン済: 28/40 (70%)
未スキャン: 田中太郎, 佐藤次郎, ...
```

「まだスキャンしてないゲストがいるな」と把握できれば、受付係に「あの方にもカードを渡してあげて」と声をかけられます。

---

## 🧱 ぶつかった壁：「スマホでNFCが読めない」問題

### NFCの落とし穴

実装を進めていくと、現実的な問題にぶつかりました。

- iPhoneはNFCを読むのに**特定の位置にタッチ**する必要があり、慣れていないと読み取れない
- Androidは機種によってNFC対応状況がバラバラ
- スマホケースが厚いと読み取りにくい

**「結婚式の受付で『読めない！』は致命的」**

### 解決策：NFC + QRのデュアルインターフェース

そこで、全ての席次カードに**QRコードも印刷**しました。

```
┌─────────────────────────┐
│                         │
│    田中 太郎 様          │
│                         │
│   テーブルA / 席番号 3    │
│                         │
│    ┌─────┐              │
│    │ QR  │  ← 同じURLに │
│    │     │    アクセス   │
│    └─────┘              │
│                         │
│    [NFCタグ内蔵]         │
│                         │
└─────────────────────────┘
```

NFCで読めればスマート。読めなければQRをカメラで撮る。**どちらも同じ `/m/[token]` にアクセスする**ので、バックエンドの実装は完全に共通です。

前回のチップNFCでも同じデュアルインターフェースを採用しました。物理デバイスとWebアプリを繋ぐときは、**必ずフォールバック手段を用意する**のが鉄則です。

---

## NFC × 匿名認証 × 顔認証の統合パターン

### パターン1: NFC → 個別コンテンツ表示 → 認証の段階的導入
NFCタッチ直後は認証不要のコンテンツ（個別メッセージ）を表示し、次のアクション時に初めて認証を行う。「認証の壁」を感じさせない段階的なUX設計。

### パターン2: Supabase Anonymous Auth でゲスト認証
メールアドレスもパスワードも不要。`signInAnonymously()` でセッションを作り、ブラウザを閉じても維持される。イベント系アプリで「アカウント作成」のハードルを完全に排除できる。

### パターン3: 事前プロフィール + NFC紐付けの分離
管理者がプロフィールを事前作成 → 席次カード（NFC）のトークンと紐付け → ゲストがスキャンしたとき `supabase_user_id` を埋める。「誰であるか」は事前に決まっており、「どのセッションか」だけを当日に解決する設計。

### パターン4: NFC + QR のデュアルインターフェース
NFC読み取り失敗時の保険としてQRコードを併用。同一URLにアクセスするため、バックエンドの実装は共通。物理デバイス連携では信頼性のフォールバックが必須。

### パターン5: scanned_at による進捗トラッキング
初回スキャン時にタイムスタンプを記録し、管理画面で「未スキャンのゲスト」を可視化。当日の受付進捗をリアルタイムに把握できる。
<!-- /qiita-section -->

---

## 📐 2種類のNFC体験を振り返って

ここまでの連載で2種類のNFCを実装しました。

| | 席次カードNFC（第7回） | チップNFC（第6回） |
|--|--|--|
| **ルート** | `/m/[token]` | `/c/[code]` |
| **フェーズ** | 披露宴 | 二次会 |
| **物理形態** | 席次カード | カジノチップ |
| **目的** | 受付・個別メッセージ・プロフィール登録 | チップ紐付け・対戦・送金 |
| **最初に見えるもの** | 新郎新婦からのメッセージ | チップ登録 or 送金画面 |
| **認証方式** | Anonymous Auth（新規作成） | 既存セッション or Anonymous Auth |
| **セッション引き継ぎ** | ここで作られたセッションが二次会へ | 披露宴のセッションがあれば即完了 |

共通するのは「**NFCタッチを体験の起点にする**」という設計思想。テクノロジーの存在を意識させず、自然な動作がそのままデジタルの操作になる。この「境界のない体験」が、披露宴から二次会まで一貫した没入感を支えています。

---

## 次回予告

披露宴の席次カードで認証基盤が整いました。次回はカジノの本丸──**The Coliseum**。パリミュチュエル方式の賭けシステムで、動的オッズ計算、最低倍率保証、ロック→結果発表の演出設計を解説します。

**次回：【連載第8回】パリミュチュエル方式で実現するThe Coliseum**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-07-table-games)**
