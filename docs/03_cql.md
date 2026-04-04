# CQL / データアクセス

## Select の基本形

**概要**  
`PersistenceService` を使ってエンティティを型安全に読み取る。

**コード**  

```java
@Autowired @Qualifier(PersistenceService.DEFAULT_NAME)
private PersistenceService db;

// 全件取得
List<Books> allBooks = db.run(Select.from(Books_.class)).listOf(Books.class);

// キーで1件取得
Optional<Books> book = db.run(
    Select.from(Books_.class).where(b -> b.ID().eq(bookId))
).first(Books.class);

// 条件付き取得
List<Books> cheapBooks = db.run(
    Select.from(Books_.class).where(b -> b.price().lt(new BigDecimal("20.00")))
).listOf(Books.class);
```

**補足**  
- `PersistenceService` はサービスハンドラをバイパスして直接 DB にアクセスする。
- サービスハンドラを通す場合は対象サービスの interface を `@Autowired` で注入する。

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
