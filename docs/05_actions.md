# アクション実装

## バウンドアクション

**概要**  
特定のエンティティインスタンスに紐づくアクション（例: `Books/submitOrder`）を実装する。

**コード**  

```java
// CDS 定義例:
// action submitOrder(quantity: Integer) returns Orders;

@On(event = "submitOrder", entity = Books_.CDS_NAME)
public void submitOrder(CdsActionEventContext context) {
    // アクションのパラメータを取得
    Integer quantity = context.get("quantity");

    // キーを取得して対象エンティティを特定
    CqnAnalyzer analyzer = CqnAnalyzer.create(cdsModel);
    Object bookId = analyzer.analyze(context.getCqn()).targetValues().get(Books.ID);

    // 処理
    Orders order = processOrder(bookId, quantity);
    context.setResult(order);
    context.setCompleted();
}
```

**補足**  
- `event` にはアクション名（CDS で定義した名前）を指定する。
- `context.setCompleted()` を呼ばないとデフォルト処理が続行される。

---

## アンバウンドアクション

**概要**  
エンティティに紐づかないサービスレベルのアクションを実装する。

**コード**  

```java
// CDS 定義例:
// action resetCatalog() returns Boolean;

@On(event = "resetCatalog")
public void resetCatalog(CdsActionEventContext context) {
    // 処理
    boolean success = performReset();
    context.setResult(success);
    context.setCompleted();
}
```

**補足**  
- アンバウンドアクションは `entity` を指定しない。
- `returns` に対応する Java 型を `setResult()` で返す（プリミティブ型はラッパー型を使う）。

---

## パラメータの受け取り

**概要**  
アクションに定義した複数のパラメータを型安全に取得する。

**コード**  

```java
// CDS 定義例:
// action transferStock(fromBookId: UUID, toBookId: UUID, quantity: Integer);

@On(event = "transferStock", entity = Books_.CDS_NAME)
public void transferStock(CdsActionEventContext context) {
    String fromBookId = context.get("fromBookId");
    String toBookId   = context.get("toBookId");
    Integer quantity  = context.get("quantity");

    // 処理
    doTransfer(fromBookId, toBookId, quantity);
    context.setCompleted();
}
```

**補足**  
- `context.get(String key)` の戻り値は `Object` なので、適切な型にキャストする。
- 複合パラメータ（構造体型）は `Map` で受け取るか、生成されたクラスを使う。
