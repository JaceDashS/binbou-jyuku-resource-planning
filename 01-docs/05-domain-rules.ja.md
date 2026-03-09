# 05. ドメインルール

## 1. 文書の目的

この文書は、**業務ドメインルール、監査ログの運用原則、状態値、削除ポリシー、権限ベースの運用ルール**を整理した基準文書である。  
DB DDLとインデックスは `06-database-schema.md`、APIは `07-api-endpoints.md`、ユーザーフローは `08-workflows.md` を基準とする。

---

## 2. 変更履歴（audit log）の概念と運用原則

システムの主要データの修正は、**変更履歴（audit log）** として記録されなければならない。

目的:

- 誰がデータを修正したかを追跡する
- いつ、どのような変更が発生したかを記録する
- 問題発生時に過去の状態を復元または分析できるようにする

### 2-1. 保存および記録原則

- データの **現在状態は本テーブル（students、teachers など）に保存**する。
- 変更が発生するたびに、`audit_logs` テーブルへ1件のログ row を追加する。
- 各ログには、修正前の状態（`before_data`）と修正後の状態（`after_data`）を snapshot 形式で保存する。
- MVP 最終構成では別途 `revision_no` を設けず、`id` と `created_at` の順序を基準に変更履歴を追跡する。
- **データ生成（create）および一般的な作成行為も `audit_logs` に記録する**

### 2-2. 権限および参照原則

- `audit_logs` は修正履歴の参照用であり、いかなるユーザーも直接修正できない
- `audit_logs` は admin のみ参照可能である

### 2-3. 動作方式

1. データ生成（create）
   - 生成時点の状態を `after_data` として記録する
   - `action_type = CREATE`

2. データ修正（update）
   - 既存状態を `before_data` に保存する
   - 修正後の結果状態を `after_data` に保存する
   - `action_type = UPDATE`

3. データ削除（delete）
   - 本システムの削除は **soft delete** であるため、物理削除の代わりに `deleted_at` を記録する
   - 削除直前の有効状態を `before_data` に保存する
   - soft delete が適用され、`deleted_at` が埋められた状態を `after_data` に保存する
   - `action_type = DELETE`

### 2-4. この構造の特徴

- 現在状態の参照は本テーブルで高速に行える
- 変更前後を1つの row で比較できる
- 同じ対象が複数回修正されても、`target_table + target_id + created_at` の順で履歴を追跡できる
- 監査（audit）および運用上の原因分析に適している
- UI では `audit_logs` に記録された変更対象の値を赤色で表示し、ユーザーが変更された値を即座に識別できるようにする

つまり、システムは

**現在状態テーブル + 変更履歴テーブル（`audit_logs`）**

の構造で運用される。

---

## 3. 状態値ルール

### 3-1. `student_classes.status`

- `ACTIVE`: 現在受講中
- `PAUSED`: 一時停止
- `DROPPED`: 中途離脱
- `COMPLETED`: 正常終了

### 3-2. `inquiries.status`

- `OPEN`: 受付済み
- `IN_PROGRESS`: 対応中
- `CLOSED`: クローズ済み
- `HOLD`: 保留

### 3-3. `payrolls.status`

- `DRAFT`: 下書き状態
- `CONFIRMED`: 金額確定・未払い状態
- `PAID`: 支払い完了
- `CANCELLED`: 無効処理

### 3-4. `audit_logs.action_type`

- `CREATE`
- `UPDATE`
- `DELETE`

---

## 4. ログインおよび認証の運用ルール

### 4-1. アカウント識別原則

- メインログイン画面では、まず `teacher` / `student` の種別を選択する
- したがって、`teachers.login_id` と `students.login_id` が互いに同じ値を持つことは許容する
- 各テーブル内部では、active row 基準で UNIQUE でなければならない
- soft delete された row の UNIQUE 値は **再利用可能** でなければならない

### 4-2. teacher 接続およびログイン原則

- teacher がマスターサーバー機能へアクセスする際は、**必ず IP + 接続トークン + teacher login_id + password** で初回ログインしなければならない
- ログイン成功時、サーバーは `teacher_auth_token` を発行する
- 接続トークンは **初回ログイン時にのみ使用**する
- 以後、出席処理、チェックイン、チェックアウト、管理機能のリクエストでは **接続トークンを再使用しない**
- 以後の teacher 機能は、ログイン済み teacher 端末の認証状態を基準に処理する
- teacher 関連リクエストは **Authorization ヘッダーの `teacher_auth_token` 検証**を経る
- `teacher_auth_token` は **発行時点のマスターサーバーのローカル日付まで有効**であり、日付変更後の最初の teacher リクエストで無効化する
- また、teacher のログアウト、パスワード変更、`teachers.is_active = false`、マスターサーバー再起動時にも無効化される
- 同一デバイスから接続する場合でもこの原則は維持し、この場合 `127.0.0.1:40343` を使用できる

### 4-3. 登録および初期アカウント原則

- **会員登録 UI は teacher アカウントに対してのみ提供**する
- 初期インストール時には、**基本 admin teacher アカウント1件をデフォルトで提供**する

### 4-4. 学生アカウント利用原則

- 学生は会員登録 UI を使用しない
- 学生アカウントは teacher または admin が登録する
- 学生デバイスでは、**初回ログイン1回と初回 QR 生成1回のみ**、マスターと通信可能な状態が必要である

---

## 5. 変更権限ルール

### 5-1. 基本権限原則

- 一般的なマスターデータの変更は **admin teacher のみ可能**
- すなわち、`teachers.is_admin = true` のアカウントのみが、学生・教師・授業・給与などの一般修正 API を呼び出せる
- ただし、アカウント/個人情報修正には例外ルールを設ける
- teacher は **自分のアカウントの** `login_id`, `password`, `name`, `email`, `phone`, `emergency_contact` を self-edit 可能である
- student アカウント情報は **teacher 画面を通じてのみ** 修正可能であり、修正範囲は `login_id`, `password`, `name`, `email`, `guardian_email`, `phone`, `guardian_phone` に限定する
- MVP 第1段階では **student 本人が直接 students アカウント情報を修正しない**
- admin アカウントでログインした場合には、**状態/権限変更ページへ移動できるボタン** を表示し、当該ページで teacher の `is_active`, `is_admin` と student の `is_active` を変更できる
- admin は出席 note を含むすべての修正について制限なく処理可能である

### 5-2. non-admin teacher の例外権限

- 例外的に non-admin teacher は、**自分のアカウント情報修正**、**student アカウント情報修正**、**新規 inquiry 作成**、`response` が NULL である inquiry に対する初回回答入力、**自分が最後に修正した inquiry response の修正**、**制限付き inquiry status 変更**、**出席 note 作成**、**自分が最後に修正した出席 note の修正**のみ可能である
- non-admin teacher は `CLOSED` 状態の inquiry を再び `OPEN` または `IN_PROGRESS` に戻すことはできず、reopen は admin のみ可能である

### 5-3. 個人情報修正アーキテクチャ原則

- 現在の MVP 第1段階では、**直接修正主体と実際の保存主体を単純に維持**することを優先する
- teacher 自身のアカウント情報修正は、**teachers テーブルの許可フィールドのみを直接修正**する方式で処理する
- student アカウント情報修正は、**teacher 画面で students テーブルの許可フィールドのみを修正**する方式で処理する
- teacher の許可フィールドは `login_id`, `password`, `name`, `email`, `phone`, `emergency_contact` に限定する
- student の許可フィールドは `login_id`, `password`, `name`, `email`, `guardian_email`, `phone`, `guardian_phone` に限定する
- ここでの `password` 入力は、保存時に `password_hash` として反映される
- `is_active`, `is_admin` は一般アカウント修正ページではなく、**admin 専用状態/権限変更ページ** でのみ修正する
- 一方、`teacher_code`, `student_code`, `created_at`, `updated_at`, `deleted_at` のようなシステム管理用フィールドはアカウント修正範囲から除外する
- 後続バージョンで student の自己修正機能を導入する場合は、students テーブルへ直接書き込むより、**修正リクエスト生成 → teacher 承認 → 実反映** の構造へ拡張するのが、現在のアーキテクチャと最も整合する

### 5-4. `login_id` 変更時の再同期原則

- 現在の QR 生成ルールには `login_id` が含まれるため、`login_id` が変更されると既存端末のローカル QR 生成情報はもはや最新状態ではない可能性がある
- したがって、teacher または student の `login_id` が変更された場合、当該端末は再ログインまたは再同期が必要になる場合がある

---

## 6. audit_logs 記録原則

- `audit_logs` ページは **admin のみアクセス可能** である
- `audit_logs` データは **誰も修正できない**
- 生成（create）および一般的な作成行為も `audit_logs` に記録する
- admin による実際の修正（update/delete）が発生した場合、`audit_logs` を記録する
- soft delete は実際の SQL では `deleted_at` 更新であるが、`audit_logs.action_type` は `DELETE` として記録する
- `action_type = CREATE` の場合、`reason` はユーザー入力を強制せず、デフォルト値 `CREATE` を使用する
- `action_type = UPDATE` または `DELETE` の場合、`reason` は **必須** であり、これは文書ルールだけでなく DDL の CHECK 制約でも強制する
- QR スキャン、状態ボタン変更、note 修正などのシステム性 update/delete についても、`reason` 記録を例外なく強制する
- システム性処理の場合、サーバーは事前定義された system reason 文字列を自動記録する
- つまり、`audit_logs.reason` は QR スキャン専用ではなく、一般修正、状態変更、note 修正など **すべての変更行為に対する共通理由** を記録するためのフィールドである

---

## 7. 給与ルール

- 給与は月給制のみ対応する
- 金額単位は KRW 整数型（INTEGER）
- `final_amount = monthly_salary + bonus_amount - deduction_amount` を保存する方式
- `worked_minutes` は参考用データである
- 同じ teacher に対して、同じ精算期間の給与は1件のみ許容する
- MVP 第1段階の payroll 範囲は **給与計算、金額確定、支払状態管理** までに限定する
- 実際の送金はシステム外で手動または別手続きで処理する

---

## 8. 問い合わせルール

### 8-1. データ構造原則

- inquiry は **学生基準で紐づく問い合わせデータ** として統合管理する
- 実際の問い合わせ主体が保護者か学生かは別カラムとして分離せず、**問い合わせ内容に明記して管理**する
- `author_teacher_id` は、問い合わせをシステムに作成/登録した teacher ID を意味する
- `response_teacher_id` は、**現在の response の最終修正者 ID** を意味する
- 名前 snapshot は、その時点の表示名を保持するために保存する

### 8-2. 作成および修正権限

- 元の response 作成者や過去の修正履歴を確認するには `audit_logs` を参照する必要がある
- non-admin teacher でも **新規 inquiry 作成** と **未回答 inquiry への初回回答入力** は可能である
- また、non-admin teacher は **自分が最後に修正した inquiry response** に限り修正可能である
- inquiry 本文の修正は不可であり、**他人が最後に修正した既存回答** の再修正も不可である

### 8-3. 状態変更ルール

- inquiry `status` は UI 上のボタンで修正できる
- すべての teacher は inquiry `status` を変更できる
- ただし、non-admin teacher は `CLOSED` 状態の inquiry を再び `OPEN` または `IN_PROGRESS` に戻すことはできず、このような reopen は admin のみ可能である
- 状態値は `OPEN`, `IN_PROGRESS`, `CLOSED`, `HOLD` の4つのみ使用する

---

## 9. 出席記録ルール

### 9-1. 基本保存原則

- MVP 第1段階では、`teacher_attendance` と `student_attendance` は **記録そのものを保存し、状態は解釈して表示する方式** を使用する
- したがって attendance テーブルには `status` と `reason` を置かず、必要時のみ手動入力用の `note` を使用する
- `teacher_attendance.note`, `student_attendance.note` は基本的に NULL を許容する
- note は、特殊なケースがあったかを簡潔に手動記入するための用途である

### 9-2. 出席 QR の保存および検証基準

- 出席 QR は `LOGIN_ID|CODE|TIME_SLOT|PORT` 形式を使用する
- `CODE` は `student_code` または `teacher_code` であり、prefix で student / teacher を区別する
- `TIME_SLOT` は60秒単位の再生成基準値である
- サーバーは次の順で QR を検証する
  1. QR をパースする
  2. code prefix で student / teacher を区別する
  3. 該当テーブルで code を検索する
  4. row の `login_id` と QR の `login_id` が一致するか確認する
  5. QR の `time_slot` と現在のマスターサーバー `time_slot` が一致するか確認する
  6. `port` 値が現在のサーバーポートと一致するか確認する
  7. すべて一致すれば有効と判定する
- 現在の QR 生成ルールは **MVP 第1段階向けの単純な検証構造** であり、強固な偽造防止を保証する秘密署名構造ではない

### 9-3. note 作成 / 修正権限

- 出席 note 関連の作成者 ID は **元の作成者** ではなく **現在値の最終修正者** を意味する
- したがって、`note_editor_teacher_id` は現在の note 値を最後に修正した teacher ID を保存する
- 元の作成者を確認するには `audit_logs` を参照する必要がある
- non-admin teacher でも出席処理過程で note 作成は可能である
- non-admin teacher は自分が最後に修正した出席 note のみ再修正可能であり、他人が最後に修正した note は修正できない
- admin は出席 note 修正に制限を設けない

### 9-4. teacher 出退勤解釈ルール

- teacher の出退勤状態は保存せず、`checkin_at`, `checkout_at` の値を基準に解釈する
- 同じ日の1回目のスキャンは出勤、2回目のスキャンは退勤、3回目以降のスキャンはエラーとして処理する
- つまり、`teacher_attendance` は1日最大2回のスキャンのみ許容する
- teacher QR の1回目のスキャンは `teacher_attendance` row の生成、2回目のスキャンは同じ row の `checkout_at` 更新として処理する
- 出勤のみ記録し、退勤を記録しないまま日付が変わった場合、翌日の最初のスキャンは新しい出勤として処理する
- 前日の未完了 row は維持し、その後 UI で退勤未記録状態として解釈して表示できる
- teacher 出席 QR は、過去にログインして必要なローカル保存情報を確保した端末であれば、現在のログイン状態に関係なく表示可能である
- 一方、teacher QR をスキャンしてサーバーに出勤/退勤記録を残すスキャンモードは、**ログイン済み teacher 端末** でのみ可能である

### 9-5. student 出席ルール

- `student_attendance` は MVP 第1段階で **授業別出席ではなく、学生単位のスキャン記録** を保存する構造を使用する
- student QR スキャンでは `class_id` を使用しない
- student 出席 row は `student_id` と `scanned_at` を中心に保存する
- student 出席の日付判定は `scanned_at` のマスターサーバーのローカル日付基準で処理する
- 学生は重複出席が可能だが、同一学生基準で5分以内の再スキャンはできないよう制限し、多重入力を防止する
- 学生が複数回出席できる理由は、連続していない時間帯に複数の授業を受ける場合があるためである
- このルールは MVP 第1段階の単純な保護ルールであり、必要に応じて後続バージョンで授業/時間ベースのポリシーへさらに精緻化できる

---

## 10. 変更履歴 UI ルール

- `audit_logs` に記録がある値は UI 上で赤色表示する
- この表示は、変更履歴が存在することを視覚的に示すことを目的とする
- `audit_logs` ページは admin のみ参照可能な参照専用ページであり、修正機能は提供しない

---

## 11. 削除ポリシー

### 11-1. 基本削除方式

- 削除ポリシーは **soft delete 方式** を採用する
- soft delete カラムは `audit_logs` を除くすべてのテーブルで `deleted_at` に統一する
- 削除時は物理削除の代わりに `deleted_at` に削除時刻を記録する
- 基本参照は `deleted_at IS NULL` 条件を基準に処理する

### 11-2. 参照および状態解釈原則

- soft delete されたデータはアプリの基本画面では参照されず、直接 DB にアクセスして初めて確認できる
- `is_active = false` はアプリ上で確認可能な **非アクティブ状態** を意味する
- `deleted_at IS NOT NULL` はアプリの基本参照から除外される **削除状態** を意味する

### 11-3. UNIQUE および運用目的

- soft delete された row の UNIQUE 値は再利用可能でなければならず、そのため UNIQUE は active row 基準でのみ強制する
- この方式の目的は、運用履歴の保存、復旧可能性の確保、`audit_logs` との整合性維持にある