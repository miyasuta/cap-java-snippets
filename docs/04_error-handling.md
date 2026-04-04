# エラーハンドリング

## req.error() と ServiceException の使い分け

**概要**  
バリデーションエラーは `req.error()` で複数まとめて返し、予期しない例外は `ServiceException` でスローする。

**コード**  

```java
// req.error(): バリデーション（複数エラーをまとめて返せる）
@Before(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
public void validate(CdsCreateEventContext context, List<Books> books) {
    books.forEach(book -> {
        if (book.getTitle() == null || book.getTitle().isBlank()) {
            context.getMessages().error("TITLE_REQUIRED");
        }
        if (book.getStock() != null && book.getStock() < 0) {
            context.getMessages().error("STOCK_NEGATIVE");
        }
    });
}

// ServiceException: 予期しないエラー（即座にスロー）
@On(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void handler(CdsReadEventContext context) {
    try {
        // 外部サービス呼び出しなど
    } catch (Exception e) {
        throw new ServiceException(ErrorStatuses.SERVER_ERROR, "EXTERNAL_SERVICE_FAILED", e);
    }
}
```

**補足**  
- `req.error()` はハンドラ終了後にまとめてエラーレスポンスを生成する（処理は続行する）。
- `ServiceException` は即座に処理を中断する。HTTP ステータスは `ErrorStatuses` 定数で指定する。

---

## i18n メッセージの取得

**概要**  
エラーメッセージを `i18n.properties` から取得し、多言語対応する。

**コード**  

```java
// src/main/resources/i18n/i18n.properties
// TITLE_REQUIRED=Title is required
// STOCK_NEGATIVE=Stock cannot be negative

context.getMessages().error("TITLE_REQUIRED");
// または引数つき
context.getMessages().error("STOCK_RANGE", 0, 9999);
// i18n: STOCK_RANGE=Stock must be between {0} and {1}
```

**補足**  
- プロパティファイルは `src/main/resources/i18n/` に配置する。
- CAP はリクエストの `Accept-Language` ヘッダに基づいて言語を自動選択する。

---

## フィールドレベルエラー

**概要**  
Fiori Elements で特定フィールドをハイライト表示させるために、エラーにターゲットを指定する。

**コード**  

```java
context.getMessages().error("TITLE_REQUIRED")
    .target("in", Books_.class, b -> b.title());

// ネストしたパスの例（アソシエーション先のフィールド）
context.getMessages().error("AUTHOR_NAME_MISSING")
    .target("in", Books_.class, b -> b.author().name());
```

**補足**  
- `.target("in", ...)` の第1引数は CDS アノテーションの `target` に相当する。通常は `"in"` を使う。
- Fiori Elements の List Report / Object Page はフィールドレベルエラーをインライン表示できる。
