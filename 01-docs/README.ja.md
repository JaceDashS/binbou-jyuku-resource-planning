# Academy Integrated Management System Docs

## ドキュメント一覧

1. [`01-product-scope.ja.md`](./01-product-scope.ja.md)
   - 背景
   - 目標
   - 基本方針
   - MVP 範囲
   - 除外範囲
   - 拡張候補
   - 優先順位

2. [`02-runtime-architecture-and-connection.ja.md`](./02-runtime-architecture-and-connection.ja.md)
   - マスターアプリ / クライアントアプリ構成
   - SQLite 運用方針
   - ポート規則
   - 接続トークンと teacher 認証トークン規則
   - サーバー接続 QR
   - teacher の初回ログインおよび再ログイン方式
   - WiFi 再接続時の処理方針

3. [`03-role-permission-and-client-flow.ja.md`](./03-role-permission-and-client-flow.ja.md)
   - アカウント種別
   - ログイン規則
   - 権限モデル
   - アカウント修正範囲
   - クライアントアプリ UI フロー
   - 管理モード分岐

4. [`04-qr-and-attendance-policy.ja.md`](./04-qr-and-attendance-policy.ja.md)
   - QR の種類
   - 表示モード / スキャンモード
   - サーバー接続 QR
   - student / teacher 出席 QR 生成規則
   - teacher 出勤 / 退勤規則
   - student 出席規則
   - 出席 note 規則

5. [`05-domain-rules.ja.md`](./05-domain-rules.ja.md)
   - 業務ルール
   - 監査ログ運用原則
   - 状態値ルール
   - 権限ベース運用ルール
   - 給与ルール
   - 問い合わせルール
   - 出席記録ルール
   - 削除ポリシー

6. [`06-database-schema.ja.md`](./06-database-schema.ja.md)
   - MVP スキーマ概要
   - テーブル一覧
   - SQLite DDL
   - audit_logs 詳細構造
   - インデックス設計
   - 後続実装作業

7. [`07-api-endpoints.ja.md`](./07-api-endpoints.ja.md)
   - 認証 / 認可方式
   - 共通リクエスト / レスポンス規則
   - 共通エラー規則
   - Teacher / Student / Class / Attendance / Inquiry / Payroll / Audit Log API

8. [`08-workflows.ja.md`](./08-workflows.ja.md)
   - teacher 初回ログインフロー
   - student 初回設定フロー
   - student / teacher QR 表示フロー
   - 出席 / 出勤 / 退勤処理フロー
   - アカウント修正フロー
   - 問い合わせ / 給与 / 監査ログ処理フロー
   - 日付変更後の再ログインフロー

---

## 現在の主要設計要約

### 1. システム構成

- システムは **マスターアプリ + クライアントアプリ** 構成で運用する
- マスターアプリは Go ベースのマスターサーバーと SQLite 原本 DB を含む実行型アプリである
- source of truth は **マスターサーバーが管理する SQLite DB** 一つに維持する
- クライアントアプリはマスターサーバーに接続して QR、出席、管理機能を使用する
- マスターアプリは前面 UI として実行される場合もあれば、バックグラウンドのサーバープロセスとして維持される場合もある
- マスターサーバーとクライアントアプリは同じ端末で同時に動作することも、別の端末に配置されることもある

### 2. teacher ログインおよび認証方式

- teacher は **IP + 接続トークン + teacher login_id + password** で初回ログインする
- ログイン成功時にサーバーは **`teacher_auth_token`** を発行する
- 以後の teacher 関連機能は **`teacher_auth_token` ベース** で利用する
- 接続トークンは **初回ログイン時のみ使用** し、出席処理 / チェックイン / チェックアウト / 管理機能リクエスト時には再使用しない
- サーバーは Cookie ベースのセッションを維持せず、teacher 関連リクエストは `teacher_auth_token` を基準に処理する
- `teacher_auth_token` は **発行時点のマスターサーバーローカル日付まで有効** であり、日付変更後の最初の teacher リクエストで無効処理される
- また、ログアウト、パスワード変更、アカウント無効化、マスターサーバー再起動時にも無効化される

### 3. ネットワーク再接続処理

- 現在のバージョンでは、WiFi 切断後の再接続に対する **自動 fallback / 自動復旧ロジック** は実装しない
- ただし `teacher_auth_token` が有効であれば、ネットワーク復旧後に login_id / password を手動で再入力せずにリクエストを再試行できる
- 自動再接続、未送信リクエストの自動再送、サーバー IP 自動再探索、トークン自動復旧処理は **後続バージョンで検討** する

### 4. QR および出席処理構造

- QR は **サーバー接続 QR** と **出席 QR** に分ける
- サーバー接続 QR は teacher 端末がマスターサーバーに初回ログインする際に必要な **サーバーアドレス + 接続トークン** を確保するためのもの
- 出席 QR は **student または teacher の出席処理用** である

#### 出席 QR 生成規則

- student QR と teacher QR はどちらも **ローカル生成可能**
- student / teacher 端末は初回ログインまたは初回同期後、必要な情報をローカルに保存できる
- 出席 QR は次の形式を使用する

```text
LOGIN_ID|CODE|TIME_SLOT|PORT
```

例:

```text
kim01|S0001|29384721|40343
park02|T0003|29384721|40343
```

- `CODE` prefix により student / teacher を区別する
- `TIME_SLOT` は **60秒単位の再生成基準値**
- `PORT` は現在アプリで使用しているポート番号を意味する

#### 出席 QR 検証規則

サーバーは次の順序で検証する。

1. QR 解析
2. code prefix により student / teacher を区別
3. 対応テーブルで code を照会
4. row の `login_id` と QR の `login_id` が一致するか確認
5. QR の `time_slot` と現在のマスターサーバー `time_slot` が一致するか確認
6. `port` 値が現在のサーバーポートと一致するか確認
7. すべて一致した場合に有効と処理

### 5. teacher 端末利用原則

- **teacher QR 表示** は、過去に一度ログインして必要なローカル情報を確保した端末であれば、現在のログイン状態に関係なく可能
- 一方で **スキャンモード、出席処理、チェックイン、チェックアウト、管理機能リクエスト** は、ログイン済みの teacher 端末でのみ可能
- つまり **QR を見せること** と **QR を読み取ってサーバーに記録すること** は区別する
- teacher の出勤 / 退勤は、teacher QR を別のログイン済み teacher 端末がスキャンする方式で処理する

### 6. 権限モデル要約

- アカウント種別は `TEACHER`, `STUDENT`
- 別途 staff アカウントは置かない
- `teachers.is_admin` により管理者権限を区別する
- Student は QR 中心の最小機能のみ使用する
- Teacher は QR 画面 + 管理モードボタンを持つ
- non-admin teacher は限定された例外機能のみ使用可能
- admin teacher は全管理機能を使用可能

#### non-admin teacher 例外権限要約

- 本人アカウント情報修正
- student アカウント情報修正
- inquiry 作成
- 未回答 inquiry への初回回答入力
- 自分が最後に修正した inquiry response の修正
- inquiry status 変更
- 出席 note 作成
- 自分が最後に修正した出席 note の修正

### 7. ドメインルール要約

- 一般的なマスターデータ変更は **admin teacher のみ可能**
- `audit_logs` は主要な生成 / 修正 / 削除だけでなくシステム性変更行為まで記録する
- inquiry は学生基準で接続される問い合わせデータとして統合管理する
- payroll は月給制基準で運用し、同一 teacher + 同一期間の給与は 1件のみ許可する
- 削除は **soft delete 方式** を使用する
- 基本照会は `deleted_at IS NULL` 条件を基準に処理する

### 8. DB および監査ログ

- DB は SQLite ベースのローカル DB として運用する
- マスターサーバーのみが DB に直接アクセス可能
- システムの主要データ修正は `audit_logs` に記録する
- 現在状態は本テーブルに保存し、変更前 / 後の状態は `audit_logs.before_data / after_data` snapshot として保存する
- soft delete ポリシーを使用し、基本照会は `deleted_at IS NULL` 条件を基準に処理する

### 9. API 設計要約

- API base URL は `http://<master-ip>:40343/v1`
- teacher 関連リクエストは **Authorization ヘッダーの `teacher_auth_token`** を使用する
- 接続トークンは teacher 初回ログイン時のみ使用する
- student は常時サーバー接続主体ではなく、初回設定後はローカル QR 生成中心で動作する
- 出席処理 API は student 出席と teacher 出勤 / 退勤を一つのスキャン API に統合して処理する
- 一覧 API は基本的な paging / filter / sort query parameter を使用できる

### 10. ワークフロー文書分離理由

- エンドポイント定義とユーザーフローは常に一緒に変わるとは限らないため、別文書に分離する
- `07-api-endpoints.ja.md` は **サーバーインターフェース定義**
- `08-workflows.ja.md` は **ユーザー / システム動作フロー定義**
- この分離により、API 変更と UX フロー変更を独立して検討できる

---

## 分割基準

- 一緒によく変わる内容同士をまとめる
- 接続 / 認証 / サーバー接続 QR は一つのランタイム接続文書として管理する
- 権限 / UI フローは一つの役割文書として管理する
- 出席用 QR と出席ポリシーは一つの出席文書として管理する
- ドメインルールと DB スキーマは分離して管理する
- API エンドポイントとユーザーワークフローは別文書として管理する

---

## 現在残っている設計メモ

- ポート変更時の QR 再同期規則は **MVP 1次ではまだ扱わない**
- `login_id` 変更時には、その端末の既存ローカル QR 生成情報が最新状態ではなくなる可能性があるため、再ログインまたは再同期が必要になる場合がある
- 現在の QR 生成規則は **MVP 1次向けの単純検証構造** であり、強い偽造防止を保証する秘密署名構造ではない
- ポート番号は QR 生成規則の構成要素として使用するが、強い秘密値としては扱わない
- 後続バージョンでは seed, HMAC, 署名ベース方式によるセキュリティ強化を検討できる

---

## 文書利用原則

- 製品範囲変更は [`01-product-scope.ja.md`](./01-product-scope.ja.md) を優先修正
- 接続 / 認証方式変更は [`02-runtime-architecture-and-connection.ja.md`](./02-runtime-architecture-and-connection.ja.md) を優先修正
- 権限 / UI フロー変更は [`03-role-permission-and-client-flow.ja.md`](./03-role-permission-and-client-flow.ja.md) を優先修正
- QR / 出席ポリシー変更は [`04-qr-and-attendance-policy.ja.md`](./04-qr-and-attendance-policy.ja.md) を優先修正
- 業務ルール変更は [`05-domain-rules.ja.md`](./05-domain-rules.ja.md) を優先修正
- DB カラム / DDL 変更は [`06-database-schema.ja.md`](./06-database-schema.ja.md) を優先修正
- エンドポイント定義変更は [`07-api-endpoints.ja.md`](./07-api-endpoints.ja.md) を優先修正
- ユーザー / システムフロー変更は [`08-workflows.ja.md`](./08-workflows.ja.md) を優先修正