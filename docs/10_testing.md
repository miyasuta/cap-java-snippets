# 自動テスト

## 目次

- [テスト3レイヤーの使い分け](#テスト3レイヤーの使い分け)
- [レイヤー1: イベントハンドラ単体テスト](#レイヤー1-イベントハンドラ単体テスト)
- [レイヤー2: サービス層テスト](#レイヤー2-サービス層テスト)
- [レイヤー3: 結合テスト（OData/HTTP 越し）](#レイヤー3-結合テストodatahttp-越し)
- [テスト実行前の前提（H2 とテストデータ）](#テスト実行前の前提h2-とテストデータ)

---

## テスト3レイヤーの使い分け

**概要**  
CAP Java のテストは公式に **3 レイヤー** に整理されている。下のレイヤーほど高速・狭範囲、上のレイヤーほど低速・広範囲（テストピラミッド）。**まずレイヤー2（サービス層）を主力にし**、純粋なロジックはレイヤー1で速く、プロトコル依存の挙動だけレイヤー3で確認する、という配分が基本。

| レイヤー | 何をテストするか | 起動するもの | 主な道具 | 速度 |
|---|---|---|---|---|
| **1. ハンドラ単体** | ハンドラメソッドの純粋なロジック（値引き計算・整形など） | なし（Spring を起動しない） | JUnit5 + Mockito | 最速 |
| **2. サービス層** | CRUD / アクションの振る舞い、ハンドラチェーン全体、例外 | Spring コンテキスト + H2 | `@SpringBootTest` + 生成サービス | 中速 |
| **3. 結合（HTTP）** | OData プロトコル経由のリクエスト/レスポンス全経路 | Spring + Web 層 + H2 | `@SpringBootTest` + `MockMvc` | 低速 |

**判断の目安**  

| やりたいこと | 選ぶレイヤー |
|---|---|
| 計算ロジックや整形だけを速く検証したい | レイヤー1 |
| `service.run(...)` / アクション / 例外など業務挙動を検証したい（**最もコスパが良い**） | レイヤー2 |
| `$filter` / `$expand` などの OData クエリや認証（401/200）を検証したい | レイヤー3 |

**補足**  
- レイヤー2 はプロトコル（OData/REST）に依存しないため、URL を組み立てる必要がなく最も書きやすい。迷ったらまずレイヤー2。
- レイヤー1 は `@Before` の入力チェックや `@After` の整形のように「DB に触れない純粋ロジック」に向く。DB 経由の振る舞いを確かめたいならレイヤー2 に上げる。
- レイヤー3 は遅いので「OData プロトコルでしか確認できないこと」に絞る。件数を増やしすぎない。

> 参照: [Testing Applications – CAP Java](https://cap.cloud.sap/docs/java/developing-applications/testing)

---

## レイヤー1: イベントハンドラ単体テスト

**概要**  
Spring を起動せず、ハンドラを `new` してメソッドを直接呼ぶ。依存（`PersistenceService` など）は Mockito でモックする。DB に触れない純粋ロジックの検証に使う。

**コード**

```java
@ExtendWith(MockitoExtension.class)
public class BooksHandlerTest {

    @Mock
    private PersistenceService db; // 依存はモック（このテストでは呼ばれない）

    @Test
    public void discountShouldApplyWhenStockOver111() {
        // Arrange: 生成型エンティティを組み立てる
        Books book1 = Books.create();
        book1.setTitle("Book 1");
        book1.setStock(10);

        Books book2 = Books.create();
        book2.setTitle("Book 2");
        book2.setStock(200);

        BooksHandler handler = new BooksHandler(db);

        // Act: @After の整形ロジックを直接呼ぶ
        handler.discountBooks(Stream.of(book1, book2));

        // Assert
        assertEquals("Book 1", book1.getTitle(), "在庫111以下は値引きされない");
        assertEquals("Book 2 -- 11% discount", book2.getTitle(), "在庫111超は値引きされる");
    }
}
```

**補足**  
- `Books.create()` は `cds-maven-plugin` 生成の型付きアクセサ。`Map` を使わない。
- DB アクセスを伴うロジックはここでは検証できない（モックは値を返さない）。その場合はレイヤー2へ。

---

## レイヤー2: サービス層テスト

**概要**  
`@SpringBootTest` で Spring コンテキストを起動し、**生成されたサービスインターフェース**（`CatalogService` など）を `@Autowired` する。HTTP 層を経由せずに CQN 実行 API・アクション・例外を検証できる。ハンドラチェーン全体は通る。

**コード**

```java
@SpringBootTest
public class CatalogServiceTest {

    @Autowired
    private CatalogService catalogService; // 生成インターフェース。CdsService は使わない

    @Test
    public void readShouldReturnDiscountedTitle() {
        // Act: CQN を run（テストデータの既知の ID を使う）
        Books book = catalogService
            .run(Select.from(Books_.class).byId("51061ce3-ddde-4d70-a2dc-6314afbcc73e"))
            .single(Books.class);

        // Assert
        assertEquals("The Raven -- 11% discount", book.getTitle());
    }

    @Test
    public void submitOrderShouldReduceStock() {
        // Arrange: 型付きアクションコンテキストを生成
        SubmitOrderContext context = SubmitOrderContext.create();
        context.setBook("4a519e61-3c3a-4bd9-ab12-d7e0c5329933"); // 在庫22の書籍
        context.setQuantity(2);

        // Act: emit でアクションを実行
        catalogService.emit(context);

        // Assert
        assertEquals(22 - context.getQuantity(), context.getResult().getStock());
    }

    @Test
    public void submitOrderShouldFailWhenExceedingStock() {
        SubmitOrderContext context = SubmitOrderContext.create();
        context.setBook("4a519e61-3c3a-4bd9-ab12-d7e0c5329933");
        context.setQuantity(30); // 在庫超過

        // Assert: ServiceException が投げられる
        assertThrows(ServiceException.class, () -> catalogService.emit(context));
    }
}
```

**補足**  
- CRUD は `service.run(Select/Insert/Update/Delete...)`、アクションは `context = XxxContext.create()` → `service.emit(context)` → `context.getResult()`。
- 例外系は `assertThrows(ServiceException.class, ...)` で検証する。
- `@Autowired` には `CatalogService` のような生成インターフェースを使う。汎用の `CdsService` は使わない。

---

## レイヤー3: 結合テスト（OData/HTTP 越し）

**概要**  
`@AutoConfigureMockMvc` で `MockMvc` を有効にし、OData エンドポイントへ HTTP リクエストを投げて JSON レスポンスを検証する。プロトコルアダプタ → サービス → ハンドラ → アダプタの全経路を通る。`$filter` など OData プロトコル固有の挙動の検証に使う。

**コード**

```java
@SpringBootTest
@AutoConfigureMockMvc
public class CatalogServiceITest {

    private static final String booksURI = "/odata/v4/CatalogService/Books";

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void listShouldApplyDiscountForHighStock() throws Exception {
        mockMvc.perform(get(booksURI + "?$filter=stock gt 200&$top=1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.value[0].title").value(containsString("11% discount")));
    }

    @Test
    public void listShouldNotApplyDiscountForLowStock() throws Exception {
        mockMvc.perform(get(booksURI + "?$filter=stock lt 100&$top=1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.value[0].title").value(not(containsString("11% discount"))));
    }
}
```

**補足**  
- OData クエリオプションは `$filter` / `$top` / `$expand` などをそのまま URL に書く。
- レスポンス検証は `jsonPath("$.value[0]....")` + Hamcrest マッチャ（`containsString` / `not` など）。
- 認証付きで検証する場合は `@WithMockUser(username = "...")` を付け、`status().isUnauthorized()`（401）/`status().isOk()`（200）を確認する（`spring-boot-starter-security` がテスト classpath に必要）。
- クラス名は慣習として結合テストに `*ITest` / `*IT`、単体・サービス層に `*Test` を使い分けると区別しやすい。

---

## テスト実行前の前提（H2 とテストデータ）

**概要**  
レイヤー2・3 は実際に DB を起動するため、テスト用 DB（H2）とテストデータが必要。

**pom.xml（H2 を runtime 依存に追加）**

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

**CSV ロードは単一メカニズム（`cds.dataSource.csv.paths`）**  
CAP Java は起動時に `cds.dataSource.csv.paths` の glob で**ファイルシステムから** CSV を読み込み DB に import する。これは通常実行・テストで共通。

- パスは**プロセスの作業ディレクトリからの相対**。`mvn spring-boot:run` も Maven Surefire（`mvn test`）も既定の作業ディレクトリは**モジュールディレクトリ（`srv/`）で同じ**なので、`../db/data/**` は `<project-root>/db/data` を指す。
- `csv.paths` の**既定値が `db/data` を指している**ため、`db/data/<entity>.csv` は**追加設定なし**で読み込まれる（classpath バンドルではなくファイルシステムから読まれる）。

**⚠️ `csv.paths` は「追加」ではなく「置換」**  
`csv.paths` を明示すると既定値を**上書き**する。`test/data` だけを書くと `db/data` が読まれなくなる。両方読みたいなら**両方を列挙する**。

```yaml
# srv/src/test/resources/application.yaml（テスト専用に上書きする場合の全体例）
spring:
  config.activate.on-profile: default
  sql.init.platform: h2
cds:
  data-source.auto-config.enabled: false
  dataSource.csv.paths:
    - ../db/data/**     # ← これを省くと db/data が読まれなくなる
    - ../test/data/**   # 作業ディレクトリ(srv/)からの相対 → <project-root>/test/data
```

テスト専用に上書きするなら `srv/src/test/resources/application.yaml` に置く（テストリソースは main リソースを classpath 上で同名ファイルごと上書きするので、`mock.users` など必要な設定も併記する）。個別テスト直前のデータ投入には Spring の `@Sql` も併用できる。

> ✅ この挙動は本リポジトリの検証用プロジェクト（`verify-testdata/`、`cds init --java --add sample`、CAP Java / cds-dk 9.x）で実証済み。`db/data` に Books 5件・`test/data` に 2件を置き、`@SpringBootTest` で件数を確認した結果:
> - `csv.paths` 未設定 → 計 **5**（`test/data` は読まれない）
> - `csv.paths: [../test/data/**]` のみ → 計 **2**（`db/data` が落ちる ＝ 置換の証拠）
> - `csv.paths: [../db/data/**, ../test/data/**]` → 計 **7**（両方ロード）
>
> 起動ログ `c.s.c.s.impl.persistence.CsvDataLoader : Filling <entity> from <path>` で実際の読み込み元を確認できる。

**発行 SQL を確認したいとき（`src/test/resources/application.yaml`）**

```yaml
logging:
  level:
    com.sap.cds.persistence.sql: DEBUG
```

**実行とログ取得**

```bash
mvn test > tmp/test.log 2>&1                 # 全テスト（CAP のログは stderr なので 2>&1 必須）
mvn test -Dtest=CatalogServiceTest#submitOrderShouldReduceStock > tmp/test.log 2>&1
grep "pattern" tmp/test.log
```

**補足**  
- H2 は Java ネイティブで並行アクセス・行ロックに対応するため、ローカルテストの既定 DB として推奨。
- 実 DB（PostgreSQL / SAP HANA）に対する結合テストが必要なら Testcontainers を使い、`mvn cds:watch -DtestRun` で起動できる。
- 別 Maven モジュールに結合テストを分離したい場合: `mvn com.sap.cds:cds-maven-plugin:add -Dfeature=INTEGRATION_TEST`。

> 参照: [Testing Applications – CAP Java](https://cap.cloud.sap/docs/java/developing-applications/testing) / [Mock User Authentication – CAP](https://cap.cloud.sap/docs/guides/security/authentication)
