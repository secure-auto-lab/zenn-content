---
title: "エンジニアが新婦のために結婚式にITで全力で貢献しようとした話【連載第3回】写真をAI顔認識で自動タグ付けする仕組み"
emoji: "💒"
type: "tech"
topics: ["nextjs", "supabase", "aws", "個人開発"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/wedding-it-03-photo-ai"
---

![OGP](/images/wedding-it-03-photo-ai-ogp.png)


# 写真をAI顔認識で自動タグ付けする仕組み

## 〜HEIC変換、Face Collection、不正検出まで〜

---

## 😰 「結婚式の写真」の現実

結婚式の写真って、どうなってますか？

プロカメラマンが撮った集合写真は手元に届く。でも、ゲスト同士がスマホで撮った何百枚もの写真は？ たいてい「LINEのアルバムに上げといてー」で終わり、半数は共有されないまま埋もれていきます。

新婦が本当に欲しいのは、プロの正装写真だけじゃない。**ゲストが自然に笑っている何気ない1枚**だったりする。

「全員の写真が、自動で集まって、誰が写っているかまで分かる仕組みがあったら——」

この発想が、AI顔認識システムの出発点でした。

---

## 💭 なぜ「写真を撮ること」にゲーム要素を入れたのか

写真共有の仕組みを作るだけなら、Googleフォトの共有アルバムで十分です。

でも私が作りたかったのは、**「写真を撮ること自体が楽しくなる」** 仕組みでした。

カジノチップという経済圏を設計した時、写真撮影は最も自然にチップを流通させる手段だと気づきました。

- 写真を撮る → 撮影者にチップが入る
- 写真に写っている人が認識される → 被写体にもチップが入る
- チップが増えれば、カジノゲームでもっと遊べる

**「撮る人も、撮られる人も嬉しい」**——この双方向の報酬設計が、会場全体の写真撮影を促進する鍵でした。

写真を「義務」ではなく「ゲームの一部」にする。その実装を、これから詳しく解説します。

---

## 📸 写真アップロード：iPhoneの罠

結婚式のゲスト40人のうち、7割以上がiPhoneユーザーです。そして多くのiPhoneは、写真を**HEIC形式**で保存しています。

HEICはファイルサイズが小さく高画質な優れた形式ですが、ブラウザでの表示やサーバーでの処理は一筋縄ではいきません。「写真を選んでアップロードしたら真っ白」では、ゲストの体験を台無しにしてしまいます。

### クライアントサイドでJPEGに変換

サーバーに負荷をかけず、アップロード前にブラウザ側で変換する方針を採用しました。

```typescript
// lib/photo-upload.ts — HEIC判定と変換

export function isHeicFile(file: File): boolean {
  const type = file.type.toLowerCase();
  const name = file.name.toLowerCase();
  return (
    type === "image/heic" ||
    type === "image/heif" ||
    name.endsWith(".heic") ||
    name.endsWith(".heif")
  );
}

export async function convertHeicToJpeg(file: File): Promise<File> {
  // heic2anyライブラリを動的インポート（バンドルサイズ削減）
  const heic2any = (await import("heic2any")).default;

  const blob = await heic2any({
    blob: file,
    toType: "image/jpeg",
    quality: 0.9,
  });

  const resultBlob = Array.isArray(blob) ? blob[0] : blob;
  const newName = file.name.replace(/\.(heic|heif)$/i, ".jpg");
  return new File([resultBlob], newName, { type: "image/jpeg" });
}
```

ポイントは3つあります。

**1. 動的インポート**：`heic2any` は約800KBあるライブラリです。`import("heic2any")` で必要な時だけ読み込むことで、初回ロードを高速に保ちます。

**2. HEIF対応**：iPhoneの設定によっては `.heif` 拡張子で保存されることもあるため、HEIC/HEIF両方を判定しています。

**3. 品質0.9**：結婚式の写真は思い出として残るもの。圧縮率を上げすぎず、高品質を維持しました。

### リサイズで通信量を削減

さらに、高解像度の写真は2048pxにリサイズしてからアップロードします。

```typescript
export async function resizeImage(
  file: File,
  maxWidth: number = 2048,
  maxHeight: number = 2048
): Promise<File> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    const canvas = document.createElement("canvas");
    const objectUrl = URL.createObjectURL(file);

    img.onload = () => {
      URL.revokeObjectURL(objectUrl); // メモリリーク防止

      let { width, height } = img;
      if (width <= maxWidth && height <= maxHeight) {
        resolve(file); // リサイズ不要
        return;
      }

      // アスペクト比を維持してリサイズ
      const ratio = Math.min(maxWidth / width, maxHeight / height);
      width = Math.round(width * ratio);
      height = Math.round(height * ratio);

      canvas.width = width;
      canvas.height = height;
      canvas.getContext("2d")?.drawImage(img, 0, 0, width, height);

      canvas.toBlob(
        (blob) => {
          if (blob) resolve(new File([blob], file.name, { type: "image/jpeg" }));
          else reject(new Error("Canvas to blob failed"));
        },
        "image/jpeg",
        0.9
      );
    };
    img.src = objectUrl;
  });
}
```

iPhone 15 Proの写真は4800万画素（8064x6048px）。そのままアップロードすると1枚あたり10MB以上になります。2048pxへのリサイズで **ファイルサイズが約1/10に** 。40人が各10枚アップロードしても、ストレージは数百MBで済みます。

### アップロードUIの設計

ゲストのUXを最優先に設計しました。

![写真アップロード画面](./images/wedding-03-upload-ui.png)

```typescript
// app/reception/upload/page.tsx（一部抜粋）

// 初期化完了後、自動でファイル選択ダイアログを開く
useEffect(() => {
  if (isInitialized && photos.length === 0) {
    const timer = setTimeout(() => {
      fileInputRef.current?.click();
    }, 100);
    return () => clearTimeout(timer);
  }
}, [isInitialized, photos.length]);
```

ページを開いた瞬間にファイル選択ダイアログが自動表示されます。「写真を撮る → すぐアップロード」のフローを最速にするための工夫です。

アップロード中はサムネイルにスピナーを表示し、完了するとチェックマーク、エラーならバツ印のオーバーレイが表示されます。各写真にはオプションのコメント（最大50文字）を添えることができ、AI分析が完了するとその場で**認識された参加者の名前と信頼度**がリアルタイムに表示されます。

---

## 🤖 AWS Rekognition：顔認識パイプラインの全体像

「自分が写っている写真」を自動で見つける。これはゲストにとって最も嬉しい機能であり、同時にチップ経済圏の基盤でもあります。

システムは3つのEdge Functionで構成されています。

```
register-face: ゲスト登録時に顔を学習
     ↓
analyze-face: 写真アップロード時にAI分析
     ↓
search-face:  プロフィール検索時に顔を照合
```

### Face Collection の設計

AWS Rekognition の Face Collection は、顔の特徴ベクトルを格納する「顔データベース」です。

**登録フロー（register-face Edge Function）：**

1. ゲストがオンボーディングでアバター写真をアップロード
2. `register-face` Edge Functionが起動
3. Rekognition の `IndexFaces` APIで顔の特徴を登録
4. `ExternalImageId` にプロフィールIDを紐づけて保存

```typescript
// supabase/functions/register-face/index.ts（概要）

// 既存の顔データがあれば削除（再登録対応）
const existingFaces = await callRekognition("ListFaces", {
  CollectionId: COLLECTION_ID,
});
const oldFace = existingFaces.Faces?.find(
  (f) => f.ExternalImageId === profileId
);
if (oldFace) {
  await callRekognition("DeleteFaces", {
    CollectionId: COLLECTION_ID,
    FaceIds: [oldFace.FaceId],
  });
}

// 新しい顔を登録
await callRekognition("IndexFaces", {
  CollectionId: COLLECTION_ID,
  Image: { Bytes: imageBase64 },
  ExternalImageId: profileId,
  QualityFilter: "AUTO",
  MaxFaces: 1,
});
```

**検索フロー（analyze-face Edge Function）：**

1. 写真がアップロードされる
2. `analyze-face` Edge Functionが3段階のAI分析を実行
3. マッチした顔IDからプロフィールを逆引き
4. `photo_tags` テーブルに記録
5. データベーストリガーでチップ報酬を自動付与

### 3段階のAI分析パイプライン

`analyze-face` は単に顔を探すだけではありません。3つのRekognition APIを順番に呼び出す、多層の分析パイプラインになっています。

**Stage 1: DetectModerationLabels** — 不適切コンテンツの検出

```typescript
// 露骨な画像、暴力、武器、ヘイトシンボルをブロック
// ただしアルコール・タバコ・ギャンブルは結婚式なので許可
const moderationResult = await callRekognition("DetectModerationLabels", {
  Image: { Bytes: imageBase64 },
  MinConfidence: 70,
});
```

結婚式にはお酒もカジノゲームもあるので、アルコールやギャンブル関連のラベルは明示的に許可リストに入れています。

**Stage 2: DetectLabels** — スクリーンショット等の検出

```typescript
// system_configから動的にブロックリストを取得
const { data: config } = await supabase
  .from("system_config")
  .select("value")
  .eq("key", "blocked_photo_labels")
  .single();

// Screenshot, Text, Document, Web Page などを検出
```

ブロックリストはDBに保存しているので、**運用中に管理画面からリアルタイムに変更可能**です。当日「このラベルも弾きたい」と思ったら、SQLを書かずに管理画面から即座に対応できます。

**Stage 3: SearchFacesByImage** — 顔認識

```typescript
const searchResult = await callRekognition("SearchFacesByImage", {
  CollectionId: "wedding-guests",
  Image: { Bytes: imageBase64 },
  FaceMatchThreshold: 80,
  MaxFaces: 10,
});
```

マッチ閾値80%で、1枚の写真から最大10人まで認識します。結婚式の集合写真には大勢が写るため、MaxFacesは余裕を持たせています。

### SDKなし！手動Signature V4認証

Supabase Edge Functions（Deno環境）ではAWS SDKが使えません。そのため、**AWS Signature V4認証を手動で実装**しています。

```typescript
// HMAC-SHA256署名を手動計算
const kDate = hmacSHA256("AWS4" + secretKey, dateStamp);
const kRegion = hmacSHA256(kDate, region);
const kService = hmacSHA256(kRegion, "rekognition");
const kSigning = hmacSHA256(kService, "aws4_request");
const signature = hmacSHA256(kSigning, stringToSign);
```

SDKに頼らずAPI直接呼び出しすることで、Edge Functionのコールドスタートも高速に保てています。

---

## 💰 チップ報酬トリガーの設計

顔認識でタグ付けされると、**撮影者と被写体の両方**にチップが付与されます。このトリガーには、3つの不正対策が組み込まれています。

```sql
-- supabase/migrations/20260201100006_tagging_rewards.sql

CREATE OR REPLACE FUNCTION give_chips_on_tagging()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_photo_uploader_id UUID;
  v_tagger_reward INTEGER;
  v_tagged_reward INTEGER;
  v_cooldown_minutes INTEGER;
  v_last_reward_at TIMESTAMP WITH TIME ZONE;
  v_source_ref TEXT;
BEGIN
  -- 報酬設定をsystem_configから動的に取得
  SELECT
    (value->>'photo_tagger')::INTEGER,
    (value->>'photo_tagged')::INTEGER,
    (value->>'pair_cooldown_minutes')::INTEGER
  INTO v_tagger_reward, v_tagged_reward, v_cooldown_minutes
  FROM system_config WHERE key = 'chip_rewards';

  -- デフォルト: 撮影者30チップ、被写体100チップ
  v_tagger_reward := COALESCE(v_tagger_reward, 30);
  v_tagged_reward := COALESCE(v_tagged_reward, 100);
  v_cooldown_minutes := COALESCE(v_cooldown_minutes, 30);

  SELECT user_id INTO v_photo_uploader_id
  FROM photos WHERE id = NEW.photo_id;

  -- ① 自作自演チェック
  IF v_photo_uploader_id = NEW.user_id THEN
    NEW.reward_processed := TRUE;
    RETURN NEW;
  END IF;

  -- ② ペアクールダウン（30分）
  SELECT MAX(t.created_at) INTO v_last_reward_at
  FROM transactions t
  WHERE t.source_ref LIKE 'photo_tag:%'
    AND (
      (t.user_id = v_photo_uploader_id
        AND t.description LIKE '%' || NEW.user_id::TEXT || '%')
      OR
      (t.user_id = NEW.user_id
        AND t.description LIKE '%' || v_photo_uploader_id::TEXT || '%')
    )
    AND t.created_at > NOW() - (v_cooldown_minutes || ' minutes')::INTERVAL;

  IF v_last_reward_at IS NOT NULL THEN
    NEW.reward_processed := TRUE;
    RETURN NEW; -- クールダウン中
  END IF;

  -- ③ 冪等性保証（ON CONFLICT DO NOTHING）
  v_source_ref := 'photo_tag:' || NEW.id::TEXT;

  INSERT INTO transactions (user_id, amount, game_type, source_ref, description)
  VALUES
    (v_photo_uploader_id, v_tagger_reward, 'photo_reward',
     v_source_ref || ':tagger',
     '写真に ' || NEW.user_id::TEXT || ' がタグ付けされました'),
    (NEW.user_id, v_tagged_reward, 'photo_reward',
     v_source_ref || ':tagged',
     'あなたが写真にタグ付けされました')
  ON CONFLICT (source_ref) DO NOTHING;

  NEW.reward_processed := TRUE;
  RETURN NEW;
END;
$$;
```

3つの不正対策を解説します。

**① 自作自演チェック**：自分で自分を撮影した場合は報酬を付与しません。セルフィーでチップを稼ぐ行為を防ぎます。

**② ペアクールダウン（30分）**：同じ「撮影者↔被写体」ペアでは30分間に1回だけ報酬が発生します。友人同士で交互に撮り合ってチップを稼ぐ「チップ農場」を防ぐ仕組みです。双方向チェック（A→BもB→Aもカウント）なのがポイントです。

**③ `source_ref` による冪等性**：`'photo_tag:abc123:tagger'` のようなユニークキーで `ON CONFLICT DO NOTHING` 。ネットワーク不安定な会場で同じリクエストが二重に届いても安全です。

### 報酬の非対称設計

撮影者30チップ、被写体100チップという**非対称な報酬設計**にも理由があります。

- **被写体の報酬を高く**：写真に写ることを嫌がる人にもインセンティブを与える
- **撮影者の報酬を抑えめに**：大量撮影によるチップインフレを防ぐ
- **報酬額はDB設定**：当日の状況に合わせてリアルタイムに調整可能

---

## 🛡️ 不正画像の検出

チップが絡むシステムでは、不正が必ず試みられます。「ネットで拾った画像をアップロードしてチップを稼ぐ」行為を防ぐため、2段構えの検出を行います。

**第1段: DetectModerationLabels**

暴力・露骨な画像・武器・ヘイトシンボルなど、根本的に不適切なコンテンツを検出してブロックします。結婚式の場にふさわしくない画像を自動排除する安全弁です。

**第2段: DetectLabels**

スクリーンショット、テキスト画像、Webページのキャプチャなど、「カメラで撮った写真ではない」コンテンツを検出します。

ブロックされた写真は**アップロード自体は成功しますが、報酬対象外としてマーク**されます。「写真を削除する」のではなく「報酬を出さない」だけにすることで、誤検知時のダメージを最小限に抑えています。

管理者は管理画面でブロックされた写真を確認し、誤検知の場合は手動で解除できます。

![管理者認証画面](./images/wedding-03-admin-photos.png)

---

## 📝 次回予告

写真がアップロードされ、AIで顔認識されたら、次はそれを**会場のスクリーンに映し出す**番です。

次回は、CSS Ken Burns Effect で写真をドラマチックに表示するスライドショーと、複数スクリーンで同じ写真が重複表示されないようにする PostgreSQL の `FOR UPDATE SKIP LOCKED` を使った排他制御について解説します。

新婦のために作り込んだ、スライドショーの裏側をお見せします。

**次回：「Ken Burns EffectとFOR UPDATE SKIP LOCKEDで作るスライドショー」**

---

**この記事が参考になったら、ぜひシェアをお願いします！**

---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/wedding-it-03-photo-ai)**
