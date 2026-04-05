# CQL / データアクセス

## 目次

- [PersistenceService vs ApplicationService](#persistenceservice-vs-applicationservice)
- [Select の基本形](#select-の基本形)
- [Insert の基本形](#insert-の基本形)
- [Update の基本形](#update-の基本形)
- [Delete の基本形](#delete-の基本形)
- [バルク Upsert](#バルク-upsert)

---

## PersistenceService vs ApplicationService

**概要**  
CQL を実行するサービスは目的に応じて使い分ける。

| | `PersistenceService` | `ApplicationService`（生成 IF） |
|---|---|---|
| アクセス先 | DB に直接アクセス | サービスのイベントパイプラインを経由 |
| ハンドラ実行 | **バイパス**（Before/On/After は動かない） | **実行される** |
| 認可チェック | なし | あり |
| 主な用途 | ハンドラ内での補助的な DB 操作、内部データ取得 | 別サービスへの委譲、ドラフト操作 |

**コード**

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class OrdersHandler implements EventHandler {

    private final PersistenceService db;       // DB 直接アクセス（ハンドラをバイパス）
    private final AdminService adminService;   // ApplicationService（ハンドラを通る）

    public OrdersHandler(PersistenceService db, AdminService adminService) {
        this.db = db;
        this.adminService = adminService;
    }

    @Before(event = CqnService.EVENT_CREATE, entity = Orders_.CDS_NAME)
    public void beforeCreate(List<Orders> orders) {
        // 在庫確認: DB を直接参照（CatalogService のハンドラを起動させたくない）
        Books book = db.run(Select.from(Books_.class).byId(bookId)).single(Books.class);

        // 注文確定: AdminService のハンドラ（認可・監査ログ等）を通して実行
        adminService.run(Update.entity(Orders_.class).entry(updatedOrder));
    }
}
```

**補足**  
- `PersistenceService` はサービスエンティティではなく**永続化エンティティ**を対象とする。サービス上の View プロジェクションにしか存在しない要素は使えない。
- `ApplicationService` はサービス名で識別する。cds-maven-plugin が生成した型付き IF（例: `AdminService`）があればそちらを優先する（`@Qualifier` 不要）。
- ハンドラ内で自サービスを `ApplicationService` で呼ぶと**再帰的にハンドラが動く**ので注意。

> 参照: [Persistence Services – CAP Java](https://cap.cloud.sap/docs/java/cqn-services/persistence-services) / [Application Services – CAP Java](https://cap.cloud.sap/docs/java/cqn-services/application-services)

---

## Select の基本形

**概要**  
`PersistenceService` を使ってエンティティを型安全に読み取る。

**コード**  

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class BooksHandler implements EventHandler {

    private final PersistenceService db;

    public BooksHandler(PersistenceService db) {
        this.db = db;
    }

    // 全件取得
    private List<Books> findAll() {
        return db.run(Select.from(Books_.class)).listOf(Books.class);
    }

    // キーで1件取得
    private Optional<Books> findById(String bookId) {
        return db.run(
            Select.from(Books_.class).where(b -> b.ID().eq(bookId))
        ).first(Books.class);
    }

    // 条件付き取得
    private List<Books> findCheap() {
        return db.run(
            Select.from(Books_.class).where(b -> b.price().lt(new BigDecimal("20.00")))
        ).listOf(Books.class);
    }
}
```

**補足**  
- `PersistenceService` はサービスハンドラをバイパスして直接 DB にアクセスする。
- サービスハンドラを通す場合は対象サービスの interface をコンストラクタで注入する。

---

## Insert の基本形

**概要**  
新規レコードを挿入する。

**コード**  

```java
Books newBook = Books.create();
newBook.setId(UUID.randomUUID().toString());
newBook.setTitle("Clean Code");
newBook.setStock(10);

db.run(Insert.into(Books_.class).entry(newBook));
```

**補足**  
- `Books.create()` は生成されたエンティティクラスのファクトリメソッド。
- UUID は `CDS.DataTypes.UUID` フィールドに設定する場合、CAP が自動生成することもある（`@cds.on.insert: $uuid`）。

---

## Update の基本形

**概要**  
既存レコードを更新する。

**コード**  

```java
Books patch = Books.create();
patch.setStock(5);

db.run(
    Update.entity(Books_.class)
        .data(patch)
        .where(b -> b.ID().eq(bookId))
);
```

**補足**  
- `data()` に渡したオブジェクトのうち、非 null フィールドのみが更新される。
- `entry()` を使うと全フィールドを上書きする。

---

## Delete の基本形

**概要**  
条件に一致するレコードを削除する。

**コード**  

```java
db.run(
    Delete.from(Books_.class).where(b -> b.ID().eq(bookId))
);
```

**補足**  
- 論理削除（ソフトデリート）が必要な場合は `Update` でフラグを立てる。

---

## バルク Upsert

**概要**  
複数レコードを一括で挿入または更新する。

**コード**  

```java
List<Books> books = buildBooksToUpsert();

db.run(Upsert.into(Books_.class).entries(books));
```

**補足**  
- `Upsert` はキーが存在すれば UPDATE、なければ INSERT を実行する。
- 大量データの場合はバッチサイズを考慮すること。

> 参照: [Executing CQL Statements – CAP Java](https://cap.cloud.sap/docs/java/working-with-cql/query-execution)
