# CQN / クエリ操作

## CqnAnalyzer によるキー抽出

**概要**  
WHERE 句からエンティティのキー値を取り出したいときに `CqnAnalyzer` を使う。ドラフト対応エンティティでは `IsActiveEntity` もキーに含まれる。

**コード**  

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class BooksHandler implements EventHandler {

    private final CqnAnalyzer analyzer;

    public BooksHandler(CdsModel cdsModel) {
        // CqnAnalyzer はコンストラクタで一度だけ生成してフィールドに保持する
        this.analyzer = CqnAnalyzer.create(cdsModel);
    }

    @Before(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
    public void handler(CdsReadEventContext context) {
        Map<String, Object> keys = analyzer.analyze(context.getCqn()).targetValues();

        String bookId        = (String)  keys.get(Books.ID);
        Boolean isActiveEntity = (Boolean) keys.get(DraftService.DRAFT_COLUMN_ISAACTIVEENTITY);
    }
}
```

**補足**  
- `targetValues()` はすべての等値キー条件を `Map<String, Object>` で返す。
- ドラフト有効化リクエスト（`draftActivate` アクション）では `IsActiveEntity = false`、アクティブエンティティへのアクセスでは `IsActiveEntity = true` が渡される。
- `targetValues()` は等値フィルタ（`field = value`）のみが対象。`startswith` や `IN` 句には対応していない。
- 複雑な条件を扱う場合は `CqnVisitor` を使う。

---

## WHERE 条件の追加

**概要**  
既存のクエリに動的に WHERE 条件を追加する。`CQL.copy()` に `Modifier` を渡して WHERE 句を差し替える。

**コード**  

```java
import com.sap.cds.ql.CQL;
import com.sap.cds.ql.cqn.Modifier;
import com.sap.cds.ql.cqn.Predicate;

@On(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void filterActiveOnly(CdsReadEventContext context) {
    CqnSelect original = context.getCqn();

    CqnSelect filtered = CQL.copy(original, new Modifier() {
        @Override
        public Predicate where(Predicate where) {
            Predicate activeFilter = CQL.get(Books.ACTIVE).eq(true);
            // 既存条件が存在する場合は AND で結合、なければそのまま使用
            return where != null ? CQL.and(where, activeFilter) : activeFilter;
        }
    });

    context.setCqn(filtered);
}
```

**補足**  
- `CQL.copy(query, modifier)` で元のクエリを変更せずコピーを生成する。
- `Modifier#where(Predicate)` の引数はコピー元の WHERE 句。`null` の場合（WHERE なし）も考慮する。
- AND ではなく OR で結合したい場合は `CQL.or(where, ...)` を使う。

> 参照: [Introspecting CQL Statements – CAP Java](https://cap.cloud.sap/docs/java/working-with-cql/query-introspection)
