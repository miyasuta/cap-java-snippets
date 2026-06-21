# CAP Java スニペット集

SAP Cloud Application Programming Model (CAP) Java SDK の実装パターンをまとめたリファレンスです。

## ドキュメント一覧

| # | タイトル | 内容 |
|---|---|---|
| [01](docs/01_event-handler.md) | イベントハンドラ | `@Before`/`@On`/`@After` の使い分け、ドラフトイベント、バリデーション |
| [02](docs/02_cqn.md) | CQN / クエリ操作 | `CqnAnalyzer` によるキー抽出、WHERE 条件の追加 |
| [03](docs/03_cql.md) | CQL / データアクセス | Select/Insert/Update/Delete の基本形、バルク Upsert |
| [04](docs/04_error-handling.md) | エラーハンドリング | `req.error()` と `ServiceException` の使い分け、i18n、フィールドレベルエラー、Composition 子エンティティへのエラーターゲット指定 |
| [05](docs/05_actions.md) | アクション実装 | バウンドアクション、アンバウンドアクション、パラメータ受け取り |
| [06](docs/06_auth.md) | 認証・認可 | ローカルモック認証の設定、`UserInfo` でのユーザ・ロール取得、JWT カスタム属性 |
| [07](docs/07_logging.md) | ログ出力 | SLF4J ロガーの定義と基本出力 |
| [08](docs/08_external-services.md) | 外部サービス呼び出し | RemoteService API、Cloud SDK + Destination、結果のマージ |
| [09](docs/09_local-hybrid-run.md) | ローカルハイブリッド実行 | hybrid / cloud プロファイルの使い分け、`cds bind --exec`、`.env` 要件 |
| [10](docs/10_testing.md) | 自動テスト | テスト3レイヤーの使い分け、ハンドラ単体 / サービス層 / 結合（MockMvc）の最小サンプル |
