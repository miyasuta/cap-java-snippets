# 認証・認可

## UserInfo でのユーザ・ロール取得

**概要**  
リクエストユーザの ID やロールを `UserInfo` から取得する。

**コード**  

```java
@Autowired
private UserInfo userInfo;

@Before(event = CdsService.EVENT_CREATE, entity = Orders_.CDS_NAME)
public void setCreatedBy(CdsCreateEventContext context, List<Orders> orders) {
    String userId = userInfo.getName();
    boolean isAdmin = userInfo.hasRole("Admin");

    orders.forEach(order -> {
        order.setCreatedBy(userId);
        if (!isAdmin && order.getNetAmount().compareTo(APPROVAL_THRESHOLD) > 0) {
            context.getMessages().error("APPROVAL_REQUIRED");
        }
    });
}
```

**補足**  
- `userInfo.getName()` は認証プロバイダによって返す値が異なる（メールアドレス、ユーザ ID など）。
- `hasRole()` の引数は CDS の `@requires` や `@restrict` で定義したロール名。

---

## JWT カスタム属性の取得

**概要**  
BTP の XSUAA トークンに含まれるカスタムクレーム（例: `xs.user.attributes`）を取得する。

**コード**  

```java
@Autowired
private UserInfo userInfo;

public void handler(EventContext context) {
    // xs.user.attributes のカスタム属性を取得
    List<String> costCenters = userInfo.getAttributeValues("costCenter");

    // テナント ID の取得（マルチテナント環境）
    String tenant = userInfo.getTenant();
}
```

**補足**  
- `getAttributeValues()` は XSUAA のユーザ属性に対応する。xs-security.json の `attribute` 定義と一致させること。
- ローカル開発時は `application.yaml` の `cds.security.mock.users` でテスト用ユーザ属性を設定できる。
