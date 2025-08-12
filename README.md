# ko2-sinmei-app

## 1) ko2-sinmei-app の定義
**ko2-sinmei-app** は、保険・法務用途の医療関連 PDF/画像から、**Google Gemini** を用いて
- 文書の種類の判定（ページ分類）
- 受傷日・医療機関名・通院回数などの抽出
- 画像からの診療情報抽出
を行い、**最終的な集計結果（sinmei_results）** を生成する Web アプリです。

**Vercel** へのデプロイに最適化されており、**大容量ファイルのアップロード**は **Vercel Blob** にクライアントから直接送信（直送）し、サーバ側は **Blob の生URLをクライアントへ開示しません**。  
ダウンロード時は **Supabase 認証必須**・**10分間有効**・**1回限り使用**のトークン（JWT）を発行し、**独自のダウンロードAPI経由**で配布します。

---

## 2) 仕組み（要点）
- **アップロード**：フロント → Vercel Blob へ**直送**（4.5MB制限を回避）  
  - 直送のための短期トークンを `/api/blob/upload` が発行（Vercel公式の `handleUpload` パターン）
- **サーバ処理**：`/api/upload` が Blob の PDF を取得 → **44MB以下に分割** → 分割PDFを**再び Blob に保存**  
  - 分割結果は **Blob の生URLを KV にのみ保存**（キー `part:<partId>`）、**クライアントには partId のみ返却**
- **安全なダウンロード**：  
  - クライアントが「一時リンク発行」を押すと `/api/blob/ticket` が **10分・1回限りのトークン（JWT）** を生成し、  
    **自前の `/api/blob/download?token=...`** を返却  
  - `/api/blob/download` は **Supabase 認証**＋**JWT検証**＋**KV の `GETDEL` で一回限り消費**し、サーバが Blob から取得して**ストリーミング返却**  
  - これにより **Blob の実URLはネットワークパネルにも出ない**
- **AI 処理**：`/api/classify` / `page_meta` / `extract_info` などが **lib/gemini.ts** を経由して Gemini を呼び出し

> 用語メモ  
> - **直送（ダイレクトアップロード）**：ブラウザがストレージへ直接アップロードする方式。サーバの本文サイズ制限にかからない。  
> - **KV**：Key-Value ストア（Vercel KV / Upstash Redis）。今回、**一時チケット**や **partId→実URL** を保存。  
> - **JWT**：改ざん防止の署名付きトークン。**有効期限**や**利用者ID**などを詰められる。

---

## 3) 環境変数
Vercel の Project Settings → **Environment Variables** に設定（ローカルは `.env.local`）。

| 変数名 | 必須 | 用途 |
|---|:---:|---|
| `GEMINI_API_KEY` | ✅ | Google Generative AI の API キー |
| `BLOB_READ_WRITE_TOKEN` | ✅ | Vercel Blob 接続時に付与（Storage 画面で作成） |
| `TOKEN_SIGNING_KEY` | ✅ | JWT 署名用のランダム文字列（32+バイト推奨） |
| `KV_REST_API_URL` | ✅ | Vercel KV 接続情報 |
| `KV_REST_API_TOKEN` | ✅ | Vercel KV 接続トークン |
| `SUPABASE_URL` | ✅ | Supabase プロジェクト URL |
| `SUPABASE_ANON_KEY` | ✅ | Supabase 公開鍵（SSRでも使用） |
| `NEXT_PUBLIC_BASE_URL` | ✅ | 自サイトのベースURL（例: `https://your-app.vercel.app`） |

> フロントで Supabase クライアントを使う場合は `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` も必要です（本プロジェクトは SSR 認証を基本方針としています）。

---

## 4) 依存関係のインストール
- 必須ライブラリ
  - npm i next react react-dom
- AI
  - npm i @google/generative-ai
- PDF操作
  - npm i pdf-lib
- Blob（直送＆サーバ保存）
  - npm i @vercel/blob
- 認証・トークン・KV
  - npm i @supabase/ssr @supabase/supabase-js jose @vercel/kv

---

## 5) 技術ステック
- フロント：Next.js (App Router)
- 認証：Supabase Auth（SSR：@supabase/ssr）
- ストレージ：Vercel Blob（公開URLは露出せず、常に自前APIでプロキシ）
- 一時データ：Vercel KV（チケット・partメタデータ）
- AI：Google Gemini（gemini-2.5-flash-preview-05-20 / gemini-2.5-pro-preview-06-05）
- PDF：pdf-lib
- Python スクリプト（ローカル/Cloud Run向け）：画像化・Excel転記（Vercel 上では実行しない）

---

## 6) 処理全体の構成（工程）
1. アップロード（/upload ページ）
- ブラウザ → Blob に直送（/api/blob/upload でトークン発行）
- /api/upload へ pdfUrl / excelUrl を送信
2. PDF 分割（/api/upload）
- 44MB以下になるようにページ単位で分割 → 各パートを Blob へ保存
- **KV に part:<uuid> として {url, filename, contentType, groupId} を保存
- フロントへは partId のみ返却
3. AI 処理（必要時）
- /api/classify（ページ分類）、/api/page_meta（受傷日・医院名・通院回数）、/api/extract_info（画像抽出）
- /api/correct_names（病院名正規化）、/api/hospital_filter（通院回数=0を除外）、/api/merge（必要ページ結合）、/api/complete（受傷日付与）
- Blob のデータは サーバ側のみで取得（クライアントへURLは渡さない）
4. ダウンロード（必要時）
- /api/blob/ticket：10分・1回限りのダウンロードURL（自前API）を発行
- /api/blob/download?token=...：Supabase 認証 & JWT 検証 & KV GETDEL による単回消費 → サーバが Blob から取得してストリーミング返却

---

## 7) セットアップ手順
1. 依存インストール：npm i（前述のライブラリ）
2. Vercel で Blob を作成（Storage → Blob）→ BLOB_READ_WRITE_TOKEN が自動付与
3. Vercel KV を有効化（Storage → KV）→ 接続情報を環境変数に設定
4. Supabase プロジェクト作成 & Auth 有効化（メール/パスワード等）
5. 環境変数を設定（上表）
6. 開発起動：npm run dev
7. Vercel デプロイ：GitHub 連携 → Deploy → 環境変数を Production/Preview に適用

---

## 8) 実行フロー（UI 観点）
1. /upload で PDF と Excel を選択 → アップロード
2. サーバが分割 → フロントには partId リストが表示
3. 必要なパートで「一時リンクを発行」ボタン → /api/blob/ticket が 自前ダウンロードURL を返却
4. そのURLを開くと /api/blob/download が 1回だけダウンロードを許可（要ログイン）

---

## 9) プロジェクト構成図
```
ko2-sinmei-app/
├─ app/
│  ├─ upload/
│  │  └─ page.tsx
│  └─ api/
│     ├─ blob/
│     │  ├─ upload/route.ts       # 直送トークン発行
│     │  ├─ ticket/route.ts       # 10分・1回限りのDLチケット
│     │  └─ download/route.ts     # 自前DL API（認証＋単回消費）
│     ├─ upload/route.ts          # PDF分割→Blob保存→KV登録（URLは返さない）
│     ├─ classify/route.ts        # PDFページ分類（Gemini）
│     ├─ extract/route.ts         # ✅ document_type抽出（必要ページの確定）
│     ├─ page_meta/route.ts       # 受傷日・医院名・通院回数（Gemini）
│     ├─ extract_info/route.ts    # 画像から診療情報抽出（Gemini）
│     ├─ correct_names/route.ts   # 病院名の表記揺れ補正（Gemini）
│     ├─ hospital_filter/route.ts # 通院回数=0を除外
│     ├─ merge/route.ts           # 必要ページのPDF結合
│     └─ complete/route.ts        # 受傷日を最終結果へ付与
├─ components/
│  └─ UploadForm.tsx
├─ lib/
│  ├─ gemini.ts
│  ├─ pdf_split.ts
│  ├─ auth.ts
│  └─ constants.ts                # ✅ 分類ラベル等の単一ソース
├─ types/
│  └─ medical.ts
├─ scripts/
│  ├─ convert_pdf_to_images.py
│  └─ write_to_xlsm.py
└─ README.md
```

---

## 10) 依存関係マップ
```
┌─────────────────────────────── UI / フロント ────────────────────────────────┐
│ [components/UploadForm.tsx]                                                  │
│   ├─ (直送) → [/api/blob/upload] ── handleUpload ──→ Vercel Blob             │
│   └─ POST → [/api/upload]  (pdfUrl, excelUrl)                                │
└──────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────── サーバ処理（アップロード） ──────────────────────────────┐
│ [/api/upload]                                                                   │
│   ├─ fetch( Blob の pdfUrl )                                                    │
│   ├─ [lib/pdf_split.ts] で 44MB 以下に分割                                       │
│   ├─ put() → Vercel Blob（分割PDFを再保存）                                      │
│   └─ kv.set( "part:<uuid>" = { url, filename, contentType, groupId } )          │
│        ↑  ※ クライアントへは URL を返さず、partId（<uuid>）のみ返却               │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────── サーバ処理（AI・整形） ────────────────────────────────┐
│ [/api/classify] ──┐                                                              │
│                   │  (各ページの document_type 一覧)                              │
│ [/api/page_meta] ─┤──→ 後段へ（injured_date / clinic_name / hospital_visit_times）│
│                   │                                                              │
│ [/api/extract_info]（画像→診療情報）                                             │
│ [/api/correct_names]（病院名 正規化）                                            │
│ [/api/hospital_filter]（hospital_visit_times = 0 を除外）                         │
│                                                                                   │
│ [/api/extract]  ←── 参照: [lib/constants.ts]（抽出対象の定義）                    │
│   └─ (必要ページ番号リスト) ──→ [/api/merge] ──→ 抽出PDF（Blobに保存）             │
│                                                                                   │
│ [/api/complete]                                                                  │
│   └─ (sinmei_results + page_meta の injured_date) → 最終結果 JSON                 │
│                                                                                   │
│ （共通）[lib/gemini.ts]  ← classify / page_meta / extract_info / correct_names    │
│           └─ @google/generative-ai → Google Gemini                                │
│                                                                                   │
│ 型定義： [types/medical.ts]（各 API の入出力スキーマ）                            │
└───────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────── セキュア配布（10分・1回） ───────────────────────────────┐
│ [/api/blob/ticket]  (POST: { partId })                                           │
│   ├─ [lib/auth.ts]（Supabase 認証 requireUser）                                  │
│   ├─ jose(SignJWT): { jti, sub=user.id, partId }  有効期限=10分                   │
│   └─ kv.set( "ticket:<jti>", { partId, url, filename, contentType, userId }, TTL=600 ) │
│        └→ レスポンス: downloadUrl = [/api/blob/download?token=...]               │
│                                                                                   │
│ [/api/blob/download]  (GET: ?token=...)                                          │
│   ├─ [lib/auth.ts]（Supabase 認証 requireUser）                                  │
│   ├─ jwtVerify(token), sub === user.id を検証                                     │
│   ├─ kv.getdel( "ticket:<jti>" )   ← 単回消費                                     │
│   └─ fetch( Blob 実URL ) → ストリーミング返却（Content-Disposition: attachment）  │
│        ※ Blob の実URLはクライアントへ開示しない                                  │
└───────────────────────────────────────────────────────────────────────────────────┘
```
【主要依存の関係】
- フロント直送: UploadForm.tsx → /api/blob/upload → Vercel Blob
- 分割・登録  : /api/upload → lib/pdf_split.ts, @vercel/blob, @vercel/kv
- AI 呼び出し : classify / page_meta / extract_info / correct_names → lib/gemini.ts → Gemini
- 抽出/結合   : extract（constants参照） → merge
- 最終整形   : complete（page_meta の injured_date を付与）
- 認証        : すべてのダウンロード系（ticket / download）で Supabase 必須（lib/auth.ts）

---

## 11) 注意事項（セキュリティ・運用）
- Blob の実URLは公開仕様です。クライアントへは一切返さない設計を厳守してください。
- ダウンロードは 必ず /api/blob/download 経由（Supabase 認証＋JWT＋1回限り）
- 機微データは 長期保存を避ける：処理完了後に 定期削除（KV/Blob）を推奨
- ログに個人情報を残さない（ファイル名・医療機関名等をマスク）
- ランタイム：export const runtime = "nodejs"; を API 先頭に明記（fs/pdf-lib 用）
- 地域：preferredRegion = "hnd1"（東京）で低遅延
- タイムアウト：maxDuration を適切に設定（重いPDF時）
- Python スクリプトは Vercel 上で実行しない（ローカル/Cloud Run に分離）
- 料金：Gemini/API ストレージ/転送コストを考慮。大容量PDFの再試行を抑制するリトライ戦略を検討

---

## 12) TODO（今後の拡張予定）
- Blob 自動削除：Cron で「アップロードからN時間でKV/Blobを削除」
- 暗号化保存：クライアント/サーバで暗号化し、鍵は別管理（KMS/DB）
- S3/Supabase Storage への移行（プライベート運用・署名URL/RLS対応）
- キュー/ジョブ管理（重処理をワーカーへ、進捗通知）
- レート制限/WAF：DL API へ IP/ユーザ単位のレート制限
- 監査ログ：チケット発行・DL のトレーサビリティ強化
- e2e テスト：アップロード→分割→チケット→DL の自動試験

---

## 13) 開発者向けメモ
### 13.1 API の共通設定（各 route.ts の先頭推奨）
```ts
// 例：app/api/xxx/route.ts の冒頭
export const runtime = "nodejs"
export const preferredRegion = "hnd1"
export const maxDuration = 60
export const dynamic = "force-dynamic"
```

### 13.2 ダウンロードURLの検証手順（ローカル）
1. ログイン（Supabase）後、/upload でファイル送信
2. 任意の partId で「一時リンク発行」→ URL をコピー
3. そのURLをブラウザで開く → 1回だけDL成功
4. 同じURLをもう一度開く → 410 Gone を期待（GETDEL 消費確認）

### 13.3 簡易 cURL 例（開発検証）
```bash
# チケット発行（Cookieが必要：ログイン後のブラウザで動作確認推奨）
curl -X POST https://<your-domain>/api/blob/ticket \
  -H "Content-Type: application/json" \
  -d '{"partId":"<PART_ID>"}'

# 返ってきた downloadUrl にアクセス（1回のみ有効）
curl -L "<downloadUrl>" -o out.pdf
```

### 13.4 エラー時のヒント
- 401：Supabase の Cookie が見えていない → @supabase/ssr の cookies 設定を確認
- 410：チケットが失効済み/消費済み → 有効期限切れ or 既にGET
- 500：Blob 取得失敗 → BLOB_READ_WRITE_TOKEN/権限・URL誤りを確認

### 13.5 バージョン目安
- Node.js 18+ / Next.js 14+ 推奨
- @google/generative-ai 最新安定版
- pdf-lib 最新安定版

---

## 付録：主な API の入出力（抜粋）
### /api/upload (POST)
Body
```json
{ "pdfUrl": "https://blob.vercel-storage.com/...", "excelUrl": "https://blob.vercel-storage.com/..." }
```
Res
```json
{
  "message": "アップロードと分割が完了しました。",
  "fileGroupId": "8c7a1c2e-....",
  "parts": [
    { "id": "b4b3...", "filename": "file_part_01.pdf", "sizeMB": 12.34 }
  ],
  "excelId": "3f71-..."
}
```

### /api/blob/ticket (POST, 認証必須)
Body
```json
{ "partId": "b4b3..." }
```
Res
```json
{ "downloadUrl": "https://your-app.vercel.app/api/blob/download?token=..." }
```

### /api/blob/download (GET, 認証必須・1回限り)
Query
```rudy
?token=...
```
Res
PDF/Excel のストリーム（Content-Disposition: attachment）

---

この README は、現行の「Vercel 最適化＋安全な一時DL（10分・1回限り）」の設計に基づいています。
運用要件（保存期間・監査・暗号化など）に合わせ、TODOの拡張をご検討ください。
::contentReference[oaicite:0]{index=0}
