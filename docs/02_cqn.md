# CQN / クエリ操作

## CqnAnalyzer によるキー抽出

**概要**  
WHERE 句からエンティティのキー値を取り出したいときに `CqnAnalyzer` を使う。等値条件のみが対象。

**コード**  

```java
@Autowired
private CdsModel cdsModel;

public void handler(CdsReadEventContext context) {
    CqnAnalyzer analyzer = CqnAnalyzer.create(cdsModel);
    AnalysisResult result = analyzer.analyze(context.getCqn());

    Object bookId = result.targetValues().get(Books.ID);
}
```

**補足**  
- `targetValues()` は等値フィルタ（`field = value`）のみを返す。`startswith` や `IN` 句には対応していない。
- 複雑な条件を扱う場合は `CqnVisitor` を使う（→ `02_cqn.md` 参照）。

---

## WHERE 条件の追加

**概要**  
既存のクエリに動的に WHERE 条件を追加する。

**コード**  

```java
@On(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void filterActiveOnly(CdsReadEventContext context) {
    CqnSelect original = context.getCqn();

    CqnSelect filtered = Select.copy(original)
        .where(b -> b.get(Books.ACTIVE).eq(true));

    context.setCqn(filtered);
}
```

**補足**  
- `Select.copy()` で既存クエリを複製してから条件を追加することで、元のクエリを壊さない。
- 複数条件を AND で結合する場合は `.where(...).and(...)` を使う。
