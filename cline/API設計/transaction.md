# (ファイル名: transactions_api.md)

# 取引 (Transactions) API 設計書

**バージョン:** 1.0
**作成日:** 2025年4月27日

## 1. 概要
このドキュメントは、家計簿アプリ「Family Wallet」における取引（収入・支出）データの操作に関する API の仕様を定義します。

## 2. 共通事項
* **ベースパス:** `/api` (例: `https://your-worker.your-domain.workers.dev/api`)
* **認証:** すべてのエンドポイントで認証が必要です。リクエストヘッダーに `Authorization: Bearer <Supabase JWT>` を含めることを想定します。
* **データ形式:** リクエストボディ、レスポンスボディともに基本的に JSON 形式を使用します。
* **日付形式:** ISO 8601 形式 (例: `2025-04-27T15:58:00Z`) を使用します。
* **エラーレスポンス:** エラー発生時は、適切な HTTP ステータスコードと共に、以下のような JSON 形式でエラー情報を提供することを想定します。
  ```json
  {
    "error": {
      "message": "エラーメッセージ",
      "code": "エラーコード (任意)"
    }
  }
3. エンドポイント詳細
3.1. 取引の作成 (Create)
新しい収入または支出を記録します。

メソッド: POST
パス: /transactions
認証: 必要
リクエストボディ (JSON):
JSON

{
  "categoryId": "uuid-string",        // カテゴリID (必須)
  "targetUserId": "uuid-string | null", // 使用者/収入者ID (null許可)
  "amount": 1234.56,                  // 金額 (正の数) (必須)
  "isIncome": false,                    // 収入か支出か (boolean) (必須)
  "incurredAt": "2025-04-27T10:30:00Z", // 発生日時 (ISO 8601) (必須)
  "taxType": "tax_inclusive | tax_exclusive | null", // 税込/税抜 (null許可)
  "memo": "string | null"               // メモ (null許可)
}
レスポンス (成功時):
ステータスコード: 201 Created
ボディ (JSON): 作成された取引データ (ID、サーバーで設定される項目を含む)
JSON

{
  "id": "new-uuid-transaction",
  "familyId": "uuid-of-family",      // サーバー側で設定
  "userId": "uuid-of-creator",     // サーバー側で設定 (記録者)
  "categoryId": "uuid-string",
  "targetUserId": "uuid-string | null",
  "amount": 1234.56,
  "isIncome": false,
  "incurredAt": "2025-04-27T10:30:00Z",
  "taxType": "tax_inclusive | null",
  "memo": "string | null",
  "fixedExpenseId": null,           // 通常はnull
  "createdAt": "...",              // サーバー側で設定
  "updatedAt": "..."               // サーバー側で設定
}
レスポンス (失敗時):
400 Bad Request: 入力データバリデーションエラー
401 Unauthorized: 未認証
403 Forbidden: 権限不足 (例: 家族メンバーではない)
500 Internal Server Error: サーバー内部エラー
3.2. 取引の一覧取得 (Read List)
指定された条件に合う取引履歴の一覧を取得します。

メソッド: GET
パス: /transactions
認証: 必要 (自分の家族のデータのみ取得可能)
クエリパラメータ (任意):
startDate: string (YYYY-MM-DD) - 開始日 (この日を含む)
endDate: string (YYYY-MM-DD) - 終了日 (この日を含む)
categoryId: string (uuid) - カテゴリID
targetUserId: string (uuid または 'null') - 使用者/収入者ID (nullの場合を指定する方法を検討)
isIncome: boolean (true/false) - 収入のみ/支出のみ
limit: number (default: 50) - 取得件数上限
offset: number (default: 0) - スキップする件数 (ページネーション用)
sortBy: string (default: 'incurred_at') - ソート対象カラム名
order: string ('asc' or 'desc', default: 'desc') - ソート順
レスポンス (成功時):
ステータスコード: 200 OK
ボディ (JSON): 取引データの配列とメタ情報
JSON

{
  "data": [
    { /* 取引データオブジェクト1 */ },
    { /* 取引データオブジェクト2 */ },
    // ...
  ],
  "meta": {
    "totalCount": 150, // フィルタ条件に合う総件数
    "limit": 50,
    "offset": 0
  }
}
レスポンス (失敗時):
400 Bad Request: クエリパラメータの形式エラー
401 Unauthorized: 未認証
403 Forbidden: 権限不足
500 Internal Server Error: サーバー内部エラー
3.3. 特定の取引の詳細取得 (Read Detail)
特定の取引の詳細情報を取得します (編集画面用など)。

メソッド: GET
パス: /transactions/{transactionId}
{transactionId}: 取得したい取引の ID (uuid)
認証: 必要 (自分の家族のデータのみ取得可能)
リクエスト: なし (パスパラメータのみ)
レスポンス (成功時):
ステータスコード: 200 OK
ボディ (JSON): 指定された ID の取引データオブジェクト (3.1. の成功レスポンス内のオブジェクトと同様)
レスポンス (失敗時):
404 Not Found: 取引が見つからない、またはアクセス権限がない
401 Unauthorized: 未認証
500 Internal Server Error: サーバー内部エラー
3.4. 特定の取引の更新 (Update)
既存の取引情報を部分的に更新します。

メソッド: PATCH
パス: /transactions/{transactionId}
{transactionId}: 更新したい取引の ID (uuid)
認証: 必要 (自分の家族のデータのみ更新可能)
リクエストボディ (JSON): 更新したい項目のみを含むオブジェクト。
JSON

{
  "categoryId": "new-uuid-string", // 例: カテゴリ変更
  "amount": 1500.00,             // 例: 金額変更
  "memo": "更新後のメモ"         // 例: メモ変更
  // ... 他の更新可能な項目
}
レスポンス (成功時):
ステータスコード: 200 OK
ボディ (JSON): 更新後の取引データオブジェクト
レスポンス (失敗時):
400 Bad Request: 入力データバリデーションエラー
404 Not Found: 取引が見つからない、またはアクセス権限がない
401 Unauthorized: 未認証
500 Internal Server Error: サーバー内部エラー
3.5. 特定の取引の削除 (Delete)
既存の取引情報を削除します。

メソッド: DELETE
パス: /transactions/{transactionId}
{transactionId}: 削除したい取引の ID (uuid)
認証: 必要 (自分の家族のデータのみ削除可能)
リクエスト: なし
レスポンス (成功時):
ステータスコード: 204 No Content
ボディ: なし
レスポンス (失敗時):
404 Not Found: 取引が見つからない、またはアクセス権限がない
401 Unauthorized: 未認証
500 Internal Server Error: サーバー内部エラー
