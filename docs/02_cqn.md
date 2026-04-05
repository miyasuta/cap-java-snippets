# CQN / クエリ操作

## 目次

- [CqnAnalyzer によるキー抽出](#cqnanalyzer-によるキー抽出)
- [WHERE 条件の追加](#where-条件の追加)
- [CqnVisitor による述語の走査](#cqnvisitor-による述語の走査)

---

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

---

## CqnVisitor による述語の走査

**概要**  
`startswith`（前方一致）や `IN`（複数値）など、`CqnAnalyzer.targetValues()` では取得できない複雑なフィルタ条件を扱うときは `CqnVisitor` を使う。WHERE 句のツリーを再帰的に走査し、対象フィールドの述語を収集できる。

**コード**  

```java
import com.sap.cds.ql.cqn.CqnComparisonPredicate;
import com.sap.cds.ql.cqn.CqnContainmentTest;
import com.sap.cds.ql.cqn.CqnElementRef;
import com.sap.cds.ql.cqn.CqnLiteral;
import com.sap.cds.ql.cqn.CqnVisitor;

@Component
@ServiceName(CatalogService_.CDS_NAME)
public class BooksHandler implements EventHandler {

    /**
     * title を参照する述語を収集する Visitor。
     * CqnConnectivePredicate（AND/OR）はデフォルト実装が子ノードを自動走査するため override 不要。
     */
    private static class TitleFilterVisitor implements CqnVisitor {

        private final List<Object> predicates = new ArrayList<>();

        List<Object> getPredicates() {
            return predicates;
        }

        // "title = 'value'" のような等値比較を収集
        @Override
        public void visit(CqnComparisonPredicate comp) {
            if (comp.left() instanceof CqnElementRef ref
                    && Books.TITLE.equals(ref.displayName())) {
                predicates.add(comp);
            }
        }

        // "startswith(title, 'prefix')" や "title in ('a','b')" を収集
        @Override
        public void visit(CqnContainmentTest ct) {
            if (ct.value() instanceof CqnElementRef ref
                    && Books.TITLE.equals(ref.displayName())) {
                predicates.add(ct);
            }
        }
    }

    @Before(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
    public void inspectFilter(CdsReadEventContext context) {
        // where() は Optional — WHERE 句がない場合も安全に処理する
        context.getCqn().where().ifPresentOrElse(where -> {
            TitleFilterVisitor visitor = new TitleFilterVisitor();
            where.accept(visitor);
            List<Object> predicates = visitor.getPredicates();
            // 収集した述語を使って処理を分岐する
            if (predicates.size() == 1 && predicates.get(0) instanceof CqnContainmentTest ct
                    && ct.position() == CqnContainmentTest.Position.START) {
                // startswith パターン
                String prefix = (String) ((CqnLiteral<?>) ct.term()).value();
                // ... prefix を使った処理 ...
            }
        }, () -> {
            // WHERE 句なし（全件取得）の場合の処理
        });
    }
}
```

**補足**  
- `CqnConnectivePredicate`（AND / OR）のデフォルト実装は子ノードを自動的に再帰走査する。手動で再帰する必要はない。
- `CqnComparisonPredicate` — 等値比較（`=`）や `IS NULL` など。`operator()` で演算子を確認できる。
- `CqnContainmentTest` — `startswith` や `IN` 句に対応。`position()` で前方一致（`START`）か判定できる。
- `CqnAnalyzer.targetValues()` は等値フィルタのみ対応。`startswith` / `IN` / `OR` を含む条件は `CqnVisitor` を使う。

> 参照: [Introspecting CQL Statements – CAP Java](https://cap.cloud.sap/docs/java/working-with-cql/query-introspection)
