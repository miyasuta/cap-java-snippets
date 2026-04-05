# イベントハンドラ

## 目次

- [ハンドラの基本](#ハンドラの基本)
  - [@Before / @On / @After の使い分け](#before--on--after-の使い分け)
  - [複数エンティティへの一括登録](#複数エンティティへの一括登録)
- [返り値とレスポンスの組み立て](#返り値とレスポンスの組み立て)
  - [ハンドラの返り値スタイル（June 2025 以降の推奨）](#ハンドラの返り値スタイルjune-2025-以降の推奨)
  - [ResultBuilder — イベント別の結果の組み立て方](#resultbuilder--イベント別の結果の組み立て方)
- [ドラフト](#ドラフト)
  - [ドラフトイベント](#ドラフトイベント)
  - [ドラフト保存前バリデーション](#ドラフト保存前バリデーション)

---

## ハンドラの基本

### @Before / @On / @After の使い分け

**概要**  
`@Before` は入力バリデーション、`@On` はメイン処理の差し替え、`@After` は結果への後処理に使う。

**コード**  

```java
// 例: Books エンティティへの CREATE イベント

// @Before: 常に void。入力チェックやコンテキスト書き換えのみ
@Before(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
public void validateOnCreate(CdsCreateEventContext context) {
    // 入力チェックなど
}

// @On（返り値あり）: return で値を返す（June 2025 以降の推奨スタイル）
@On(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
public Books handleCreate(CdsCreateEventContext context) {
    // デフォルト処理を差し替え、結果を return する
    Books created = doCreate(context);
    return created; // ランタイムが自動で setResult + setCompleted を処理する
}

// @On（返り値なし）: setCompleted() のみ呼ぶ
@On(event = CdsService.EVENT_DELETE, entity = Books_.CDS_NAME)
public void handleDelete(CdsDeleteEventContext context) {
    doDelete(context);
    context.setCompleted();
}

// @After: 常に void。引数で受け取ったエンティティをインプレースで書き換える
@After(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
public void enrichAfterRead(CdsReadEventContext context, List<Books> books) {
    books.forEach(b -> b.setTitle(b.getTitle() + " (enriched)"));
    // return 不要。books への変更がそのまま結果に反映される
}
```

**補足**  
- 同一フェーズに複数のハンドラを登録可能。実行順は `@HandlerOrder` で制御する。
- `@On` で返り値も `setCompleted()` も呼ばないと、後続の `@On` ハンドラ（デフォルト処理含む）も実行される。

---

### 複数エンティティへの一括登録

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

## 返り値とレスポンスの組み立て

### ハンドラの返り値スタイル（June 2025 以降の推奨）

**概要**  
CAP Java June 2025 リリースで `@On` ハンドラは `return` で値を返すスタイルが正式に推奨された。フェーズによって使い分けが異なる。

| フェーズ | 返り値 | 理由 |
|---|---|---|
| `@Before` | 常に `void` | 返り値の概念がない。context 書き換えまたは例外スローのみ |
| `@On`（返り値あり・単一エンティティ） | `return 値` | ランタイムが自動で `setResult` + `setCompleted` を処理する |
| `@On`（返り値あり・複数エンティティ） | `void` + `setResult()` + `setCompleted()` | 型が定まらないため `return` を使えない |
| `@On`（返り値なし） | `void` + `setCompleted()` | 完了を明示的に伝える必要がある |
| `@After` | 常に `void` | 引数で受け取ったエンティティをインプレースで書き換える |

**コード**  

```java
// ✅ @On（返り値あり）: return スタイル（推奨）
@On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public List<Books> readBooks(CdsReadEventContext context) {
    return db.run(context.getCqn()).listOf(Books.class);
}

// ✅ @On（返り値なし）: setCompleted() を明示
@On(event = CqnService.EVENT_DELETE, entity = Books_.CDS_NAME)
public void deleteBook(CdsDeleteEventContext context) {
    db.run(context.getCqn());
    context.setCompleted();
}

// ✅ @After: void + インプレース変更
@After(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public void discountBooks(Stream<Books> books) {
    books.filter(b -> b.getStock() > 111)
         .forEach(b -> b.setTitle(b.getTitle() + " -- 11% discount"));
}

// ✅ @On（複数エンティティ）: return の型が定まらないため void + setResult
@On(event = CqnService.EVENT_READ, entity = { Books_.CDS_NAME, Authors_.CDS_NAME })
public void readMultiple(CdsReadEventContext context) {
    Result result = db.run(context.getCqn());
    context.setResult(result);
    context.setCompleted();
}

// ❌ レガシースタイル（単一エンティティで使うのは冗長）
@On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public void readBooksLegacy(CdsReadEventContext context) {
    List<Books> result = db.run(context.getCqn()).listOf(Books.class);
    context.setResult(result);   // 単一エンティティなら return で代替できる
    context.setCompleted();      // 単一エンティティなら return で代替できる
}
```

**補足**  
- `@After` ハンドラは `Stream<Books>` または `List<Books>` を引数で受け取り、そのオブジェクトを直接変更するとレスポンスに反映される。
- `@On` で `return` を使うと `setCompleted()` の呼び忘れがなくなる。
- 複数エンティティを同一の `@On` ハンドラで扱う場合、返り値の型を一つに宣言できないため `void` + `setResult()` + `setCompleted()` を使う。

> 参照: [Event Handler Enhancements (June 2025) – CAP Java](https://cap.cloud.sap/docs/releases/2025/jun25) / [Event Handlers – CAP Java](https://cap.cloud.sap/docs/java/event-handlers/)

---

### ResultBuilder — イベント別の結果の組み立て方

**概要**  
`@On` ハンドラで結果を自前で組み立てる場合、`ResultBuilder` を使って正しい形式の `Result` を返す。イベントの種類によって使うメソッドが異なる。

| イベント | ResultBuilder メソッド | 備考 |
|---|---|---|
| READ（通常） | `selectedRows(rows).result()` | 行リストを返す |
| READ（`/$count`） | `List.of(Map.of("count", n))` | `isCountQuery()` で検出する |
| READ（`$count=true`） | `selectedRows(rows).inlineCount(n).result()` | 行リストにカウントを添付する |
| CREATE | `selectedRows(rows).result()` | READ と同じメソッドを使う |
| UPDATE | `updatedRows(count, data).result()` | 更新件数 ＋ 任意で更新後データ |
| DELETE | `deletedRows(count).result()` | 削除件数のみ |

**コード**

```java
import static com.sap.cds.ResultBuilder.selectedRows;
import static com.sap.cds.ResultBuilder.updatedRows;
import static com.sap.cds.ResultBuilder.deletedRows;
import com.sap.cds.Result;
import com.sap.cds.services.cds.CqnAnalyzer;

// READ（通常）: 行リストを返す
@On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public Result readBooks(CdsReadEventContext context) {
    List<Books> rows = db.run(context.getCqn()).listOf(Books.class);
    return selectedRows(rows).result();
}

// READ（/$count）: CqnAnalyzer.isCountQuery() で検出し、count だけを返す
@On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public List<Map<String, Object>> readBooks(CdsReadEventContext context) {
    if (CqnAnalyzer.isCountQuery(context.getCqn())) {
        long count = db.run(context.getCqn()).rowCount();
        return List.of(Map.of("count", count));
    }
    return db.run(context.getCqn()).listOf(Books.class);
}

// READ（$count=true）: 行リストにインラインカウントを添付する
@On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
public Result readBooksWithCount(CdsReadEventContext context) {
    Result result = db.run(context.getCqn());
    return selectedRows(result).inlineCount(result.rowCount()).result();
}

// CREATE: selectedRows で挿入後の行を返す（READ と同じメソッド）
@On(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
public Result createBook(CdsCreateEventContext context) {
    Result result = db.run(context.getCqn());
    return selectedRows(result).result();
}

// UPDATE: updatedRows で更新件数と更新後データを返す
@On(event = CqnService.EVENT_UPDATE, entity = Books_.CDS_NAME)
public Result updateBook(CdsUpdateEventContext context) {
    Result result = db.run(context.getCqn());
    return updatedRows((int) result.rowCount(), result.first().orElse(null)).result();
}

// DELETE: deletedRows で削除件数だけを返す
@On(event = CqnService.EVENT_DELETE, entity = Books_.CDS_NAME)
public void deleteBook(CdsDeleteEventContext context) {
    Result result = db.run(context.getCqn());
    context.setResult(deletedRows((int) result.rowCount()).result());
    context.setCompleted();
}
```

**補足**
- `/$count` (`GET /Books/$count`) と `$count=true` (`GET /Books?$count=true`) は別物。前者はカウントのみを返し、後者は行リストとカウントを同時に返す。
- `isCountQuery()` は `/$count` リクエストのみを検出する。`$count=true` の場合は CQN の `count` フラグで判断する（通常はフレームワークが自動処理するため、カスタム `@On` で全て自前実装する場合のみ意識する）。
- `updatedRows` の第2引数（更新後データ）は省略可能。更新件数だけ返せばよい場合は `updatedRows(count).result()` で可。
- デフォルトのデータベース処理に乗せる（`@Before`/`@After` のみ使う）場合は `ResultBuilder` を使う必要はない。

> 参照: [Result Handling – CAP Java](https://cap.cloud.sap/docs/java/cqn-services/application-services#result-handling)

---

## ドラフト

### ドラフトイベント

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

### ドラフト保存前バリデーション

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
