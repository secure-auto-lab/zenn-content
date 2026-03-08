---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第4回】写真が流れるスライドショーを作った話"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "postgresql", "framer-motion"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-04-slideshow"
---

![OGP](/images/wedding-it-04-slideshow-ogp.png)


# 写真が流れるスライドショーを作った話

## 〜ポラロイド風フレームが会場を漂い、リアルタイムに更新される〜

---

## 😰 「結婚式のスライドショー」の現実

結婚式でスライドショーを流す機会、ありますよね。

でも多くの場合、**事前に用意したスライドを順番に再生するだけ**。当日の写真は「後日」のお楽しみ。会場で今まさに撮られている写真がリアルタイムに映し出されることは、まずありません。

「自分が撮った写真が、数秒後にスクリーンに映る」

この体験を実現できたら、ゲスト全員が「もっと写真を撮りたい！」と思うはず。スライドショーはただの演出ではなく、**写真撮影を促進する仕組み**になるはずだ——そう考えました。

---

## 💭 なぜ「流れるスライドショー」にしたのか

### 従来のスライドショーの限界

最初に考えたのは、一般的なスライドショー。1枚ずつフルスクリーンで表示し、フェードトランジションで切り替える方式です。

しかし、実際に試すと問題がありました。

- **1枚あたりの表示時間が長い**：8秒間フルスクリーン → 次の写真。数百枚の写真を全て表示するには途方もない時間がかかる
- **ゲストが飽きる**：同じテンポの繰り返しは、数分で背景ノイズになる
- **「自分の写真が映った」瞬間を見逃す**：ずっとスクリーンを見ている人はいない

「スライドショーって、もっと動きがあっていいのでは？」

### 決め手になった3つのポイント

1. **同時表示**：複数の写真が画面上に存在するため、1枚あたりの「見せ時間」を長く取れる
2. **視覚的な楽しさ**：ランダムなサイズと位置で、見ていて飽きない
3. **情報密度**：フルスクリーン方式より単位時間あたりの表示枚数が多い

---

## 🎬 Framer Motion：写真が漂うアニメーション

このスライドショーの核は、Framer Motion による**フローイングアニメーション**です。

![スライドショー画面](/images/wedding-04-slideshow-main.png)

### ポラロイド風フレームの設計

各写真はポラロイド風の白いフレームに包まれ、撮影者名とAI認識されたタグ付きユーザーが表示されます。

```typescript
{/* 写真フレーム */}
<div className="relative bg-white rounded-2xl shadow-2xl p-3 inline-block">
  <img
    src={photo.url}
    alt=""
    className="rounded-xl block"
    style={{
      maxWidth: photo.size - 24,
      maxHeight: photo.size - 100,
    }}
    loading="lazy"
  />

  {/* 写真情報オーバーレイ */}
  <div className="absolute bottom-2 left-2 right-2
                  bg-white/90 backdrop-blur-sm rounded-lg p-3 shadow-lg">
    <p className="font-bold text-gray-800 text-sm truncate">
      📷 {photo.uploaderName}
    </p>
    {photo.taggedUsers.length > 0 && (
      <div className="flex flex-wrap gap-1 mt-1">
        {photo.taggedUsers.slice(0, 3).map((name, idx) => (
          <span key={idx}
            className="bg-pink-100 text-pink-700 px-2 py-0.5 rounded-full text-xs">
            {name}
          </span>
        ))}
      </div>
    )}
  </div>
</div>
```

フレームの下部には、半透明のオーバーレイで撮影者名を表示。AI顔認識でタグ付けされたユーザーはピンクのバッジで最大3人まで表示されます。「誰が撮って、誰が写っているか」が一目でわかる設計です。

### フローイングアニメーションの実装

写真は画面の右端から登場し、左端へとゆっくり流れていきます。

```typescript
<motion.div
  key={photo.id}
  initial={{
    x: window.innerWidth + 100,      // 右端の外から入場
    y: photo.startY,                  // ランダムなY位置
    opacity: 0,
    scale: 0.8,
    rotate: Math.random() * 6 - 3,   // ±3度のランダム回転
  }}
  animate={{
    x: -photo.size - 100,            // 左端の外へ退場
    opacity: 1,
    scale: 1,
  }}
  exit={{ opacity: 0 }}
  transition={{
    duration: photo.duration,          // 18〜26秒間
    ease: "linear",
    opacity: { duration: 0.5 },
    scale: { duration: 0.5 },
  }}
/>
```

ポイントは4つあります。

**1. ランダムなY位置**：写真が画面の縦方向にランダムに配置されるため、視線が固定されない。ただし写真全体が見切れないよう、`Math.max(0, window.innerHeight - size)` で上限を制御しています。

**2. 微かな回転（±3度）**：`Math.random() * 6 - 3` で、机の上にポラロイド写真を散らばせたような自然な傾きが生まれます。

**3. サイズのばらつき**：画面の65%〜95%のサイズ範囲でランダム生成。大きな写真と小さな写真が混在することで、奥行き感が出ます。

**4. 速度のばらつき**：18〜26秒の移動時間で、写真ごとに流れる速さが異なります。全て同じ速度だと単調ですが、速度差があることで自然な動きになります。

### 写真の追加タイミング

4秒ごとに新しい写真をフローに追加し、表示時間が過ぎた写真は `AnimatePresence` でフェードアウトして除去されます。

```typescript
// 定期的に写真を追加
useEffect(() => {
  if (photoQueue.length === 0) return;

  // 初期表示：段階的に3枚追加
  const initialTimeout1 = setTimeout(() => addFlowingPhoto(), 500);
  const initialTimeout2 = setTimeout(() => addFlowingPhoto(), 2000);
  const initialTimeout3 = setTimeout(() => addFlowingPhoto(), 4000);

  // 以降4秒ごとに追加
  const interval = setInterval(() => addFlowingPhoto(), 4000);

  return () => { /* cleanup */ };
}, [photoQueue, addFlowingPhoto]);
```

初期表示では0.5秒、2秒、4秒の間隔で3枚を段階的に投入。いきなり全部出すのではなく、1枚ずつ「登場」させることで、スライドショーが始まった感覚を演出しています。

---

## ⚡ Supabase Realtime：写真が瞬時に映る

スライドショーの最大の特徴は、**ゲストが写真をアップロードした瞬間にスクリーンに反映される**ことです。

### postgres_changes で INSERT を監視

Supabase Realtime の `postgres_changes` を使い、`photos` テーブルへの INSERT イベントをリアルタイムに購読します。

```typescript
const channel = supabase
  .channel("slideshow-photos")
  .on(
    "postgres_changes",
    {
      event: "INSERT",
      schema: "public",
      table: "photos",
    },
    async (payload) => {
      const newPhoto = payload.new as { storage_path: string; user_id: string };

      // アップローダーのプロフィールを取得
      const { data: profile } = await supabase
        .from("profiles")
        .select("display_name")
        .eq("id", newPhoto.user_id)
        .single();

      // キューの先頭に追加（最新写真を優先表示）
      setPhotoQueue((prev) => [
        {
          url: `${SUPABASE_URL}/storage/v1/object/public/photos/${newPhoto.storage_path}`,
          uploaderName: profile?.display_name || "Guest",
          taggedUsers: [],
        },
        ...prev,
      ]);
    }
  )
  .subscribe();
```

新しい写真は**キューの先頭に追加**されます。つまり、次にフローに投入される写真は最新のものになる。

ゲストが「今撮った写真」をアップロードすると、数秒後にはスクリーン上をポラロイド風フレームに包まれて流れ始める——この「即時性」が、ゲストの撮影意欲を最も刺激する設計です。

---

## 📊 優先度スコアリング：「盛り上がる写真」を先に

フロントエンドでは新しい順にシンプルに表示していますが、バックエンド側には将来の複数スクリーン対応を見据えた**優先度スコアリング**が実装されています。

### スコア計算式

```
優先度 = 未表示ボーナス(+200)
       + タグ付き人数 × 50
       - 表示回数 × 100
       + 最終表示からの経過時間（分）
```

この計算をSQLビューで実装しています。

```sql
CREATE VIEW view_slideshow_feed AS
SELECT
  p.id,
  p.storage_path,
  (
    -- 未表示ボーナス
    CASE WHEN p.last_shown_at IS NULL THEN 200 ELSE 0 END
    -- タグ付き人数ボーナス（1人につき+50）
    + COALESCE(
        (SELECT COUNT(*) * 50 FROM photo_tags WHERE photo_id = p.id), 0
      )::INTEGER
    -- 表示回数ペナルティ（1回につき-100）
    - (p.show_count * 100)
    -- 最終表示からの経過時間ボーナス（分単位）
    + COALESCE(
        EXTRACT(EPOCH FROM (NOW() - p.last_shown_at)) / 60, 0
      )::INTEGER
  ) AS priority_score
FROM photos p
WHERE p.is_blocked = FALSE
  AND p.is_valid_for_reward = TRUE
ORDER BY priority_score DESC, p.created_at DESC;
```

各要素の設計意図を解説します。

**未表示ボーナス +200**：まだ一度もスクリーンに表示されていない写真に最優先権を与えます。「全ての写真が一度は映る」ことを保証するための仕組みです。

**タグ付き人数 × 50**：AI顔認識で5人がタグ付けされた写真は250点。集合写真ほどスコアが高くなります。多くの人が写っている写真は、会場全体の盛り上がりが伝わるため、スクリーンに映す価値が高い。

**表示回数 × -100**：一度表示された写真は-100点のペナルティ。同じ写真が何度も繰り返し表示されることを強力に抑制します。初期設計では -5 でしたが、テストで同じ写真が繰り返し表示される問題が発生し、-100 に引き上げました。

**経過時間ボーナス**：最後に表示されてからの経過時間（分）がそのままスコアに加算されます。長時間表示されていない写真が徐々にスコアを回復し、再び表示されるチャンスを得る「自然な循環」を作ります。

---

## 🔒 FOR UPDATE SKIP LOCKED：複数スクリーンの排他制御

結婚式会場には複数のスクリーンを設置する想定です。各スクリーンでスライドショーを流しますが、**同じ写真が同時に複数のスクリーンに表示されたら意味がありません**。

ここで登場するのが、PostgreSQL の `FOR UPDATE SKIP LOCKED` です。

### 問題：同時アクセスでの重複

複数のスクリーンが同時に「次の写真をください」とリクエストすると、同じ最高スコアの写真を取得してしまう可能性があります。

通常の `FOR UPDATE` では、先にロックを取得したトランザクションが終わるまで他のトランザクションがブロックされます。スライドショーのように頻繁にアクセスする場面では、このブロッキングがパフォーマンスの問題になります。

### 解決策：SKIP LOCKED

`FOR UPDATE SKIP LOCKED` は、**ロック済みの行をスキップして、次に利用可能な行を取得する**という動作をします。

```sql
CREATE OR REPLACE FUNCTION fetch_next_slide()
RETURNS TABLE (
  id UUID,
  storage_path TEXT,
  uploader_id UUID,
  uploader_name TEXT,
  tagged_users JSONB
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_photo_id UUID;
BEGIN
  -- 優先度スコアに基づいて次の写真を取得（ロック付き）
  SELECT p.id INTO v_photo_id
  FROM photos p
  LEFT JOIN (
    SELECT photo_id, COUNT(*) AS tag_count
    FROM photo_tags
    GROUP BY photo_id
  ) pt ON pt.photo_id = p.id
  WHERE p.is_blocked = FALSE
    AND p.is_valid_for_reward = TRUE
  ORDER BY (
    CASE WHEN p.last_shown_at IS NULL THEN 200 ELSE 0 END
    + COALESCE(pt.tag_count * 50, 0)
    - (p.show_count * 100)
    + COALESCE(EXTRACT(EPOCH FROM (NOW() - p.last_shown_at)) / 60, 0)::INTEGER
  ) DESC, p.created_at DESC
  LIMIT 1
  FOR UPDATE SKIP LOCKED;

  IF v_photo_id IS NULL THEN RETURN; END IF;

  -- 表示情報を更新
  UPDATE photos
  SET last_shown_at = NOW(), show_count = show_count + 1
  WHERE photos.id = v_photo_id;

  -- 結果を返す（撮影者名・タグ付きユーザーをJOIN）
  RETURN QUERY
  SELECT p.id, p.storage_path,
         p.user_id AS uploader_id,
         prof.display_name AS uploader_name,
         COALESCE(
           (SELECT jsonb_agg(jsonb_build_object(
              'user_id', tags.user_id,
              'name', tag_prof.display_name,
              'confidence', tags.confidence
           ))
           FROM photo_tags tags
           JOIN profiles tag_prof ON tag_prof.id = tags.user_id
           WHERE tags.photo_id = p.id),
           '[]'::jsonb
         ) AS tagged_users
  FROM photos p
  JOIN profiles prof ON prof.id = p.user_id
  WHERE p.id = v_photo_id;
END;
$$;
```

この関数の動作を順を追って説明します。

1. **スコア計算 + 行ロック**：優先度スコアが最も高い写真を取得し、同時に `FOR UPDATE SKIP LOCKED` でロック。他のスクリーンがロック中の写真はスキップされる
2. **表示記録の更新**：選ばれた写真の `last_shown_at` を現在時刻に、`show_count` を+1に更新
3. **結果の返却**：写真データに加えて、撮影者名とタグ付きユーザー情報をJOINして返す

スクリーンAが最高スコアの写真をロックしている間、スクリーンBは**ブロックされずに**次点の写真を取得します。これにより、複数スクリーンが同時にリクエストしても、それぞれ異なる写真を重複なく表示できます。

### なぜ FOR UPDATE SKIP LOCKED なのか

他のアプローチと比較してみます。

| 方法 | 問題点 |
|------|--------|
| アプリ側でランダム選択 | 重複表示の可能性がある |
| Redis で排他制御 | インフラが増える、Supabase と相性が悪い |
| FOR UPDATE（SKIP LOCKEDなし） | 他のスクリーンがブロックされる |
| **FOR UPDATE SKIP LOCKED** | **ロック済みの行をスキップ、ブロックなし** |

PostgreSQL の機能だけで完結し、追加のインフラを必要としない。Supabase のRPC関数として呼び出せる。これが選定の決め手でした。

<!-- qiita-section -->

### FOR UPDATE SKIP LOCKED の実践的な使い方

`FOR UPDATE SKIP LOCKED` は、スライドショー以外にも様々な場面で活用できます。

**ジョブキューの実装**

複数のワーカーが同じキューからジョブを取得する場合、`SKIP LOCKED` を使えば同じジョブを二重処理することなく、ブロッキングも発生しません。

```sql
-- ジョブキューからの取得例
WITH next_job AS (
  SELECT id FROM jobs
  WHERE status = 'pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE jobs SET status = 'processing'
FROM next_job WHERE jobs.id = next_job.id
RETURNING jobs.*;
```

**GROUP BY との併用に注意**

`FOR UPDATE` は `GROUP BY` と同時に使えません。タグ数のようなCOUNTを使う場合は、サブクエリでタグ数を事前に集計し、LEFT JOINで結合する設計が必要です。

```sql
-- NG: GROUP BY と FOR UPDATE は同時に使えない
SELECT p.id, COUNT(pt.id)
FROM photos p
LEFT JOIN photo_tags pt ON pt.photo_id = p.id
GROUP BY p.id
FOR UPDATE SKIP LOCKED;  -- ERROR!

-- OK: サブクエリで事前に集計
SELECT p.id
FROM photos p
LEFT JOIN (
  SELECT photo_id, COUNT(*) AS tag_count
  FROM photo_tags GROUP BY photo_id
) pt ON pt.photo_id = p.id
ORDER BY ...
FOR UPDATE SKIP LOCKED;  -- OK!
```

**注意点**：

- `SKIP LOCKED` はトランザクション内でのみ有効です。トランザクションが終了するとロックが解放されます
- ロックされていない行が見つからない場合、空の結果を返します（ブロックしません）
- `ORDER BY` に使うカラムにインデックスがないと全行スキャンが発生します

**Supabase Realtime でリアルタイムスライドショーを実装する方法**

Supabase の `postgres_changes` を使えば、WebSocket経由でテーブルの変更をリアルタイムに受け取れます。

```typescript
// Supabase Realtime でテーブル変更を購読
const channel = supabase
  .channel("slideshow-photos")
  .on("postgres_changes", {
    event: "INSERT",
    schema: "public",
    table: "photos",
  }, async (payload) => {
    // 新しい写真が追加された時の処理
    const newPhoto = payload.new;
    // UIに反映...
  })
  .subscribe();

// クリーンアップ
return () => supabase.removeChannel(channel);
```

**Framer Motion で流れるアニメーションを実装する方法**

`AnimatePresence` と `motion.div` の組み合わせで、写真の入退場アニメーションを制御できます。

```typescript
import { motion, AnimatePresence } from "framer-motion";

<AnimatePresence>
  {photos.map((photo) => (
    <motion.div
      key={photo.id}
      initial={{ x: window.innerWidth + 100, opacity: 0, scale: 0.8 }}
      animate={{ x: -photo.size - 100, opacity: 1, scale: 1 }}
      exit={{ opacity: 0 }}
      transition={{
        duration: photo.duration,  // 18〜26秒
        ease: "linear",
        opacity: { duration: 0.5 },
      }}
      style={{ position: "absolute" }}
    >
      {/* 写真コンテンツ */}
    </motion.div>
  ))}
</AnimatePresence>
```

ポイント：
- `initial` でアニメーション開始状態（画面右端の外）を定義
- `animate` で最終状態（画面左端の外）を定義
- `exit` でフェードアウトを定義
- `transition` の `ease: "linear"` で等速移動（加減速なし）
- `opacity` と `scale` はサブプロパティで個別の duration を設定

<!-- /qiita-section -->

---

## 📝 次回予告

披露宴の写真システムが完成しました。写真をアップロードし、AIで顔認識し、スクリーンにリアルタイムで映し出す。この一連のパイプラインが、披露宴の「思い出を残す」体験を支えています。

次回からは**二次会篇**に入ります。披露宴で貯めたチップを使って、ゲスト同士が対戦し、ランキングを競い合う**チップ経済圏**の設計に踏み込みます。

ゼロサムにしないバランス設計、インフレ対策、PayPay風のチップ移動UI。新婦のために作り込んだゲームデザインの裏側をお見せします。

**次回：「ゼロサムにしないチップ経済圏の設計」**

---

**この記事が参考になったら、ぜひシェアをお願いします！**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-04-slideshow)**
