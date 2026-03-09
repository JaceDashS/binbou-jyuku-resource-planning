# 08. ワークフロー

## 1. 文書の目的

この文書は、現時点で確定している文書を基準として、**MVP 1次の主要ユーザーフローとシステム処理フロー**を整理した基準文書である。

基準文書:

- `01-product-scope.md`
- `02-runtime-architecture-and-connection.md`
- `03-role-permission-and-client-flow.md`
- `04-qr-and-attendance-policy.md`
- `05-domain-rules.md`
- `06-database-schema.md`
- `07-api-endpoints.md`

この文書では、次を扱う。

- teacher 初回ログインフロー
- student 初期設定フロー
- teacher / student QR 利用フロー
- 出席 / 出勤 / 退勤 処理フロー
- アカウント修正、問い合わせ、給与処理フロー
- 日付変更後の再ログインフロー

---

## 2. Teacher 初回ログイン - 手動入力方式

```text
teacher アプリ起動
→ サーバーIP入力
→ 接続トークン入力
→ login_id 入力
→ password 入力
→ POST /v1/auth/teacher/login
→ ログイン成功
→ teacher_auth_token 受信
→ ローカルセキュアストレージに teacher_auth_token 保存
→ teacher_code / login_id / port 保存
→ 基本QR画面に遷移
````

---

## 3. Teacher 初回ログイン - サーバー接続QR方式

```text
既にログイン済みの teacher 端末またはマスターアプリでサーバー接続QR表示
→ 新しい teacher 端末がQRスキャン
→ server_url + server_connect_token 自動入力
→ login_id 入力
→ password 入力
→ POST /v1/auth/teacher/login
→ teacher_auth_token 受信
→ 基本QR画面に遷移
```

---

## 4. Teacher 日付変更後の再ログイン

```text
既存の teacher_auth_token でリクエスト
→ サーバーが日付変更を検知
→ 401 Unauthorized / token expired
→ teacher が再度ログイン
  - 手動入力、または
  - 保存済み情報の再利用、または
  - 別の teacher 端末のサーバー接続QRをスキャン
→ 新しい teacher_auth_token 発行
→ 以降の機能を継続利用
```

---

## 5. Student 初期設定

```text
student アカウントが teacher/admin によって作成される
→ student 端末で login_id / password 入力
→ POST /v1/auth/student/setup
→ student_code / login_id / port 受信
→ ローカル保存
→ 以後 student QR をローカル生成
```

---

## 6. Student QR 表示フロー

```text
student 端末がローカル保存された login_id / student_code / port を確認
→ 現在の60秒 time_slot を計算
→ LOGIN_ID|CODE|TIME_SLOT|PORT 形式でQR生成
→ student がQR画面を表示
```

---

## 7. Teacher QR 表示フロー

```text
teacher 端末が過去のログイン時に保存した login_id / teacher_code / port を確認
→ 現在の60秒 time_slot を計算
→ LOGIN_ID|CODE|TIME_SLOT|PORT 形式でQR生成
→ teacher が本人QR画面を表示
```

備考:

* teacher QR の表示は現在 `teacher_auth_token` が期限切れの状態でも可能
* ただし、スキャンモードおよびサーバー記録リクエストはログイン済みの teacher 端末でのみ可能

---

## 8. Student 出席処理フロー

```text
student 端末がローカルでQR生成
→ QR形式: LOGIN_ID|CODE|TIME_SLOT|PORT
→ ログイン済みの teacher 端末がスキャンモード実行
→ POST /v1/attendance/scan
→ サーバーがQRをパース
→ code prefix で student を判別
→ code を照会
→ login_id 一致可否確認
→ 現在のマスターサーバー time_slot 一致可否確認
→ port 一致可否確認
→ student_attendance row 作成
→ audit_logs 記録
→ 出席成功レスポンス
```

---

## 9. Teacher 出勤処理フロー

```text
teacher A が本人QR表示
→ teacher B のログイン済み端末がスキャンモード実行
→ POST /v1/attendance/scan
→ サーバーが teacher QR を検証
→ 当日の既存記録がないことを確認
→ teacher_attendance row 作成
→ checkin_at 保存
→ audit_logs 記録
→ 出勤成功レスポンス
```

---

## 10. Teacher 退勤処理フロー

```text
teacher A が本人QR表示
→ teacher B のログイン済み端末がスキャンモード実行
→ POST /v1/attendance/scan
→ サーバーが teacher QR を検証
→ 当日の既存 checkin row が存在することを確認
→ checkout_at 更新
→ audit_logs 記録
→ 退勤成功レスポンス
```

---

## 11. Teacher 当日3回目スキャン処理フロー

```text
teacher QR スキャン
→ サーバーが既存の当日 teacher_attendance row を確認
→ 既に checkin_at / checkout_at が両方存在
→ 当日3回目のスキャンと判断
→ 409 Conflict 返却
→ 追加の出退勤記録は生成しない
```

---

## 12. Student 5分以内再スキャン処理フロー

```text
student QR スキャン
→ サーバーが直近の student_attendance を確認
→ 同じ student の直近 scanned_at が5分以内か検査
→ 5分以内なら重複スキャンと判断
→ 409 Conflict 返却
→ 新しい出席 row は生成しない
```

---

## 13. Teacher 本人アカウント修正フロー

```text
teacher ログイン
→ 自分のプロフィール画面に遷移
→ 修正可能フィールド入力
→ PATCH /v1/teachers/me/profile
→ サーバーが許可フィールドのみ検証
→ teachers 更新
→ audit_logs 記録
→ 成功レスポンス
```

備考:

* `login_id` 変更時、当該端末のQRローカル生成情報は再同期が必要になる可能性がある

---

## 14. Student 情報修正フロー

```text
teacher ログイン
→ student 詳細照会
→ 修正可能フィールドのみ編集
→ PATCH /v1/students/{student_id}/profile
→ サーバーが権限および許可フィールド検証
→ students 更新
→ audit_logs 記録
→ 成功レスポンス
```

備考:

* `login_id` 変更時、student 端末は再ログインまたは再同期が必要になる可能性がある

---

## 15. Inquiry 作成フロー

```text
teacher ログイン
→ student 選択
→ 問い合わせ内容入力
→ POST /v1/inquiries
→ inquiry 作成
→ author_teacher_id / snapshot 保存
→ audit_logs 記録
→ 成功レスポンス
```

---

## 16. Inquiry 初回回答フロー

```text
teacher ログイン
→ inquiry 詳細に遷移
→ response が NULL か確認
→ 回答入力
→ POST /v1/inquiries/{id}/response
→ response 保存
→ response_teacher_id 更新
→ audit_logs 記録
→ 成功レスポンス
```

---

## 17. Inquiry 回答修正フロー

```text
teacher ログイン
→ inquiry 照会
→ 自身が最後の response 修正者か検査
→ PATCH /v1/inquiries/{id}/response
→ 許可されれば response 修正
→ response_teacher_id 更新
→ audit_logs 記録
→ 成功レスポンス
```

備考:

* non-admin teacher は自身が最後の修正者である場合にのみ可能

---

## 18. Inquiry ステータス変更フロー

```text
teacher ログイン
→ inquiry 一覧 / 詳細照会
→ ステータス変更ボタン選択
→ PATCH /v1/inquiries/{id}/status
→ サーバーが権限検証
→ status 変更
→ audit_logs 記録
→ 成功レスポンス
```

備考:

* non-admin teacher は `CLOSED -> OPEN/IN_PROGRESS` reopen 不可

---

## 19. Payroll 作成および支給処理フロー

```text
admin ログイン
→ 給与対象 teacher 選択
→ 精算期間入力
→ 金額計算および入力
→ POST /v1/payrolls
→ payroll 作成
→ audit_logs 記録

その後
→ 金額確定時 PATCH /v1/payrolls/{id}/status = CONFIRMED
→ 実際の支給後 PATCH /v1/payrolls/{id}/status = PAID
→ paid_at 保存
→ audit_logs 記録
```

---

## 20. Audit Log 照会フロー

```text
admin ログイン
→ audit_logs ページに遷移
→ 対象テーブル / 対象ID / 修正者 / 期間フィルタ入力
→ GET /v1/audit-logs
→ 変更履歴一覧照会
→ 必要に応じて GET /v1/audit-logs/{id} で詳細確認
```

---

## 21. WiFi 切断後の再接続フロー

```text
teacher 端末使用中にWiFi切断
→ リクエスト失敗
→ ネットワーク復旧
→ アプリが teacher_auth_token をそのまま保持している
→ 再度リクエスト試行
→ teacher_auth_token が依然として有効なら正常処理
→ teacher_auth_token が無効なら401返却
→ teacher が再ログイン
```

備考:

* 現在バージョンでは自動再接続 / 自動認証再試行 / 未送信リクエスト自動再送信は実装しない

---

## 22. Teacher ログアウトフロー

```text
teacher ログイン状態
→ ログアウトボタン選択
→ POST /v1/auth/teacher/logout
→ サーバーが teacher_auth_token 無効化
→ クライアントがローカル保存された teacher_auth_token 削除
→ 管理機能 / スキャンモード使用不可
→ 必要に応じて再ログイン
```

---

## 23. 実装メモ

* 出席QRは現在 **MVP 1次向けの単純検証構造**である。
* 現在のQR生成規則は `LOGIN_ID|CODE|TIME_SLOT|PORT` 形式を使用する。
* ポート番号はQR生成規則の構成要素だが、強い秘密値ではない。
* ポート変更時の再同期ルールは **MVP 1次範囲ではまだ扱わない**
* `login_id` 変更時、当該端末は再ログインまたは再同期が必要になる可能性がある。
* セキュリティ高度化(seed, HMAC, signature)は後続バージョンで検討する。