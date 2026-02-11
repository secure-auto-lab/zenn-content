---
title: "B2Bサービスサイトを Astro 5 + Tailwind CSS v4 で構築した設計と実装"
emoji: "⚡"
type: "tech"
topics: ["astro", "tailwindcss", "typescript", "cloudflare"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/astro5-b2b-service-site"
---

![OGP](/images/astro5-b2b-service-site-ogp.png)


# B2Bサービスサイトを Astro 5 + Tailwind CSS v4 で構築した設計と実装

---

### 決め手になった3つのポイント

最終的に Astro 5 + Tailwind CSS v4 に決めた理由：

1. **ゼロJSがデフォルト**: `.astro` ファイルはサーバーでHTMLに変換され、クライアントにJSを送らない。必要な部分だけ React island で hydrate できる
2. **Tailwind CSS v4 の `@theme`**: CSS カスタムプロパティベースのデザイントークンを CSS ファイル1つで定義。JavaScript の設定ファイルが不要になった
3. **Cloudflare Pages との親和性**: `output: 'static'` で dist/ を丸ごとデプロイ。CDNエッジから配信されるので世界中で高速

**「使わない技術を選ばない」という判断が、結果的に最もパフォーマンスの良いサイトを生んだ** — これが今回の最大の学びでした。

---

## 具体的な実装方法

### 全体アーキテクチャ

```
secure-auto-lab-portfolio/
├── src/
│   ├── layouts/BaseLayout.astro    ← 共通レイアウト（SEO, Fonts, JSON-LD）
│   ├── components/                 ← Astro コンポーネント（9個）
│   │   └── react/                  ← React islands（カウンター）
│   ├── data/                       ← TypeScript データファイル
│   ├── pages/
│   │   ├── index.astro             ← メインページ（9セクション）
│   │   ├── services/               ← サービス詳細（3ページ）
│   │   ├── works/                  ← 実績詳細（6ページ）
│   │   ├── contact.astro
│   │   └── privacy.astro
│   └── styles/global.css           ← デザインシステム（Tailwind v4）
├── public/images/works/            ← プロジェクトスクリーンショット
└── astro.config.mjs
```

12ページを生成するが、ルーティングの設定は一切不要。`pages/` ディレクトリにファイルを置くだけでAstroがファイルベースルーティングを処理する。

### Step 1: Tailwind CSS v4 によるデザインシステム

Tailwind CSS v4 最大の変更点は、設定ファイル（`tailwind.config.ts`）が不要になったこと。`@theme` ディレクティブで CSS ファイル内にデザイントークンを直接定義できる。

```css
/* src/styles/global.css */
@import "tailwindcss";

@theme {
  /* Midnight Authority palette */
  --color-bg: #0A0A0F;
  --color-bg-card: #111118;
  --color-bg-hover: #1A1A24;
  --color-gold: #C9A96E;
  --color-gold-light: #E8D5A8;
  --color-gold-dim: #C9A96E40;
  --color-text: #F0F0F2;
  --color-text-sub: #9898A8;
  --color-text-muted: #5A5A6E;
  --color-border: #2A2A38;

  /* Fonts */
  --font-heading: "Shippori Mincho B1", serif;
  --font-body: "Noto Sans JP", "Inter", sans-serif;
  --font-english: "Inter", sans-serif;
  --font-mono: "JetBrains Mono", monospace;

  /* Animation easing */
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
}
```

このアプローチの利点：

- **CSS カスタムプロパティ**なので、Tailwind のクラスからも `var()` からも参照可能
- **1ファイルで完結** — デザイントークンの変更はここだけ
- **`@theme` 内の変数は Tailwind のユーティリティクラスとして自動的に使える**（例: `text-[var(--color-gold)]`、`bg-[var(--color-bg-card)]`）

### Step 2: BaseLayout — SEO と構造化データ

全12ページで共有する BaseLayout に、SEO の基盤を集約した。

```astro
---
// src/layouts/BaseLayout.astro
import '../styles/global.css';
import Header from '../components/Header.astro';
import Footer from '../components/Footer.astro';

interface Props {
  title: string;
  description?: string;
  ogImage?: string;
}

const {
  title,
  description = 'AI統合・業務自動化・セキュアなシステム構築。',
  ogImage = '/og-image.png',
} = Astro.props;

const canonicalURL = new URL(Astro.url.pathname, Astro.site);
---

<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="description" content={description} />
  <link rel="canonical" href={canonicalURL} />

  <!-- OGP -->
  <meta property="og:type" content="website" />
  <meta property="og:title" content={title} />
  <meta property="og:description" content={description} />
  <meta property="og:image" content={new URL(ogImage, Astro.site)} />

  <!-- JSON-LD 構造化データ -->
  <script type="application/ld+json" set:html={JSON.stringify({
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Secure Auto Lab",
    "url": "https://secure-auto-lab.com",
    "service": [
      { "@type": "Service", "name": "業務自動化・AI統合" },
      { "@type": "Service", "name": "セキュアなシステム開発" },
      { "@type": "Service", "name": "アプリ・サービス開発" },
    ],
  })} />

  <title>{title}</title>
</head>
<body class="min-h-screen">
  <Header />
  <main><slot /></main>
  <Footer />
</body>
</html>
```

ポイント：

- `Astro.site` と `Astro.url` でカノニカルURLとOGP URLを自動生成
- `set:html` で JSON-LD を安全にインライン化
- `@astrojs/sitemap` との組み合わせで `sitemap-index.xml` も自動生成

### Step 3: スクロールアニメーション（ゼロ依存）

Framer Motion や GSAP を使わず、**IntersectionObserver + CSS keyframes** だけで実装した。

```css
/* CSS keyframes */
@keyframes fade-up {
  from {
    opacity: 0;
    transform: translateY(24px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.animate-on-scroll {
  opacity: 0;
}

.animate-on-scroll.visible {
  animation: fade-up 0.6s var(--ease-out-expo) forwards;
}

/* 子要素のスタガー（時差）アニメーション */
.animate-on-scroll.visible > .stagger:nth-child(1) { animation-delay: 0ms; }
.animate-on-scroll.visible > .stagger:nth-child(2) { animation-delay: 100ms; }
.animate-on-scroll.visible > .stagger:nth-child(3) { animation-delay: 200ms; }
```

```html
<!-- 使い方：親に animate-on-scroll、子に stagger -->
<div class="grid md:grid-cols-3 gap-8 animate-on-scroll">
  <div class="stagger">カード1</div>
  <div class="stagger">カード2</div>
  <div class="stagger">カード3</div>
</div>
```

```javascript
// BaseLayout.astro 内の script タグ（全ページで共有）
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add('visible');
        observer.unobserve(entry.target);
      }
    });
  },
  { threshold: 0.1 }
);

document.querySelectorAll('.animate-on-scroll').forEach((el) => {
  observer.observe(el);
});
```

この方式の利点：

- **ライブラリ依存ゼロ** — バンドルサイズへの影響なし
- **GPU アクセラレーション** — `transform` と `opacity` のみ使用で60fps
- **一度だけ発火** — `unobserve` で再アニメーションを防止

### Step 4: データ駆動のページ生成

実績やサービスのデータを TypeScript ファイルに分離し、複数のページから参照する設計にした。

```typescript
// src/data/projects.ts
export interface Project {
  slug: string;
  title: string;
  subtitle: string;
  tags: string[];
  image: string;
}

export const projects: Project[] = [
  {
    slug: 'event-platform',
    title: 'リアルタイムイベント体験基盤',
    subtitle: '40人規模イベントでAI顔認識・即時報酬・P2P送金を実現',
    tags: ['Next.js 15', 'Supabase', 'AWS Rekognition', 'QR/NFC'],
    image: '/images/works/event-platform.png',
  },
  // ... 他5件
];
```

このデータは、メインページの実績グリッド、サービス詳細ページの関連実績、個別の実績詳細ページの3箇所から参照される。データの変更が即座に全ページに反映される。

![実績詳細ページ](./images/sal-works-detail.png)

### Step 5: Astro 設定とデプロイ

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import sitemap from '@astrojs/sitemap';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  site: 'https://secure-auto-lab.com',
  output: 'static',
  integrations: [react(), sitemap()],
  vite: {
    plugins: [tailwindcss()],
  },
});
```

Tailwind CSS v4 は `@astrojs/tailwind` ではなく `@tailwindcss/vite` を使う。Astro 5 は内部で Vite を使っているため、Vite プラグインとして直接登録する。

Cloudflare Pages へのデプロイは、GitHub リポジトリを接続するだけ：

- **ビルドコマンド**: `npm run build`
- **出力ディレクトリ**: `dist`
- **フレームワーク**: Astro

push するたびに自動ビルド・デプロイが走る。

---

## よくある質問（FAQ）

### Q1: Tailwind CSS v4 で v3 の `tailwind.config.ts` は使えますか？

A: 互換モードがありますが、新規プロジェクトなら `@theme` ディレクティブへの移行をお勧めします。CSS ファイル内で完結するため、設定ファイルの管理が不要になります。

### Q2: Astro で React コンポーネントを使う場合、SSR はどうなりますか？

A: `@astrojs/react` を追加すると、`.tsx` ファイルをインポートできます。`client:visible` ディレクティブを付けると、要素がビューポートに入った時に hydrate されます。今回はカウンターアニメーションだけ React island にしました。

### Q3: Cloudflare Pages のビルド設定は？

A: フレームワーク「Astro」を選択、ビルドコマンド `npm run build`、出力ディレクトリ `dist` だけです。カスタムドメインの設定も Cloudflare DNS と連携するので数クリックで完了します。

---

## まとめ：今日からできるアクションプラン

この記事で解説した内容をまとめます：

1. **技術選定**: B2Bサービスサイトには Astro 5 の静的出力が最適。必要な部分だけ React island で hydrate
2. **デザインシステム**: Tailwind CSS v4 の `@theme` で色・フォント・間隔を CSS 1ファイルに集約
3. **アニメーション**: IntersectionObserver + CSS keyframes でライブラリ不要の軽量アニメーション
4. **デプロイ**: Cloudflare Pages で GitHub 連携の自動デプロイ

**今日からできる具体的なアクション：**

> まずは `npm create astro@latest` で新規プロジェクトを作成し、`@tailwindcss/vite` を導入してみてください。`@theme` でカラーパレットを定義するだけで、デザインシステムの基盤が完成します。

---

## 参考リンク

- [Astro 5 公式ドキュメント](https://docs.astro.build/)
- [Tailwind CSS v4 ドキュメント](https://tailwindcss.com/docs)
- [Cloudflare Pages ドキュメント](https://developers.cloudflare.com/pages/)
- [Secure Auto Lab](https://secure-auto-lab.com)


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/astro5-b2b-service-site)**
