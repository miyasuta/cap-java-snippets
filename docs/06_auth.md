# 認証・認可

## 目次

- [UserInfo でのユーザ・ロール取得](#userinfo-でのユーザロール取得)
- [JWT カスタム属性の取得](#jwt-カスタム属性の取得)

---

## UserInfo でのユーザ・ロール取得

**概要**  
リクエストユーザの ID やロールを `UserInfo` から取得する。`UserInfo` はリクエストスコープ Bean のため、シングルトンである EventHandler ではコンストラクタ注入は使えない。EventContext から取得するのが最もシンプルで確実。

**コード**  

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class OrdersHandler implements EventHandler {

    @Before(event = CdsService.EVENT_CREATE, entity = Orders_.CDS_NAME)
    public void setCreatedBy(CdsCreateEventContext context, List<Orders> orders) {
        // EventContext から直接取得する（推奨）
        UserInfo userInfo = context.getUserInfo();
        String userId = userInfo.getName();
        boolean isAdmin = userInfo.hasRole("Admin");

        orders.forEach(order -> {
            order.setCreatedBy(userId);
            if (!isAdmin && order.getNetAmount().compareTo(APPROVAL_THRESHOLD) > 0) {
                context.getMessages().error(MessageKeys.APPROVAL_REQUIRED);
            }
        });
    }
}
```

**補足**  
- `context.getUserInfo()` はすべての EventContext サブタイプで使える。
- `userInfo.getName()` は認証プロバイダによって返す値が異なる（メールアドレス、ユーザ ID など）。
- `hasRole()` の引数は CDS の `@requires` や `@restrict` で定義したロール名。
- EventContext を持たないメソッドから参照したい場合は `@Autowired UserInfo userInfo`（フィールド注入）が使える。Spring がリクエストスコープのプロキシを自動生成する。

---

## JWT カスタム属性の取得

**概要**  
BTP の XSUAA トークンに含まれるカスタムクレーム（例: `xs.user.attributes`）を取得する。

**コード**  

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class TenantHandler implements EventHandler {

    @Before(event = CqnService.EVENT_READ, entity = Orders_.CDS_NAME)
    public void handler(CdsReadEventContext context) {
        UserInfo userInfo = context.getUserInfo();

        // xs.user.attributes のカスタム属性を取得
        List<String> costCenters = userInfo.getAttributeValues("costCenter");

        // テナント ID の取得（マルチテナント環境）
        String tenant = userInfo.getTenant();
    }
}
```

**補足**  
- `getAttributeValues()` は XSUAA のユーザ属性に対応する。xs-security.json の `attribute` 定義と一致させること。
- ローカル開発時は `application.yaml` の `cds.security.mock.users` でテスト用ユーザ属性を設定できる。

> 参照: [Security – CAP Java](https://cap.cloud.sap/docs/java/security) / [Request Contexts – CAP Java](https://cap.cloud.sap/docs/java/event-handlers/request-contexts)
