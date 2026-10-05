# EC セキュリティ 最低限リスト

対象: React / Cloudflare Workers / D1 / R2 / Stripe の小規模EC
原則: **価格・在庫・割引・注文・決済結果・認証・権限はすべてサーバー側で決める**

---

## 1. 通信・Cookie

- [ ] http → https にリダイレクトされる
- [ ] セッションCookieに `HttpOnly; Secure; SameSite=Lax`（以上）
- [ ] `document.cookie` からセッション値が読めない

## 2. 認証・セッション

- [ ] パスワードは Argon2 / bcrypt / scrypt でハッシュ化（平文なし）
- [ ] **本番で**ハッシュ処理が Workers の CPU 制限に収まる（実測）
- [ ] セッションはサーバー側(D1)管理。ログイン時に再発行、ログアウトで無効化
- [ ] 絶対期限・アイドル期限あり。期限切れは拒否
- [ ] パスワード変更/リセットで全セッション破棄
- [ ] ログイン失敗文面でメールの有無が分からない
- [ ] パスワードリセットトークン: 期限あり・1回限り・長いランダム値
- [ ] 管理者アカウントは2段階認証必須

## 3. 認可（API側で毎回）

- [ ] 未ログインで `/orders/:id` `/admin/*` → 401/403
- [ ] 他人の注文ID → 403/404（`WHERE id=? AND user_id=?`）
- [ ] 他人の注文のキャンセル/変更 → 拒否
- [ ] 一般ユーザーで管理API → 拒否
- [ ] リクエストJSONをそのままUPDATEしない（`role` `price` 等の書き換え不可）



## 4. 入力・出力

- [ ] SQLは全て `prepare().bind()`。`' OR '1'='1` で通らない・SQLエラーも出ない
- [ ] `<script>` を保存・表示しても実行されない（`dangerouslySetInnerHTML` 不使用）
- [ ] 入力の型・長さ・範囲を検証（数量の負数・巨大値は拒否）
- [ ] 本番エラー応答にSQL・stack trace・内部パス・Secretが無い



## 5. CSRF・CORS・ヘッダー

- [ ] 住所/メール/パスワード変更・注文キャンセルが `Origin: https://evil.example` で拒否される
- [ ] `Access-Control-Allow-Origin: *` を認証APIで使っていない
- [ ] CSP / `X-Content-Type-Options: nosniff` / Referrer-Policy / Permissions-Policy / HSTS が返る



## 6. Rate Limit

- [ ] `/login` `/signup` `/password-reset` `/checkout` `/contact` が連打で429
- [ ] 同一アカウントを別IPから失敗 → ロック
- [ ] 実装は Cloudflare Rate Limiting Rules か Durable Objects（Workersのメモリカウンタは無効）



## 7. 金額・クーポン・在庫

- [ ] `price` を書き換えて送っても、決済額はDB価格から再計算
- [ ] 送料・割引・税もサーバーで確定。端数処理のルール統一
- [ ] 期限切れ・対象外・上限超過のクーポン → 拒否
- [ ] 在庫は条件付きUPDATE（`SET stock = stock - ? WHERE id=? AND stock >= ?`）、0件なら拒否
- [ ] stock=1 に同時2注文 → 成功1件、マイナスにならない
- [ ] 在庫確保 → Checkout Session作成の順。確保の自動解放あり



## 8. Stripe

- [ ] Checkout / Elements を使用。カード番号は自社DB・ログに無い
- [ ] Secret Key / Webhook Secret はWorker側のみ（`wrangler secret put`）
- [ ] Webhook署名検証あり。改ざん → 400、5分超の古い通知 → 拒否
- [ ] `event.id` を保存し、再送で二重処理しない（支払済み化・メール・在庫）
- [ ] `UPDATE orders SET status='paid' WHERE id=? AND status='pending'`（1件成功時のみメール）
- [ ] DB更新に失敗したらWebhookに成功を返さない（Stripeの再送で復旧）
- [ ] `/success` を開くだけでは支払済みにならない（Stripe APIで確認）
- [ ] 在庫解放: `checkout.session.expired` と未払いの `async_payment_failed` のみ
- [ ] `payment_intent.payment_failed` では在庫を戻さない
- [ ] `completed` かつ `payment_status=unpaid` は支払済みにも在庫解放にもしない
- [ ] 返金は管理APIからStripe返金APIを呼ぶ。超過返金・二重返金の防止



## 9. R2・アップロード

- [ ] サイズ制限・MIME/拡張子確認・ランダムなファイル名。SVG・実行ファイルは拒否
- [ ] 非公開R2は公開URL無効。未ログイン/他人のアクセス → 拒否



## 10. Secret・依存・ログ

- [ ] Gitの現在・履歴にSecretが無い（流出したらキー再発行）
- [ ] `.env` `.dev.vars` が `.gitignore` 済み
- [ ] 本番/ステージング/開発でD1・R2・Stripeキーを分離
- [ ] ログにパスワード・カード・Cookie・セッションID・Authorization・Secretが無い
- [ ] `npm audit` に重大な脆弱性なし。lockfileコミット、CIは `npm ci`



## 11. 法務（国内）

- [ ] 特商法表記とプライバシーポリシーが購入前画面から辿れる
- [ ] 価格は税込総額表示
- [ ] 最終確認画面で数量・金額・支払方法・引渡時期・返品条件が見え、修正できる
- [ ] 返品条件を明記（不可ならその旨）
- [ ] 最終的な記載内容は消費者庁資料等で確認



## 12. 運用

- [ ] D1バックアップ/復元可能期間を確認し、検証環境への復旧を1回実施
- [ ] 本番デプロイ・ロールバック・マイグレーション手順を文書化
- [ ] 送信メールのSPF / DKIM / DMARC設定
- [ ] Stripe・Cloudflare・ドメインの名義と権限の引き渡し方針を決定
- [ ] 「実施済み／対象外」対応表と確認結果の証跡（スクショ・ログ）を納品物に含める
- [ ] 保守範囲・不具合対応期間・免責を契約に明記

---



## 別オプ

Turnstile / WAF Managed Rules / 監査ログ / メール認証 / Stripe Radar設定 / 外部脆弱性診断 / 保守契約