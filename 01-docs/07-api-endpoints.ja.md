# 07. API エンドポイント

## 1. 文書の目的

この文書は、現在までに確定した文書を基準として、**MVP 第1段階の API エンドポイントと認証/認可ルール**を整理した基準文書である。

基準文書:

- `01-product-scope.md`
- `02-runtime-architecture-and-connection.md`
- `03-role-permission-and-client-flow.md`
- `04-qr-and-attendance-policy.md`
- `05-domain-rules.md`
- `06-database-schema.md`

この文書では次を扱う。

- 認証/認可方式
- 共通のリクエスト/レスポンスルール
- リソース別 API エンドポイント
- 権限範囲

ユーザーフローは `08-workflows.md` を基準とする。

---

## 2. API 基本ルール

### 2-1. Base URL

```text
http://<master-ip>:40343/v1
```

同じデバイスから接続する場合の例:

```text
http://127.0.0.1:40343/v1
```

### 2-2. Content-Type

基本的なリクエスト/レスポンスには JSON を使用する。

```http
Content-Type: application/json
Accept: application/json
```

### 2-3. 時刻基準

- すべてのサーバー検証基準時刻は **マスターサーバーのローカル時刻** を使用する。
- 出席 QR 検証の `TIME_SLOT` 基準もマスターサーバーのローカル時刻基準である。
- `teacher_auth_token` の失効基準もマスターサーバーのローカル日付基準である。

---

## 3. 認証 / 認可モデル

### 3-1. 認証手段の区分

#### A. 接続トークン (`server_connect_token`)
- teacher が **初回ログイン** する際にのみ使用
- サーバー接続およびログイン開始用
- ログイン後の一般 API リクエストでは再使用しない

#### B. teacher 認証トークン (`teacher_auth_token`)
- teacher のログイン成功後にサーバーが発行
- 以後の teacher 関連 API リクエストは `Authorization` ヘッダーで送信
- teacher 機能、スキャンモード、管理機能へのアクセスに使用
- マスターサーバーのローカル日付が変わると無効化される

#### C. student 初期設定用ログイン
- student は常時サーバー接続の主体ではない
- student デバイスは **初回ログイン / 初回 QR 生成情報の確保時点** にのみサーバーと通信する
- 以後、student QR はローカル生成可能

### 3-2. Authorization ヘッダー

teacher 関連リクエストでは以下のヘッダーを使用する。

```http
Authorization: Bearer <teacher_auth_token>
```

### 3-3. teacher_auth_token 無効化条件

次の場合、`teacher_auth_token` は無効化される。

- マスターサーバーのローカル日付変更
- teacher のログアウト
- teacher のパスワード変更
- `teachers.is_active = false`
- マスターサーバーの再起動

日付が変わった場合は即時に接続を切るのではなく、**次回の teacher API リクエスト時点で検証失敗** として処理する。

### 3-4. 権限要約

#### student
- 自分の QR を表示
- 初回ログイン / 初回 QR 生成情報の確保
- 管理機能なし

#### non-admin teacher
- 出席処理
- 一部参照
- 自分のアカウント情報修正
- student アカウント情報修正
- inquiry 作成
- `response IS NULL` の inquiry に対する初回回答作成
- 自分が最後に修正した inquiry response の修正
- inquiry status 変更
- 出席 note の作成/修正（自分が最後に修正した note のみ）

#### admin teacher
- teacher の全権限
- teacher / student / class / payroll / audit_logs を含む全権限
- 状態/権限変更可能
- audit_logs 参照可能

---

## 4. 共通レスポンス形式

### 4-1. 成功レスポンス

```json
{
  "ok": true,
  "data": {}
}
```

### 4-2. 失敗レスポンス

```json
{
  "ok": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "authentication required"
  }
}
```

---

## 5. 共通エラールール

### 400 Bad Request
- 必須フィールド欠落
- 不正な形式
- 許可されていない状態値
- 不正な QR payload 形式

メッセージ例:

```text
invalid request
```

### 401 Unauthorized
- 接続トークンエラー
- teacher login_id / password エラー
- `teacher_auth_token` がない
- `teacher_auth_token` が失効または無効

メッセージ例:

```text
authentication required
invalid credentials
invalid token
token expired
```

### 403 Forbidden
- `is_active = false`
- 権限不足
- non-admin teacher による禁止された修正リクエスト

メッセージ例:

```text
forbidden
insufficient permission
inactive account
```

### 404 Not Found
- 対象 teacher/student/class/inquiry/payroll が存在しない

メッセージ例:

```text
not found
```

### 409 Conflict
- UNIQUE 衝突
- 重複作成
- 同一期間の payroll 重複
- 出席ルール違反（例: teacher の当日3回目のスキャン）

メッセージ例:

```text
conflict
duplicate resource
attendance conflict
```

---

## 6. 共通ページング / ソートルール

一覧 API では次の query parameter を使用できる。

```text
page
page_size
sort
order
q
status
is_active
from
to
```

例:

```text
GET /v1/students?page=1&page_size=20&q=kim
```

レスポンス例:

```json
{
  "ok": true,
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100
    }
  }
}
```

---

## 7. 認証 / 接続 API

### 7-1. POST `/v1/auth/teacher/login`

説明:
- teacher の初回ログイン
- 接続トークンは **この時点でのみ使用**

リクエスト:

```json
{
  "server_connect_token": "random-generated-token",
  "login_id": "teacher01",
  "password": "plain-password"
}
```

レスポンス:

```json
{
  "ok": true,
  "data": {
    "teacher_auth_token": "token-value",
    "expires_on_server_date": "2026-03-09",
    "teacher": {
      "id": 1,
      "teacher_code": "T0001",
      "login_id": "teacher01",
      "name": "Park Admin",
      "is_admin": true,
      "is_active": true
    },
    "server": {
      "port": 40343
    }
  }
}
```

備考:
- teacher 端末は、以後のローカル QR 生成のために `teacher_code`, `login_id`, `port` を保存できる

### 7-2. GET `/v1/auth/teacher/me`

ヘッダー:

```http
Authorization: Bearer <teacher_auth_token>
```

レスポンス:

```json
{
  "ok": true,
  "data": {
    "id": 1,
    "teacher_code": "T0001",
    "login_id": "teacher01",
    "name": "Park Admin",
    "is_admin": true,
    "is_active": true
  }
}
```

### 7-3. POST `/v1/auth/teacher/logout`

ヘッダー:

```http
Authorization: Bearer <teacher_auth_token>
```

レスポンス:

```json
{
  "ok": true,
  "data": {
    "logged_out": true
  }
}
```

備考:
- サーバーは該当 `teacher_auth_token` を無効化する
- クライアントはローカル保存された `teacher_auth_token` を削除しなければならない

### 7-4. POST `/v1/auth/student/setup`

説明:
- student 端末の初回ログイン
- student QR のローカル生成に必要な最小情報を確保するため

リクエスト:

```json
{
  "login_id": "student01",
  "password": "plain-password"
}
```

レスポンス:

```json
{
  "ok": true,
  "data": {
    "student": {
      "id": 10,
      "student_code": "S0001",
      "login_id": "student01",
      "name": "Kim"
    },
    "server": {
      "port": 40343
    }
  }
}
```

備考:
- student 端末は、以後のローカル QR 生成のために `student_code`, `login_id`, `port` を保存できる
- student は常時サーバーログイン状態を維持しない

### 7-5. GET `/v1/server/connection-qr`

ヘッダー:

```http
Authorization: Bearer <teacher_auth_token>
```

レスポンス:

```json
{
  "ok": true,
  "data": {
    "server_url": "http://192.168.0.10:40343",
    "server_connect_token": "random-generated-token",
    "qr_payload": "SERVER:http://192.168.0.10:40343\nTOKEN:random-generated-token"
  }
}
```

説明:
- 既にログイン済みの teacher 端末、またはマスターアプリが新しい teacher デバイス接続用 QR を表示する際に使用

---

## 8. Teacher API

### 8-1. GET `/v1/teachers`

権限:
- admin only

### 8-2. GET `/v1/teachers/{teacher_id}`

権限:
- admin only

### 8-3. POST `/v1/teachers`

権限:
- admin only

リクエスト例:

```json
{
  "teacher_code": "T0003",
  "login_id": "teacher03",
  "name": "Lee",
  "email": "lee@example.com",
  "phone": "010-0000-0000",
  "emergency_contact": "010-9999-9999",
  "password": "plain-password",
  "is_admin": false
}
```

備考:
- 保存時、password は `password_hash` として反映される
- `audit_logs` 記録

### 8-4. PATCH `/v1/teachers/{teacher_id}/status`

権限:
- admin only

リクエスト例:

```json
{
  "is_active": true,
  "is_admin": false,
  "reason": "権限調整"
}
```

備考:
- `audit_logs` 記録
- 状態/権限変更専用 API

### 8-5. PATCH `/v1/teachers/me/profile`

権限:
- ログイン済み teacher

リクエスト例:

```json
{
  "login_id": "teacher01",
  "password": "new-password",
  "name": "Park Admin",
  "email": "park@example.com",
  "phone": "010-1111-1111",
  "emergency_contact": "010-2222-2222",
  "reason": "個人情報修正"
}
```

許可フィールド:
- `login_id`
- `password`
- `name`
- `email`
- `phone`
- `emergency_contact`

備考:
- `login_id` 変更時、該当端末では QR ローカル生成情報の再同期が必要になる場合がある
- `audit_logs` 記録

---

## 9. Student API

### 9-1. GET `/v1/students`

権限:
- teacher 以上

### 9-2. GET `/v1/students/{student_id}`

権限:
- teacher 以上

レスポンスには必要に応じて関連 teacher / class 情報を含めることができる

### 9-3. POST `/v1/students`

権限:
- admin only

リクエスト例:

```json
{
  "student_code": "S0001",
  "login_id": "student01",
  "name": "Kim",
  "email": "kim@example.com",
  "guardian_email": "parent@example.com",
  "phone": "010-3333-3333",
  "guardian_phone": "010-4444-4444",
  "password": "plain-password"
}
```

備考:
- `audit_logs` 記録

### 9-4. PATCH `/v1/students/{student_id}/profile`

権限:
- teacher 以上
- ただし、修正範囲は teacher に許可された範囲に限定される

許可フィールド:
- `login_id`
- `password`
- `name`
- `email`
- `guardian_email`
- `phone`
- `guardian_phone`

リクエスト例:

```json
{
  "login_id": "student01",
  "name": "Kim Min",
  "guardian_phone": "010-5555-5555",
  "reason": "生徒情報修正"
}
```

備考:
- `login_id` 変更時、student 端末では再ログインまたは再同期が必要になる可能性がある
- `audit_logs` 記録

### 9-5. PATCH `/v1/students/{student_id}/status`

権限:
- admin only

リクエスト例:

```json
{
  "is_active": false,
  "reason": "休塾処理"
}
```

備考:
- `audit_logs` 記録

---

## 10. Class API

### 10-1. GET `/v1/classes`

権限:
- teacher 以上

### 10-2. GET `/v1/classes/{class_id}`

権限:
- teacher 以上

### 10-3. POST `/v1/classes`

権限:
- admin only

リクエスト例:

```json
{
  "class_code": "C0001",
  "teacher_id": 1,
  "name": "小学生英語 A",
  "tuition_fee": 200000,
  "reason": "新規講座開設"
}
```

### 10-4. PATCH `/v1/classes/{class_id}`

権限:
- admin only

許可例:
- 担当 teacher 変更
- 講座名変更
- 受講料変更
- 有効/無効処理

### 10-5. POST `/v1/student-classes`

権限:
- admin only

リクエスト例:

```json
{
  "student_id": 10,
  "class_id": 3,
  "status": "ACTIVE",
  "enrolled_at": "2026-03-09T10:00:00",
  "reason": "受講登録"
}
```

### 10-6. PATCH `/v1/student-classes/{id}`

権限:
- admin only

リクエスト例:

```json
{
  "status": "PAUSED",
  "ended_at": null,
  "reason": "一時停止"
}
```

---

## 11. Attendance API

### 11-1. POST `/v1/attendance/scan`

権限:
- ログイン済み teacher only

説明:
- student 出席
- teacher 出勤/退勤
- いずれもこの API で統合処理する

リクエスト例:

```json
{
  "qr_payload": "kim01|S0001|29384721|40343"
}
```

サーバー処理順序:
1. QR パース
2. code prefix で student / teacher を区分
3. 該当テーブルで code を照会
4. row の `login_id` と QR の `login_id` が一致するか確認
5. QR の `time_slot` と現在のマスターサーバー `time_slot` が一致するか確認
6. `port` 値が現在のサーバーポートと一致するか確認
7. student なら `student_attendance` を生成
8. teacher なら当日の1回目スキャン/2回目スキャンルールに従って `teacher_attendance` を生成または更新
9. `audit_logs` 記録

student レスポンス例:

```json
{
  "ok": true,
  "data": {
    "scan_type": "STUDENT_ATTENDANCE",
    "attendance_id": 101,
    "student_id": 10,
    "scanned_at": "2026-03-09T09:00:00"
  }
}
```

teacher レスポンス例:

```json
{
  "ok": true,
  "data": {
    "scan_type": "TEACHER_CHECKIN",
    "attendance_id": 55,
    "teacher_id": 3,
    "checkin_at": "2026-03-09T08:59:00"
  }
}
```

または:

```json
{
  "ok": true,
  "data": {
    "scan_type": "TEACHER_CHECKOUT",
    "attendance_id": 55,
    "teacher_id": 3,
    "checkout_at": "2026-03-09T18:05:00"
  }
}
```

エラー例:
- teacher の当日3回目スキャン → `409 Conflict`
- student の5分以内再スキャン → `409 Conflict`

### 11-2. GET `/v1/student-attendance`

権限:
- teacher 以上

フィルタ例:

```text
student_id
from
to
```

### 11-3. GET `/v1/teacher-attendance`

権限:
- teacher 以上
- 一般 teacher は範囲が制限される場合がある
- admin は全件参照可能

フィルタ例:

```text
teacher_id
from
to
missing_checkout=true
```

### 11-4. PATCH `/v1/attendance/{attendance_type}/{attendance_id}/note`

説明:
- `attendance_type` は `student` または `teacher`

権限:
- teacher 以上
- non-admin teacher は自分が最後に修正した note のみ修正可能

リクエスト例:

```json
{
  "note": "遅刻理由確認",
  "reason": "出席 note 修正"
}
```

備考:
- `note_editor_teacher_id` 更新
- `audit_logs` 記録

---

## 12. Inquiry API

### 12-1. GET `/v1/inquiries`

権限:
- teacher 以上

フィルタ例:

```text
student_id
author_teacher_id
status
from
to
```

### 12-2. GET `/v1/inquiries/{inquiry_id}`

権限:
- teacher 以上

### 12-3. POST `/v1/inquiries`

権限:
- teacher 以上

リクエスト例:

```json
{
  "student_id": 10,
  "content": "保護者問い合わせ: 来月の時間割変更希望"
}
```

備考:
- `author_teacher_id`, snapshot 値を保存
- `audit_logs` 記録

### 12-4. POST `/v1/inquiries/{inquiry_id}/response`

権限:
- teacher 以上
- non-admin teacher でも可
- ただし、`response IS NULL` の場合の初回回答入力のみ許可

リクエスト例:

```json
{
  "response": "来月から水曜日クラスへ移動可能です。",
  "reason": "問い合わせ初回回答"
}
```

### 12-5. PATCH `/v1/inquiries/{inquiry_id}/response`

権限:
- teacher 以上
- non-admin teacher は自分が最後に修正した response のみ修正可能

リクエスト例:

```json
{
  "response": "来月第2週から水曜日クラスへ移動可能です。",
  "reason": "回答修正"
}
```

備考:
- `response_teacher_id` は現在の response の最終修正者
- `audit_logs` 記録

### 12-6. PATCH `/v1/inquiries/{inquiry_id}/status`

権限:
- teacher 以上

リクエスト例:

```json
{
  "status": "IN_PROGRESS",
  "reason": "処理開始"
}
```

制約:
- non-admin teacher は `CLOSED -> OPEN`
- non-admin teacher は `CLOSED -> IN_PROGRESS`
- 上記 reopen は admin のみ可能

---

## 13. Payroll API

### 13-1. GET `/v1/payrolls`

権限:
- admin only

### 13-2. GET `/v1/payrolls/{payroll_id}`

権限:
- admin only

### 13-3. POST `/v1/payrolls`

権限:
- admin only

リクエスト例:

```json
{
  "teacher_id": 1,
  "period_start": "2026-03-01",
  "period_end": "2026-03-31",
  "worked_minutes": 9600,
  "monthly_salary": 2500000,
  "bonus_amount": 100000,
  "deduction_amount": 50000,
  "final_amount": 2550000,
  "status": "DRAFT",
  "reason": "3月給与作成"
}
```

備考:
- 同一 teacher + 同一期間は 1 件のみ許可

### 13-4. PATCH `/v1/payrolls/{payroll_id}`

権限:
- admin only

### 13-5. PATCH `/v1/payrolls/{payroll_id}/status`

権限:
- admin only

状態:
- `DRAFT`
- `CONFIRMED`
- `PAID`
- `CANCELLED`

リクエスト例:

```json
{
  "status": "PAID",
  "paid_at": "2026-03-31T18:00:00",
  "reason": "給与支給完了"
}
```

---

## 14. Audit Log API

### 14-1. GET `/v1/audit-logs`

権限:
- admin only

フィルタ例:

```text
target_table
target_id
editor_teacher_id
action_type
from
to
```

### 14-2. GET `/v1/audit-logs/{log_id}`

権限:
- admin only