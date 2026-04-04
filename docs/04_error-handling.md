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
            context.getMessages().error(MessageKeys.TITLE_REQUIRED);
        }
        if (book.getStock() != null && book.getStock() < 0) {
            context.getMessages().error(MessageKeys.STOCK_NEGATIVE);
        }
    });
}

// ServiceException: 予期しないエラー（即座にスロー）
@On(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void handler(CdsReadEventContext context) {
    try {
        // 外部サービス呼び出しなど
    } catch (Exception e) {
        throw new ServiceException(ErrorStatuses.SERVER_ERROR, MessageKeys.EXTERNAL_SERVICE_FAILED, e);
    }
}
```

**補足**  
- `req.error()` はハンドラ終了後にまとめてエラーレスポンスを生成する（処理は続行する）。
- `ServiceException` は即座に処理を中断する。HTTP ステータスは `ErrorStatuses` 定数で指定する。

---

## messages.properties によるメッセージ管理

**概要**  
メッセージテキストは `src/main/resources/messages.properties` で定義し、キーは `MessageKeys` 定数クラスで一元管理する。

**コード**  

```properties
# src/main/resources/messages.properties
book.require.stock = Not enough books on stock (only {0} left)
book.stock.range   = Stock must be between {0} and {1}
title.required     = Title is required
```

```java
// MessageKeys.java: キーを定数で管理し、文字列の直書きを避ける
public class MessageKeys {
    public static final String BOOK_REQUIRE_STOCK = "book.require.stock";
    public static final String BOOK_STOCK_RANGE   = "book.stock.range";
    public static final String TITLE_REQUIRED     = "title.required";
}
```

```java
// 使用例
context.getMessages().error(MessageKeys.TITLE_REQUIRED);

// プレースホルダ引数あり（{0}, {1} の順に渡す）
context.getMessages().error(MessageKeys.BOOK_REQUIRE_STOCK, book.getStock());
```

**補足**  
- ファイルは `src/main/resources/messages.properties`（i18n サブディレクトリではない）。
- キーはドット区切り小文字を使う（例: `book.require.stock`）。
- メッセージキーを文字列リテラルで直書きすると typo やキー変更時の追跡が困難になるため、`MessageKeys` クラスに定数として集約する。

---

## フィールドレベルエラー

**概要**  
Fiori Elements で特定フィールドをハイライト表示させるために、エラーにターゲットを指定する。

**コード**  

```java
context.getMessages().error(MessageKeys.TITLE_REQUIRED)
    .target("in", Books_.class, b -> b.title());

// ネストしたパスの例（アソシエーション先のフィールド）
context.getMessages().error(MessageKeys.AUTHOR_NAME_MISSING)
    .target("in", Books_.class, b -> b.author().name());
```

**補足**  
- `.target("in", ...)` の第1引数は CDS アノテーションの `target` に相当する。通常は `"in"` を使う。
- Fiori Elements の List Report / Object Page はフィールドレベルエラーをインライン表示できる。
