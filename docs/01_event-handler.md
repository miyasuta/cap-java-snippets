# イベントハンドラ

## @Before / @On / @After の使い分け

**概要**  
`@Before` は入力バリデーション、`@On` はメイン処理の差し替え、`@After` は結果への後処理に使う。

**コード**  

```java
// 例: Books エンティティへの CREATE イベント
@Before(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
public void validateOnCreate(CdsCreateEventContext context) {
    // 入力チェックなど
}

@On(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
public void handleCreate(CdsCreateEventContext context) {
    // デフォルト処理を完全に差し替える場合に実装
    context.setResult(...);
    context.setCompleted();
}

@After(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void enrichAfterRead(CdsReadEventContext context, List<Books> books) {
    // 結果セットへの加工
}
```

**補足**  
- 同一フェーズに複数のハンドラを登録可能。実行順は `@HandlerOrder` で制御する。
- `@On` で `setCompleted()` を呼ばないと後続の `@On` ハンドラ（デフォルト処理含む）も実行される。

---

## 複数エンティティへの一括登録

**概要**  
同じハンドラを複数エンティティに適用したい場合、`entity` 属性を省略するか配列で指定する。

**コード**  

```java
// すべてのエンティティに適用
@Before(event = CdsService.EVENT_CREATE)
public void beforeAnyCreate(CdsCreateEventContext context) {
    // ...
}

// 特定の複数エンティティに適用
@Before(event = { CdsService.EVENT_CREATE, CdsService.EVENT_UPDATE },
        entity = { Books_.CDS_NAME, Authors_.CDS_NAME })
public void validateCreateOrUpdate(EventContext context) {
    // ...
}
```

**補足**  
- `entity` 省略時はサービス内の全エンティティが対象になる。

---

## ドラフトイベント

**概要**  
ドラフト対応エンティティでは `DraftService` のイベント定数を使ってドラフト固有のイベントを捕捉する。

**コード**  

```java
@Before(event = DraftService.EVENT_DRAFT_PATCH, entity = Orders_.CDS_NAME)
public void onDraftPatch(CdsUpdateEventContext context, List<Orders> orders) {
    // ドラフト編集中の変更をリアルタイムに処理
}

@Before(event = DraftService.EVENT_DRAFT_SAVE, entity = Orders_.CDS_NAME)
public void onDraftSave(CdsUpdateEventContext context) {
    // 有効化（アクティブ化）直前の処理
}
```

**補足**  
- `EVENT_DRAFT_PATCH` は UI でフィールドを変更するたびに発火する（Side Effects Qualifier と組み合わせて使う）。
- `EVENT_DRAFT_SAVE` はドラフトを保存（アクティブ化）する直前に発火する。

---

## ドラフト保存前バリデーション

**概要**  
ドラフトを有効化する前に必須項目や業務ルールをチェックし、エラーがあれば保存を中断する。

**コード**  

```java
@Before(event = DraftService.EVENT_DRAFT_SAVE, entity = Orders_.CDS_NAME)
public void validateBeforeActivation(CdsUpdateEventContext context) {
    CqnSelect query = context.getCqn();
    Result result = db.run(query);

    result.listOf(Orders.class).forEach(order -> {
        if (order.getNetAmount() == null || order.getNetAmount().signum() <= 0) {
            context.getMessages().error("NET_AMOUNT_REQUIRED")
                .target("in", Orders_.class, o -> o.netAmount());
        }
    });
}
```

**補足**  
- `context.getMessages().error(...)` でフィールドレベルのエラーを指定できる。
- `.target()` を使うと Fiori Elements が対象フィールドをハイライト表示する。
