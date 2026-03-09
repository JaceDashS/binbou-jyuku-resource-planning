# 03. ユーザー役割 / 権限 / クライアントUIフロー

## 1. 文書の目的

この文書は、**アカウント種別、権限モデル、ログイン規則、アカウント修正可能範囲、クライアントアプリの進入構造、管理モードの権限分岐** を整理した基準文書である。  
接続方式とトークンは `02-runtime-architecture-and-connection.md`、QR / 出欠ポリシーは `04-qr-and-attendance-policy.md`、データ構造とスキーマは `05-domain-rules-and-schema.md` で詳細に定義する。

---

## 2. 基本役割モデル

### 2-1. 運営および役割の原則

- 小規模施設を前提とするため、**teacher と staff を別個の役割として区分しない**
  - その理由は、**最小実装の達成** と **UX の単純化** を優先するためであり、運営補助の役割も teacher のカテゴリ内で処理する

### 2-2. アカウント種別

```text
TEACHER
STUDENT
````

### 2-3. アカウント属性

```text
teachers.is_admin
teachers.is_active
students.is_active
```

---

## 3. ログインおよびアカウント利用ルール

### 3-1. アカウント識別原則

* メインログイン画面で最初に `teacher` / `student` 種別を選択する
* したがって、`teachers.login_id` と `students.login_id` が互いに同じ値を持っていても許容する
* 各テーブル内部では active row 基準で UNIQUE でなければならない
* soft delete された row の UNIQUE 値は **再利用可能** でなければならない

### 3-2. teacher 接続およびログイン原則

* teacher がマスターサーバー機能にアクセスする場合は、**必ず IP + 接続トークン + teacher login_id + password** でログインしなければならない
* ログイン成功時にサーバーは `teacher_auth_token` を発行する
* 接続トークンは **初回ログイン時にのみ使用** する
* 以後、出欠処理、チェックイン、チェックアウト、管理機能のリクエストでは **接続トークンを再使用しない**
* 以後の teacher 機能は、ログイン済み teacher 端末の認証状態を基準として処理する
* teacher 関連リクエストは **Authorization ヘッダーの `teacher_auth_token` 検証** を経る
* ネットワークが一時的に切断されて再接続された場合でも、`teacher_auth_token` が有効なら再ログインは不要である
* `teacher_auth_token` は **マスターサーバーのローカル日付が変更されると無効化** され、その後最初の teacher リクエスト時に再ログインが必要となる
* 同一端末から接続する場合もこの原則を維持し、この場合は `127.0.0.1:40343` を使用できる

### 3-3. 登録および初期アカウント原則

* **新規登録 UI は teacher アカウントに対してのみ提供** する
* 初期インストール時には、**基本 admin teacher アカウント 1 個をデフォルトで提供** する

### 3-4. student アカウント利用原則

* student は新規登録 UI を使用しない
* student アカウントは teacher または admin が登録する
* student 端末では **初回ログイン 1 回と初回 QR 生成 1 回のみ**、マスターと通信可能な状態が必要である

---

## 4. アカウント / 個人情報修正ルール

### 4-1. teacher 本人の修正範囲

teacher は **本人アカウントの** 次のフィールドを修正できる。

* `login_id`
* `password`
* `name`
* `email`
* `phone`
* `emergency_contact`

保存時、`password` 入力は実際には `password_hash` として反映される。

### 4-2. student 修正範囲

student アカウント情報は **teacher 画面を通じてのみ** 修正可能であり、修正範囲は次に限定する。

* `login_id`
* `password`
* `name`
* `email`
* `guardian_email`
* `phone`
* `guardian_phone`

保存時、`password` 入力は実際には `password_hash` として反映される。

### 4-3. student 本人による直接修正禁止

* MVP 1 次では **student 本人が直接個人情報を修正しない**
* student は **個人情報直接修正ページを使用しない**

### 4-4. 状態 / 権限変更専用ページ

* admin アカウントでログインした場合は、**状態 / 権限変更ページへ移動できるボタン** が表示される
* 該当ページで teacher の `is_active`, `is_admin` と student の `is_active` を変更できる
* `is_active`, `is_admin` は一般アカウント修正ページではなく、**admin 専用の状態 / 権限変更ページ** でのみ修正する

### 4-5. 修正範囲から除外されるシステム管理フィールド

次のようなフィールドはアカウント修正範囲から除外する。

* `teacher_code`
* `student_code`
* `created_at`
* `updated_at`
* `deleted_at`

### 4-6. 後続バージョンでの拡張方針

* 後続バージョンで student 自己修正機能を導入する場合は、students テーブルへ直接書き込むよりも、**修正リクエスト作成 → teacher 承認 → 実反映** 構造へ拡張するのが、現在のアーキテクチャと最も整合する

### 4-7. `login_id` 変更時の再同期原則

* 現在の QR 生成ルールには `login_id` が含まれるため、`login_id` が変更されると既存端末のローカル QR 生成情報はもはや最新状態ではない可能性がある
* したがって teacher または student の `login_id` が変更された場合、該当端末は再ログインまたは再同期が必要になる場合がある

---

## 5. クライアントアプリの基本進入構造

### 5-1. 基本原則

* クライアントアプリはログイン後、ユーザーに応じて **基本的に QR 中心画面** を最初に表示する
* すなわち、ログイン直後に直ちに管理機能画面へ分岐するのではなく、**共通の基本画面は QR 画面** として解釈する
* ユーザー種別に応じて、QR 画面の後にアクセス可能な機能が異なる

### 5-2. 共通フロー

1. ユーザー種別確認
2. 接続またはログイン条件確認
3. 基本 QR 画面表示

---

## 6. student UI ルール

### 6-1. Student 基本画面

アプリ実行後に表示される基本画面:

```text
学生本人のQRコード
```

### 6-2. 基本原則

* student は基本的に **出欠用 QR 提示画面のみを使用** する
* student は事実上 **QR 提示中心の最小機能ユーザー** として定義する
* student は **別途の新規登録 UI を使用しない**
* student は **個人情報直接修正ページを使用しない**
* student アカウントは teacher または admin がシステムで登録する

### 6-3. ログインおよび QR 生成原則

* student 端末では **初回ログイン 1 回と初回 QR 生成 1 回のみ**、マスターと通信可能な状態が必要である
* 初回ログインと初回 QR 生成後は、student が同一ネットワークに継続して接続されていなくても、基本 QR 表示の利用は可能である
* student 出欠 QR はローカルに保存された情報基準で継続生成可能である

### 6-4. 許可 / 不可

許可:

* 本人 QR 表示
* 本人関連の最小情報照会(必要時)

不可:

* 管理モード進入
* student / 授業 / 出欠の編集
* 個人情報直接修正
* 管理者機能アクセス
* student アカウント新規登録

### 6-5. フロー

```text
teacher または admin が student アカウントを登録
→ student 端末で初回ログイン
→ 初回 QR 生成に必要な情報を保存
→ 以後、本人 QR 画面を使用
```

---

## 7. teacher UI ルール

### 7-1. Teacher 基本原則

* teacher もアプリ実行時に **基本的に QR 画面** を持つ
* teacher も **本人 QR コードを表示可能**
* student との違いは、teacher にのみ **管理モード進入ボタン** が見えるという点である
* teacher がマスター機能にアクセスするには、**必ず IP + 接続トークン + teacher login_id + password でログイン** しなければならない
* ログイン成功時に発行された `teacher_auth_token` を基準として teacher 機能を実行する
* 出欠処理、チェックイン、チェックアウト時に **接続トークンを再入力したり再送信したりしない**
* `teacher_auth_token` が有効な間は、再度 login_id / password を手入力せず継続使用できる
* ただし `teacher_auth_token` が無効化されると再ログインが必要である
* teacher は **本人アカウント / 個人情報修正ページ** にアクセスできる
* student アカウント情報修正は **teacher 画面を通じて処理** する
* admin アカウントでログインした場合は、**状態 / 権限変更ページへ入れるボタン** が追加表示される

teacher 画面には次のボタンが存在する。

```text
[管理モードに入る]
```

* このボタンを押したときのみ管理機能 UI に進入する

### 7-2. teacher QR 表示原則

* teacher QR 出力は、過去に一度ログインして必要なローカル保存情報を確保した端末であれば、現在のログイン状態に関係なく可能である
* すなわち、`teacher_auth_token` が期限切れの状態でも、端末に必要情報が残っていれば本人 QR を継続表示できる
* ただし QR を読み取ってサーバーに記録するスキャンモードと管理機能リクエストは、**ログイン済み teacher 端末** でのみ可能である

### 7-3. 管理者アクセス方式

* 管理機能には **クライアントアプリの管理モード** を通じてアクセスすることを基本原則とする
* teacher 関連リクエストは **Authorization ヘッダーの `teacher_auth_token` 検証** を経なければならない
* すなわち、管理者は別途ブラウザ URL を直接入力する方式よりも、**teacher クライアントでログイン後に管理モードへ進入** するフローを基本 UX とする

---

## 8. Teacher 権限細分化ルール

### 8-1. Teacher + is_admin = false

* QR 画面使用可能
* 管理モードボタン表示
* 管理モード進入可能
* teacher 関連リクエストは **各リクエストごとに `teacher_auth_token` 検証** を経る
* `teacher_auth_token` が有効な間は teacher 機能を継続使用できる
* ただし **一般的なデータ修正権限はない**

許可例:

* 出欠処理
* 一部照会
* 本人担当授業関連機能照会
* **本人アカウント情報修正**
* **student アカウント情報修正**
* **新規 inquiry 作成**
* `response` がまだ NULL である inquiry に対する初回回答 1 回入力
* **自分が最後に修正した inquiry response の修正**
* **inquiry `status` の変更**
* **出欠 note の作成**
* **自分が最後に修正した出欠 note の修正**

追加制限:

* non-admin teacher は `CLOSED` 状態の inquiry を再び `OPEN` または `IN_PROGRESS` に戻す reopen を実行できない
* すなわち、**終了済み inquiry を再度開くことができるのは admin のみ** である

不可例:

* **アカウント / 個人情報修正範囲を超える** student / teacher / 授業 / 給与データ修正
* 他の teacher のアカウント情報修正
* `is_active`, `is_admin` 状態 / 権限変更
* 既存 inquiry 本文修正
* 他人が最後に修正した inquiry response 修正
* 他人が最後に修正した出欠 note 修正
* 終了済み inquiry の reopen
* システム設定
* 全体変更履歴管理

すなわち、non-admin teacher には **一般的な管理性修正権限はなく**、例外的に本人アカウント情報修正、student アカウント情報修正、inquiry 作成、未回答 inquiry への初回回答入力、自分が最後に修正した inquiry response 修正、限定された inquiry status 変更、出欠 note 作成、自分が最後に修正した出欠 note 修正のみ可能である。

フロー:

```text
IP 入力
→ 接続トークン入力
→ teacher login_id / password 入力
→ ログイン成功
→ teacher_auth_token 発行
→ 本人 QR 画面
→ [管理モード] ボタン表示
→ 管理モード進入時に制限機能 UI
```

### 8-2. Teacher + is_admin = true

* QR 画面使用可能
* 管理モードボタン表示
* 管理モード進入可能
* teacher 関連リクエストは **各リクエストごとに `teacher_auth_token` 検証** を経る
* `teacher_auth_token` が有効な間は teacher 機能を継続使用できる
* 全権限許可

すなわち:

```text
Teacher(is_admin=true) = 管理者権限を含む teacher
```

フロー:

```text
IP 入力
→ 接続トークン入力
→ teacher login_id / password 入力
→ ログイン成功
→ teacher_auth_token 発行
→ 本人 QR 画面
→ [管理モード] ボタン表示
→ 管理モード進入時に全機能 UI
```

---

## 9. クライアントアプリ画面フロー

### 9-1. Student

```text
teacher または admin が student アカウントを登録
→ student 端末で初回ログイン
→ 初回 QR 生成に必要な情報を保存
→ 以後、本人 QR 画面を使用
```

### 9-2. Teacher (non-admin)

```text
IP 入力
→ 接続トークン入力
→ teacher login_id / password 入力
→ ログイン成功
→ teacher_auth_token 発行
→ 本人 QR 画面
→ [管理モード] ボタン表示
→ 管理モード進入時に制限機能 UI
```

### 9-3. Teacher (admin)

```text
IP 入力
→ 接続トークン入力
→ teacher login_id / password 入力
→ ログイン成功
→ teacher_auth_token 発行
→ 本人 QR 画面
→ [管理モード] ボタン表示
→ 管理モード進入時に全機能 UI
```

---

## 10. UI 権限説明原則

従来のように単純に student UI / teacher UI / admin UI と分離して説明するより、次のように説明する方がより正確である。

* **基本画面は QR 画面**
* **Teacher のみ管理モードボタンを表示**
* **管理モード内部の権限は `teachers.is_admin` の有無で決定**

構造は次のとおりである。

```text
基本 QR 画面
 ├─ Student: QR のみ使用
 └─ Teacher: 管理モードボタン追加
          ├─ non-admin: 制限管理機能
          └─ admin: 全体管理機能
```

---

## 11. 文書用追加文案

```text
クライアントアプリはログイン後、ユーザーに応じて基本 QR 画面を提供する。

student アカウントはアプリ実行後、本人出欠用 QR コードのみが表示される最小機能画面を使用する。
student は別途の新規登録 UI と個人情報直接修正 UI を使用せず、teacher または admin がアカウントを登録した後、student 端末で初回ログインと初回 QR 生成に必要な情報保存を実行する。

teacher アカウントも基本的に本人 QR コードを見ることができるが、teacher にのみ管理モード進入ボタンが表示される。
teacher は本人アカウント / 個人情報修正ページを使用でき、student アカウント情報修正も teacher 画面を通じて処理する。
admin アカウントでログインした場合は、状態 / 権限変更ページへ入ることができるボタンが追加で表示される。

teacher がマスター機能にアクセスするには、必ず IP + 接続トークン + teacher login_id + password でログインしなければならない。
ログイン成功時に `teacher_auth_token` が発行され、以後 teacher 関連リクエストは Authorization ヘッダー基盤で処理する。
ただし teacher QR 出力は、過去に一度ログインして必要なローカル保存情報を確保した端末であれば、現在のログイン状態に関係なく可能である。
一方で、スキャンモード、出欠処理、チェックイン、チェックアウト、管理機能リクエストはログイン済み teacher 端末でのみ可能である。

teacher が管理モードに進入した場合、権限範囲は teachers.is_admin 値に応じて決定される。

- teachers.is_admin = false の場合: 制限された管理機能のみ許可
- teachers.is_admin = true の場合: 全管理機能許可
```

---

## 12. 最終追加要約

1. **クライアントアプリはログイン後の基本画面を QR 画面とする**
2. **Student は QR 中心の最小機能のみを使用し、個人情報を直接修正しない**
3. **Teacher は QR 画面 + 管理モードボタン表示、本人アカウント情報修正可能**
4. **Student アカウント情報修正は teacher 画面を通じて処理**
5. **Admin ログイン時は状態 / 権限変更ページボタンが追加表示される**
6. **管理モード権限は `teachers.is_admin` 値で分岐**
7. **teacher 機能は `teacher_auth_token` 基盤で使用する**
8. **teacher QR 出力と teacher スキャン / 管理機能は区分される**
