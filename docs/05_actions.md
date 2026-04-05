# アクション実装

## 目次

- [バウンドアクション](#バウンドアクション)
- [アンバウンドアクション](#アンバウンドアクション)
- [パラメータの受け取り](#パラメータの受け取り)

---

## バウンドアクション

**概要**  
特定のエンティティインスタンスに紐づくアクション（例: `Books/submitOrder`）を実装する。cds-maven-plugin が生成した型付き `EventContext` を使うと、パラメータにタイプセーフにアクセスできる。

**コード**  

```java
// CDS 定義例:
// action submitOrder(quantity: Integer) returns Orders;

// cds-maven-plugin が生成する EventContext インターフェース（参考）
@EventName("submitOrder")
public interface SubmitOrderContext extends EventContext {
    CqnSelect getCqn();        // バウンドアクション: 対象エンティティへの参照
    Integer getQuantity();
    void setResult(Orders result);
}

@Component
@ServiceName(CatalogService_.CDS_NAME)
public class BooksHandler implements EventHandler {

    private final CqnAnalyzer analyzer;

    public BooksHandler(CdsModel cdsModel) {
        // CqnAnalyzer はコンストラクタで一度だけ生成してフィールドに保持する
        this.analyzer = CqnAnalyzer.create(cdsModel);
    }

    @On(event = "submitOrder", entity = Books_.CDS_NAME)
    public Orders submitOrder(SubmitOrderContext context) {
        // 生成された型付きゲッターでパラメータを取得
        Integer quantity = context.getQuantity();

        // キーを取得して対象エンティティを特定
        Object bookId = analyzer.analyze(context.getCqn()).targetValues().get(Books.ID);

        // return で返す（ランタイムが setResult + setCompleted を自動処理）
        return processOrder(bookId, quantity);
    }
}
```

**補足**  
- `event` にはアクション名（CDS で定義した名前）を指定する。
- 返り値ありの `@On` は `return` で返すだけでよい。`setResult` / `setCompleted` は不要。
- 型付き `EventContext` は cds-maven-plugin により CDS モデルから自動生成される。

---

## アンバウンドアクション

**概要**  
エンティティに紐づかないサービスレベルのアクションを実装する。

**コード**  

```java
// CDS 定義例:
// action resetCatalog() returns Boolean;

// cds-maven-plugin が生成する EventContext インターフェース（参考）
@EventName("resetCatalog")
public interface ResetCatalogContext extends EventContext {
    void setResult(Boolean result);
}

@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    @On(event = "resetCatalog")
    public Boolean resetCatalog(ResetCatalogContext context) {
        return performReset();
    }
}
```

**補足**  
- アンバウンドアクションは `entity` を指定しない。
- `returns` に対応する Java ラッパー型（`Boolean`, `Integer` など）を `return` する。

---

## パラメータの受け取り

**概要**  
アクションに定義した複数のパラメータを型付き `EventContext` で取得する。

**コード**  

```java
// CDS 定義例:
// action transferStock(fromBookId: UUID, toBookId: UUID, quantity: Integer);

// cds-maven-plugin が生成する EventContext インターフェース（参考）
@EventName("transferStock")
public interface TransferStockContext extends EventContext {
    CqnSelect getCqn();
    String getFromBookId();
    String getToBookId();
    Integer getQuantity();
}

@Component
@ServiceName(CatalogService_.CDS_NAME)
public class BooksHandler implements EventHandler {

    @On(event = "transferStock", entity = Books_.CDS_NAME)
    public void transferStock(TransferStockContext context) {
        String fromBookId = context.getFromBookId();
        String toBookId   = context.getToBookId();
        Integer quantity  = context.getQuantity();

        doTransfer(fromBookId, toBookId, quantity);
        context.setCompleted();
    }
}
```

**補足**  
- 型付き `EventContext` を使うと `context.get(String)` によるキャストが不要になる。
- 複合パラメータ（構造体型）も生成されたクラスで型安全に受け取れる。
- `returns` を持たないアクションは `void` + `context.setCompleted()` を使う。

> 参照: [Actions and Functions – CAP Java](https://cap.cloud.sap/docs/java/cqn-services/application-services)
