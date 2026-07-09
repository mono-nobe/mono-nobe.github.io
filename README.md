# Oira 公開 Web（GitHub Pages ユーザーサイト）

M23(1) の成果物。**Google Play 審査の必須要件（プライバシーポリシー・利用規約・アカウント削除リクエスト）**と、**札共有ディープリンク（M22(4)）の受け皿・App Links 検証ファイル**を配信する静的サイト。

> 文書はすべて日本語（CLAUDE §ドキュメント言語）。世界観トーン（静謐・朱は印のみ・絵文字なし）を保つ。

---

## 構成

| ファイル | 役割 | 対応 |
| --- | --- | --- |
| `index.html` | サイトトップ（各ページへの入口） | — |
| `privacy.html` | プライバシーポリシー | M23(1)① / Play 必須・Data safety と整合 |
| `terms.html` | 利用規約（UGC 禁止事項を含む） | M23(1)② / Play UGC ポリシー |
| `account-deletion.html` | アカウント削除リクエスト（ウェブ経路） | M23(1)③ / Play 必須（アプリ内と別） |
| `404.html` | 共有リンク `/m/<code>` の受け皿 ＋ 汎用 404 | M23(1)④ / M22(4) |
| `.well-known/assetlinks.json` | Android App Links 検証ファイル | M23(1)⑤ / M22(4) |
| `style.css` | 共通スタイル（図録トーン） | — |
| `.nojekyll` | Jekyll を無効化（`.well-known` を配信させるため必須） | — |

### なぜ `.nojekyll` が必要か
GitHub Pages の既定ビルダー（Jekyll）は、**ドット始まりのディレクトリ（`.well-known`）を出力から除外する**。空の `.nojekyll` を置くと Jekyll 処理を止め、すべてを静的配信するため、`.well-known/assetlinks.json` が `https://<host>/.well-known/assetlinks.json` で 200 応答する。

### なぜ `404.html` が受け皿になるか
`/m/<code>` はサーバー上に実ファイルが無いため、GitHub Pages は `404.html` を返す。その JS が `location.pathname` から館コードを取り出し、「アプリで開く（`oira://m/<code>`）」と「Google Play で入手」を出す。**App Links が有効な端末では、ブラウザに来る前にアプリが直接開く**（この受け皿は未インストール者向け）。

### 既知の制約：`/m/<code>` の HTTP ステータスは 404
上記の仕組み上、`/m/<code>` は **ブラウザには受け皿ページが正しく描画されるが、HTTP ステータスは 404** を返す（GitHub Pages はカスタムリライトに対応しないため、任意サブパスを 200 で返せない）。
- **人が開く分には問題ない**（ページは表示され、App Links 有効端末はそもそもブラウザに来ずアプリが開く）。
- **機械的なチェックには影響しうる**：リンクプレビュー生成・URL 監視・審査時のクローラは「壊れた URL（404）」と扱う可能性がある。共有リンクを SNS 等でプレビューさせたい場合は特に留意。
- **200 で返したい場合の代替**（採用時はコード変更＝別作業）：
  - (a) 共有リンク形式を `/m/?code=<code>`（クエリ方式）にし、`web/m/index.html`（実ファイル＝200）で受ける。ただし deep link 形式が変わるため `EXPO_PUBLIC_WEB_BASE_URL` 由来の `app.config.ts` `intentFilters`・`src/lib/deepLink.ts`（`CODE_RE`/パス解析）・共有リンク生成（`lib/share.ts`）・`web/404.html` の解析を合わせて改修する必要がある（決定 #58 のリンク形式変更）。
  - (b) リライト可能なホスト（Netlify/Cloudflare Pages 等）へ寄せる。ただし App Links 用の `.well-known/assetlinks.json` をルート直下で配信できることが条件（決定 #52 の GitHub Pages 採用を見直す）。
- 現状は **GitHub Pages ＋ `/m/<code>` ＋ 404 受け皿**を維持（Android の主経路は App Links＝ブラウザを経由しないため実害が小さい）。方式変更が必要になったら上記 (a)/(b) を検討する。

---

## デプロイ手順（GitHub Pages ユーザーサイト）

App Links は **ユーザーサイト `https://<username>.github.io`（ルート直下配信）が必須**。プロジェクトページ（`<username>.github.io/<repo>/` のサブパス）では `assetlinks.json` がルート直下に来ないため不可。

1. GitHub で `<username>.github.io` という名前の**公開リポジトリ**を作成する（`<username>` は自分の GitHub ユーザー名）。
2. この `web/` ディレクトリの中身（`.well-known/`・`.nojekyll` を含む）をそのリポジトリのルートに置いて push する。
3. リポジトリの Settings › Pages で、Source をデプロイ元ブランチ（例 `main` / ルート）に設定する。
4. 数分後、`https://<username>.github.io/` で公開される。

### 公開前に必ず差し替える箇所
- **連絡先メールアドレス**：`privacy.html` / `terms.html` / `account-deletion.html` の
  `【公開前に…設定してください】` を実際の連絡先に置換する（Play 審査で有効な連絡先が必要）。
- **`.well-known/assetlinks.json` の SHA256 フィンガープリント**（2つのプレースホルダ）：
  - `REPLACE_WITH_PLAY_APP_SIGNING_SHA256_FINGERPRINT` … Play Console › リリース › アプリの署名 で表示される
    **アプリ署名鍵（Play App Signing）**の SHA-256（App Links の一致にはこれが本命）。
  - `REPLACE_WITH_UPLOAD_KEY_SHA256_FINGERPRINT` … `eas credentials -p android` で得られる**アップロード鍵**の SHA-256。
  - どちらか一方でも配列に含めれば検証は通るが、両方入れておくと Play 配布ビルド・ローカルビルドの双方で App Links が成立する。

### アプリ側との整合（単一設定源）
アプリの `EXPO_PUBLIC_WEB_BASE_URL`（**EAS 環境変数**＝各ビルドプロファイルに設定。ローカル検証時のみ `.env`）に
**`https://<username>.github.io` を設定する**。`eas.json` に固定値や空値を直書きしない（プロファイルごとに
EAS 環境変数で与える＝決定 #52 の「リモート%配信基盤は作らず EAS 環境変数で刻む」方針・空値の混入を防ぐ）。
- `app.config.ts` がこの host を Android `intentFilters` に注入する（App Links の host）。
- `src/lib/deepLink.ts` が受理する共有リンクの host もこの値由来（他ホストは拒否）。
- host が一致しないと App Links は検証されない（https リンクがアプリに向かわない）。
- ネイティブ設定（intentFilters）変更のため、値を入れたら **Dev Client / release を再ビルド**する。

---

## 検証（公開後）

- `https://<username>.github.io/.well-known/assetlinks.json` が JSON を 200 で返す（リダイレクトなし・HTTPS 直応答）。
- Google の [Statement List Generator and Tester](https://developers.google.com/digital-asset-links/tools/generator) で package 名とフィンガープリントが一致する。
- 端末で `https://<username>.github.io/m/<code>` を開く：インストール済み→アプリが該当館を開く／未インストール→この受け皿ページが出る。
- 各ページ（privacy / terms / account-deletion）がスマートフォン幅で崩れず読める。
