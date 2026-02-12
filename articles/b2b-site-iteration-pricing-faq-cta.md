---
title: "B2Bサービスサイト公開後に気づいた「発注されない理由」と7つの改善実装"
emoji: "+"
type: "tech"
topics: ["astro", "tailwindcss", "typescript", "web"]
published: false
canonical: "https://blog.secure-auto-lab.com/articles/b2b-site-iteration-pricing-faq-cta"
---

![OGP](/images/b2b-site-iteration-pricing-faq-cta-ogp.png)


# B2Bサービスサイト公開後に気づいた「発注されない理由」と7つの改善実装

---

## 具体的な実装方法

### 全体の変更マップ

```
secure-auto-lab-portfolio/        ← B2Bサービスサイト
├── src/
│   ├── components/
│   │   ├── Hero.astro             ← コピー変更
│   │   ├── Header.astro           ← ナビ6項目化
│   │   ├── Contact.astro          ← カテゴリ追加
│   │   └── Process.astro          ← 詳細ステップ追加
│   ├── data/
│   │   └── pricing.ts             ← NEW: 料金データ
│   └── pages/
│       ├── pricing.astro          ← NEW: 料金ページ
│       ├── faq.astro              ← NEW: FAQページ
│       └── contact.astro          ← カテゴリ追加

article-publisher/blog/            ← 技術ブログ
├── src/
│   ├── components/
│   │   └── BlogCTA.astro          ← NEW: B2B誘導CTA
│   └── layouts/
│       └── ArticleLayout.astro    ← CTA統合
```

### Step 1: 料金ページ — TypeScriptデータ駆動 + JSON-LD

料金データをTypeScriptファイルに分離し、ページとJSON-LD構造化データの両方から参照する設計にしました。

```typescript
// src/data/pricing.ts
export interface PricingTier {
  name: string;
  price: string;
  description: string;
  duration: string;
}

export interface PricingCategory {
  id: string;
  title: string;
  icon: string;
  tiers: PricingTier[];
}

export const pricingCategories: PricingCategory[] = [
  {
    id: 'automation',
    title: '業務自動化・AI統合',
    icon: '⚡',
    tiers: [
      {
        name: 'スポット（小規模）',
        price: '10万円〜',
        description: 'SNS自動投稿、スクレイピング、レポート自動生成',
        duration: '1〜2週間',
      },
      // ...安い順に並べる
    ],
  },
  // 全4カテゴリ・14ティア
];
```

**ポイント：安い順に並べる。** 最初に見える価格が「10万円〜」と「80万円〜」では、訪問者の印象がまるで違います。ハードルを下げたいなら、最小価格を先に見せる。

料金ページではJSON-LD構造化データも出力しています：

```astro
---
// src/pages/pricing.astro
import { pricingCategories } from '../data/pricing';

const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'ProfessionalService',
  name: 'Secure Auto Lab',
  hasOfferCatalog: {
    '@type': 'OfferCatalog',
    name: 'サービス料金',
    itemListElement: pricingCategories.map((cat) => ({
      '@type': 'OfferCatalog',
      name: cat.title,
      itemListElement: cat.tiers.map((tier) => ({
        '@type': 'Offer',
        name: `${cat.title} - ${tier.name}`,
        description: tier.description,
      })),
    })),
  },
};
---
<script type="application/ld+json" set:html={JSON.stringify(jsonLd)} />
```

`ProfessionalService` + `OfferCatalog` スキーマで、Google検索に料金情報が構造化されて表示される可能性があります。

カードのグリッドは、カテゴリごとのティア数に応じて動的に列数を変えています：

```astro
<div class={`grid gap-4 ${
  cat.tiers.length === 2 ? 'md:grid-cols-2' :
  cat.tiers.length === 4 ? 'md:grid-cols-2 lg:grid-cols-4' :
  'md:grid-cols-3'
}`}>
```

### Step 2: FAQページ — JS 0KBのアコーディオン

FAQページで最も意識したのは、**JavaScriptを一切使わないこと**。HTMLの `<details>` / `<summary>` 要素はブラウザネイティブのアコーディオン機能を持っているため、開閉のためにJSは不要です。

```astro
---
// src/pages/faq.astro
const faqs = [
  {
    question: '個人（フリーランス）ですか？',
    answer: 'はい、個人事業主として活動しています。...',
  },
  // 全6問
];
---

{faqs.map((faq) => (
  <details class="faq-item group">
    <summary class="faq-summary">
      <span>{faq.question}</span>
      <span class="faq-icon">+</span>
    </summary>
    <div class="faq-answer" set:html={faq.answer} />
  </details>
))}
```

`+` アイコンの回転アニメーションもCSSだけで実現：

```css
.faq-icon {
  color: var(--color-gold);
  transition: transform 0.2s;
}

.faq-item[open] .faq-icon {
  transform: rotate(45deg);
}

/* ブラウザデフォルトの三角マーカーを消す */
.faq-summary::-webkit-details-marker {
  display: none;
}
```

`rotate(45deg)` で `+` が `×` に変わる。CSSだけで実現できるUIパターンは、積極的に活用すべきです。

構造化データは `FAQPage` スキーマ：

```javascript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: faqs.map((faq) => ({
    '@type': 'Question',
    name: faq.question,
    acceptedAnswer: {
      '@type': 'Answer',
      // HTML タグを除去してテキストのみ
      text: faq.answer.replace(/<[^>]*>/g, ''),
    },
  })),
};
```

FAQ回答にはHTML（例：料金ページへのリンク）を含む場合があるため、JSON-LD用にはHTMLタグを正規表現で除去しています。

### Step 3: ブログCTA — React不要のPure Astroコンポーネント

ブログ（`article-publisher/blog`）には `@astrojs/react` が入っていません。CTAコンポーネントをReactで書くと、Reactランタイムの追加（約40KB gzip）が必要になります。

**hover効果にJavaScriptは不要です。CSSの `:hover` で十分。**

```astro
---
// blog/src/components/BlogCTA.astro
interface Props {
  variant?: 'automation' | 'security' | 'appdev' | 'general';
}

const { variant = 'general' } = Astro.props;

const VARIANTS = {
  automation: {
    icon: '⚡',
    accent: '#00d4aa',
    accentDim: 'rgba(0,212,170,0.12)',
    tagline: 'この自動化、あなたの業務にも',
    headline: '業務自動化・AI統合を依頼しませんか？',
    // ...
  },
  security: { /* ... */ },
  appdev: { /* ... */ },
  general: { /* ... */ },
} as const;

const v = VARIANTS[variant] || VARIANTS.general;
---

<div class="blog-cta"
  style={`--cta-accent: ${v.accent}; --cta-accent-dim: ${v.accentDim};`}>
  <!-- カード本体 -->
</div>

<style>
  .cta-card:hover {
    border-color: color-mix(in srgb, var(--cta-accent) 40%, transparent);
    box-shadow: 0 0 30px color-mix(in srgb, var(--cta-accent) 8%, transparent);
  }
</style>
```

**設計のポイント：`color-mix()` 関数によるアクセント色の動的制御。**

CSS カスタムプロパティ `--cta-accent` にバリアントごとの色を設定し、`color-mix()` で透明度を動的に計算しています。これにより、4バリアントのhover効果をCSS 1つで実現しています。

### Step 4: 記事カテゴリ → CTAバリアントの自動マッピング

ブログ記事のカテゴリに応じて、表示するCTAバリアントを自動的に切り替えます。

```astro
---
// blog/src/layouts/ArticleLayout.astro
import BlogCTA from '../components/BlogCTA.astro';

const ctaVariantMap: Record<string, 'automation' | 'security' | 'appdev' | 'general'> = {
  automation: 'automation',
  ai: 'automation',       // AI記事 → 自動化CTA
  security: 'security',
  'dev-tips': 'general',
  projects: 'appdev',     // プロジェクト記事 → アプリ開発CTA
  web: 'appdev',          // Web記事 → アプリ開発CTA
  infrastructure: 'general',
};
const ctaVariant = ctaVariantMap[category] || 'general';
---

<!-- 記事本文の後に挿入 -->
<div class="my-12">
  <BlogCTA variant={ctaVariant} />
</div>
```

自動化の記事を読んでいる人には「業務自動化・AI統合を依頼しませんか？」、セキュリティの記事を読んでいる人には「セキュアなシステム開発を依頼しませんか？」と表示される。**読者の関心に合わせたCTAは、汎用CTAよりクリック率が高い**のは言うまでもありません。

### Step 5: 小さいけれど重要な3つの修正

**1. ヒーローコピー変更（1行の修正、インパクトは大）**

```diff
- あなたのビジネスを、技術で加速する。
+ あなたのアイデアを、技術で形にする。
```

「ビジネス」→「アイデア」で、個人やスタートアップを含む幅広い層に訴求。「加速する」→「形にする」で、ゼロからの開発にも対応する印象を与える。

**2. お問い合わせカテゴリ追加**

```html
<option value="automation">業務自動化・AI統合</option>
<option value="secure-dev">セキュアなシステム開発</option>
<option value="app-dev">アプリ・サービス開発</option>
<option value="lp-personal">LP制作・個人向けの相談</option>  <!-- NEW -->
<option value="other">その他</option>
```

「LP制作・個人向けの相談」を追加。個人でも気軽に選べる選択肢があるだけで、心理的ハードルが下がる。

**3. 導入の流れ 詳細化**

各ステップに具体的な手段と成果物を追加：

```typescript
const steps = [
  {
    num: '01', title: 'ヒアリング',
    desc: '課題・要件を整理し、ゴールを明確にします。',
    details: [
      'Slack・メール・Google Meetで対応',
      'NDA締結後のヒアリングも可能',
      '1〜2回のMTGで要件を整理',
    ],
  },
  // ...
];
```

「ヒアリング」とだけ書かれるより、「Slackで連絡できるんだ」「NDAにも対応してくれるんだ」とわかるほうが安心する。**抽象的なプロセスを具体化するだけで、信頼感は大きく変わります。**

---

<!-- qiita-section -->

## 実践Tips・Astroでよく使うパターン集

### Tips: `<details>/<summary>` でJS不要のアコーディオンを作る

AstroやNext.jsでFAQページを作るとき、わざわざReactの `useState` でアコーディオンを実装していませんか？

HTMLネイティブの `<details>/<summary>` を使えば、JSゼロで同じことが実現できます。

```html
<details class="faq-item">
  <summary class="faq-summary">
    <span>質問テキスト</span>
    <span class="faq-icon">+</span>
  </summary>
  <div class="faq-answer">回答テキスト</div>
</details>
```

```css
/* ブラウザデフォルトの三角マーカーを消す */
.faq-summary {
  list-style: none;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
}
.faq-summary::-webkit-details-marker {
  display: none;
}

/* 開閉アイコンのアニメーション */
.faq-icon {
  transition: transform 0.2s;
}
.faq-item[open] .faq-icon {
  transform: rotate(45deg);  /* + → × に変化 */
}
```

ポイント:
- `list-style: none` と `::-webkit-details-marker` の両方を指定（ブラウザ互換性）
- `[open]` 属性セレクタでスタイルを切り替え（JSイベント不要）
- `rotate(45deg)` で `+` が `×` に見える（フォントアイコン不要）

### Tips: Astroで型安全な料金データを管理する

料金のようなビジネスデータは、TypeScriptファイルに分離すると管理しやすくなります。

```typescript
// src/data/pricing.ts
export interface PricingTier {
  name: string;
  price: string;
  description: string;
  duration: string;
}

export interface PricingCategory {
  id: string;
  title: string;
  icon: string;
  tiers: PricingTier[];
}

export const pricingCategories: PricingCategory[] = [
  {
    id: 'automation',
    title: '業務自動化・AI統合',
    icon: '⚡',
    tiers: [
      {
        name: 'スポット（小規模）',
        price: '10万円〜',
        description: 'SNS自動投稿、スクレイピング、レポート自動生成',
        duration: '1〜2週間',
      },
    ],
  },
];
```

メリット:
- 料金変更時、データファイル1つを編集するだけ
- ページ本体とJSON-LD構造化データの両方から同じデータを参照
- TypeScriptの型チェックで、必須フィールドの漏れを防止

### Tips: `color-mix()` でCSS変数から動的に透明度を生成する

CSSカスタムプロパティに色を設定し、hover効果で透明度を変えたい場合があります。従来は `rgba()` で別の変数を用意する必要がありましたが、`color-mix()` なら1つの変数から動的に生成できます。

```css
:root {
  --accent: #00d4aa;
}

.card:hover {
  /* --accent の40%の不透明度 */
  border-color: color-mix(in srgb, var(--accent) 40%, transparent);
  /* --accent の8%の不透明度 */
  box-shadow: 0 0 30px color-mix(in srgb, var(--accent) 8%, transparent);
}
```

`color-mix(in srgb, 色 割合%, transparent)` で、任意の色に透明度を適用できます。

対応ブラウザ: Chrome 111+, Firefox 113+, Safari 16.2+ (2024年時点でほぼ全ブラウザ対応)

### Tips: JSON-LD構造化データでFAQをGoogle検索に表示する

Googleは `FAQPage` スキーマを認識すると、検索結果にFAQリッチリザルトを表示する場合があります。

```javascript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: faqs.map((faq) => ({
    '@type': 'Question',
    name: faq.question,
    acceptedAnswer: {
      '@type': 'Answer',
      text: faq.answer.replace(/<[^>]*>/g, ''),
    },
  })),
};
```

注意点:
- `text` フィールドにはHTMLタグを含めない（正規表現で除去）
- Astroでは `set:html={JSON.stringify(jsonLd)}` でインライン化
- [Google構造化データテストツール](https://search.google.com/test/rich-results)で検証可能

<!-- /qiita-section -->

---

## まとめ：今日からできるアクションプラン

この記事で解説した内容をまとめます：

1. **自己診断**: 自分のサービスサイトを「クライアントの目」で見直す。「発注の壁」5つの観点で課題を洗い出す
2. **料金の透明性**: 目安価格を掲載する。正確な見積りではなく「このくらいの規模感」が伝われば十分
3. **FAQの整備**: 問い合わせ前の小さな疑問を解消する。`<details>/<summary>` ならJS不要で実装可能
4. **ブログからの導線**: 技術記事を読んでいる人は潜在顧客。文脈に合ったCTAで自然に誘導する

**今日からできる具体的なアクション：**

> まずは信頼できる人（エンジニアでない人が理想）にサービスサイトを見てもらい、「問い合わせようと思うか？思わないとしたらなぜか？」を聞いてみてください。技術者の盲点は、非技術者の視点で初めて見えます。

---

## 参考リンク

- [前回の記事：B2Bサービスサイトを Astro 5 + Tailwind CSS v4 で構築した設計と実装](https://blog.secure-auto-lab.com/articles/astro5-b2b-service-site)
- [Astro 5 公式ドキュメント](https://docs.astro.build/)
- [Schema.org FAQPage](https://schema.org/FAQPage)
- [CSS color-mix() — MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color-mix)
- [Secure Auto Lab](https://secure-auto-lab.com)


---

**この記事の全文（ストーリー・背景解説を含む完全版）はブログで公開しています。**

**[>> ブログで全文を読む](https://blog.secure-auto-lab.com/articles/b2b-site-iteration-pricing-faq-cta)**
