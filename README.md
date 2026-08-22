# 多言語QRメニュー — 技術ドキュメント

買い切り納品・納品後は店舗が自分で更新できる多言語デジタルメニュー。SaaS化しない。販売者側に月額費用・DB管理・アカウント管理が残らない構成。

## システム構成

```
1店舗 = 1つの独立したGitHubリポジトリ = 1つの納品物
```

- **ホスティング**: GitHub Pages(無料、公開リポジトリ)
- **データベース**: リポジトリ内の `products.json`(gitの履歴がそのまま変更履歴になる)
- **画像保存**: リポジトリ内の `images/`(ブラウザ側でWebP圧縮してから保存)
- **認証**: 店舗が自分のGitHubアカウントで発行する Fine-grained Personal Access Token
- **サーバーサイドコードなし**。HTML/CSS/JSのみで完結し、GitHub Pagesだけで動く

## ファイル構成

```
/
├── menu.html          公開メニュー(誰でも閲覧、ログイン不要)。products.jsonをfetchして表示
├── admin.html          店舗スタッフ専用の管理画面(本番、GitHub REST APIで実データを更新)
├── demo-admin.html      営業デモ用の管理画面(localStorageのみ、実データには一切触れない)
├── menu-demo.html       demo-admin.htmlと連動する公開メニューのデモ版
├── qr.html              サンプル用QRコード表示ページ
├── products.json        商品データ(下記「products.json仕様」参照)
├── images/               商品写真(WebP形式)
├── README.md            このファイル(技術者向け)
└── docs/
    └── SETUP.md          店舗納品用のセットアップ手順(専門用語なし)
```

## products.json 仕様

```json
{
  "store_name": { "ja": "...", "en": "...", "zh": "...", "ko": "..." },
  "categories": [
    { "id": "drink", "name": { "ja": "ドリンク", "en": "Drinks", "zh": "饮品", "ko": "음료" } }
  ],
  "items": [
    {
      "id": "item-001",
      "published": true,
      "category": "drink",
      "name": { "ja": "カフェラテ", "en": "Café Latte", "zh": "拿铁咖啡", "ko": "카페라떼" },
      "description": { "ja": "...", "en": "", "zh": "", "ko": "" },
      "price": 550,
      "image": "images/item-001-a1b2c3d4.webp"
    }
  ]
}
```

- `published: false` の商品は`menu.html`に表示されない(削除ではなく非表示)
- `name`/`description`の各言語が空欄の場合、`menu.html`側で自動的に`ja`へフォールバック表示する(翻訳API等は使わない)
- `image`は相対パス。存在しない/読み込み失敗時は`menu.html`側で🍽アイコンにフォールバックする
- 新しい`category`を増やす場合は`categories`配列に追加するだけでよい

## menu.html の設計

- `fetch("products.json?t=" + Date.now(), { cache: "no-store" })` で毎回最新を取得し、GitHub Pages/ブラウザキャッシュによって古いメニューが残り続けないようにしている
- 読み込み中はスピナー、失敗時は再読み込みボタン付きのエラー表示
- `published !== false` の商品のみ描画
- すべてのテキストは `textContent` で挿入し、`innerHTML`へユーザー入力(商品データ)を直接流し込まない(XSS対策)

## admin.html(本番管理画面)の設計

### GitHub API

GitHub Contents API (`/repos/{owner}/{repo}/contents/{path}`) のみを使用する。

- 読み込み: `GET`(sha付きで取得)
- 保存: `PUT`(直前に取得したshaを添えて更新。**保存の直前に必ずproducts.jsonを再取得し、最新shaで上書きする**ことで、他端末の変更を検知する)
- 画像追加: `PUT` で `images/{id}-{ランダム8桁hex}.webp` へ保存
- 409(競合)発生時は自動で上書きせず、「他の端末で変更された可能性があります。最新情報を読み込んでください。」と表示する

### owner/repo の自動判定

`admin.html`はコード内にリポジトリ名を一切ハードコードしていない。`window.location.hostname`(`{owner}.github.io`)と`window.location.pathname`の先頭セグメントから自動的に owner/repo を判定する。これにより、このリポジトリをテンプレートとして複製するだけで、店舗ごとに設定を書き換える必要がない。

### 認証情報の保存方式(重要な設計判断)

**`localStorage`にPersonal Access Tokenを保存する方式を採用した。** 判断理由:

- 対象ユーザー(非エンジニアの店舗スタッフ)は、開くたびに再ログインを求められることを許容できない。`sessionStorage`ではタブを閉じるたびに再設定が必要になり、実運用に耐えない
- リスクは「その端末が盗まれる/他人に操作される」場合に限定される。この脅威モデルに対しては、Fine-grained PAT自体が有効な軽減策になっている:
  - スコープをこのリポジトリ1つ・Contents権限のみに限定できる(店舗の他の情報には触れない)
  - 有効期限を店舗自身が設定できる(最大1年、GitHubの仕様)
  - 店舗はいつでも github.com から自分でトークンを失効できる
  - 管理画面にも「この端末の設定を削除」ボタンを用意し、端末を手放す際にすぐ消せるようにしている
- トークンはコード・products.json・外部DBのいずれにも一切保存されず、URLパラメータにも含めない。`console.log`にも出力しない

### 保存処理の設計(部分失敗への対処)

GitHub Contents APIには複数ファイルにまたがる完全なトランザクションが存在しない。そのため「画像は保存できたがproducts.jsonの更新に失敗した」という状態が起こり得る。この設計では:

1. 画像を先にアップロードする(失敗したら即座に中断・エラー表示、products.jsonには触れない)
2. 画像アップロード成功後、products.jsonを**その時点で再取得**し、最新shaに対して更新する
3. products.json更新が失敗しても、アップロード済みの画像ファイルは孤立するだけで、表示中のメニューが壊れることはない(存在しない画像パスを指すことはなく、失敗時は元のproducts.jsonのままなので実害はない)
4. 保存ボタンは処理中disabledにし、二重送信を防ぐ

## demo-admin.html / menu-demo.html(営業デモ)

見た目・操作感は`admin.html`とほぼ同一だが、GitHub APIには一切アクセスせず、`localStorage`(キー: `demo_menu_products_v1`)だけを読み書きする。`menu-demo.html`も同じキーを参照するため、デモ管理画面での変更がその場でデモ公開画面に反映される体験を、実データに触れずに提供できる。「デモをリセット」で初期状態に戻せる。

## セキュリティ

- XSS: 商品名・説明等のユーザー入力は常に`textContent`で挿入。`innerHTML`は固定の骨組みマークアップにのみ使用し、ユーザー入力を直接連結しない
- 入力検証: 商品名(日本語)必須・価格は数値かつ0以上のみ許可
- 画像: MIMEタイプチェック・20MB上限・ブラウザ側でリサイズ後にアップロード
- トークン: ハードコードなし、products.json/リポジトリ/外部DBに保存しない、URLにもログにも出さない
- 店舗間の分離: リポジトリ自体が店舗ごとに独立しているため、店舗Aのトークンは店舗Bのリポジトリに対して権限を持たない(Fine-grained PATをそのリポジトリのみにスコープしている前提)

## 開発方法

サーバーサイドのビルドは不要。ローカルでは `python -m http.server` 等の静的サーバーで直接確認できる。`admin.html`の実データ確認にはGitHubのFine-grained PATが必要(テストにはPlaywright等でGitHub APIをモックする方法を推奨)。
