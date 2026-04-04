# CAP Java スニペット集

SAP Cloud Application Programming Model (CAP) Java SDK の実装パターンをまとめたリファレンスです。

## ドキュメント一覧

| # | タイトル | 内容 |
|---|---|---|
| [01](docs/01_event-handler.md) | イベントハンドラ | `@Before`/`@On`/`@After` の使い分け、ドラフトイベント、バリデーション |
| [02](docs/02_cqn.md) | CQN / クエリ操作 | `CqnAnalyzer` によるキー抽出、WHERE 条件の追加 |
| [03](docs/03_cql.md) | CQL / データアクセス | Select/Insert/Update/Delete の基本形、バルク Upsert |
| [04](docs/04_error-handling.md) | エラーハンドリング | `req.error()` と `ServiceException` の使い分け、i18n、フィールドレベルエラー |
| [05](docs/05_actions.md) | アクション実装 | バウンドアクション、アンバウンドアクション、パラメータ受け取り |
| [06](docs/06_auth.md) | 認証・認可 | `UserInfo` でのユーザ・ロール取得、JWT カスタム属性 |
| [07](docs/07_logging.md) | ログ出力 | SLF4J ロガーの定義と基本出力 |
