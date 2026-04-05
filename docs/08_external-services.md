# 外部サービス呼び出し

## 目次

- [外部 OData サービス（RemoteService API）](#外部-odata-サービスremoteservice-api)
- [外部 REST サービス（Cloud SDK + Destination）](#外部-rest-サービスcloud-sdk--destination)
- [外部呼び出し結果を内部エンティティにマージして返す](#外部呼び出し結果を内部エンティティにマージして返す)

---

## 外部 OData サービス（RemoteService API）

**概要**  
`cds import` で取り込んだ外部 OData サービスは CAP の RemoteService として登録される。`cds-maven-plugin` が生成した型付きサービスインターフェース（`ApiBusinessPartner`）をコンストラクタ注入して CQL で呼び出す。POST（Insert）の CSRF トークンは `cds-feature-remote-odata` が自動処理するため、コード側での対応は不要。

**pom.xml（子モジュール: srv/pom.xml）**

```xml
<dependency>
    <groupId>com.sap.cds</groupId>
    <artifactId>cds-feature-remote-odata</artifactId>
    <scope>runtime</scope>
</dependency>
```

**application.yaml**

```yaml
cds:
  remote.services:
    API_BUSINESS_PARTNER:
      type: odata-v2
      destination:
        name: s4-business-partner-api
      http:
        suffix: /sap/opu/odata/sap
```

**コード**

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import com.sap.cds.ql.Insert;
import com.sap.cds.ql.Select;
import com.sap.cds.ql.cqn.CqnSelect;

import cds.gen.api_business_partner.ABusinessPartner;
import cds.gen.api_business_partner.ABusinessPartner_;
import cds.gen.api_business_partner.ABusinessPartnerAddress;
import cds.gen.api_business_partner.ABusinessPartnerAddress_;
import cds.gen.api_business_partner.ApiBusinessPartner;
import cds.gen.api_business_partner.ApiBusinessPartner_;

@Component
public class BusinessPartnerClient {

    // 生成された型付きインターフェースを final フィールドで保持（コンストラクタ注入）
    private final ApiBusinessPartner bupa;

    BusinessPartnerClient(@Qualifier(ApiBusinessPartner_.CDS_NAME) ApiBusinessPartner bupa) {
        this.bupa = bupa;
    }

    // GET: 住所を1件取得
    public ABusinessPartnerAddress getAddress(String businessPartnerId) {
        CqnSelect select = Select.from(ABusinessPartnerAddress_.class)
            .columns(ABusinessPartnerAddress.ADDRESS_ID,
                     ABusinessPartnerAddress.CITY_NAME,
                     ABusinessPartnerAddress.STREET_NAME)
            .where(a -> a.BusinessPartner().eq(businessPartnerId));

        return bupa.run(select).single(ABusinessPartnerAddress.class);
    }

    // POST: Business Partner を新規作成
    // CSRF トークンは cds-feature-remote-odata が自動取得・付与するため手動処理不要
    public ABusinessPartner createPartner(String fullName) {
        ABusinessPartner partner = ABusinessPartner.create();
        partner.setBusinessPartnerFullName(fullName);
        partner.setBusinessPartnerCategory("1"); // "1" = 自然人

        return bupa.run(
            Insert.into(ABusinessPartner_.class).entry(partner)
        ).single(ABusinessPartner.class);
    }
}
```

**補足**  
- `cds-maven-plugin` が `target/generated-sources/` に `ApiBusinessPartner`（サービスインターフェース）と `ABusinessPartnerAddress_`（型クラス）を生成する。
- コンストラクタ注入＋`final` フィールドが推奨スタイル（Spring 4.3 以降は単一コンストラクタに `@Autowired` 不要）。
- `@Qualifier` の値は `application.yaml` の `cds.remote.services` のキー名（= `CDS_NAME`）と一致する。
- 結果が 0 件の場合 `single()` は例外を投げる。`first()` または `listOf()` を使うと安全。
- POST（`Insert`）・PUT（`Update`）・DELETE の CSRF トークンは `cds-feature-remote-odata` が自動で `HEAD` リクエストを発行して取得・付与する。コード側での対応は不要。
- ローカル開発時は `cds bind -2 <destination-service-instance>` で BTP の Destination サービスにバインドし、`cds bind --exec -- mvn spring-boot:run -Dspring-boot.run.profiles=hybrid` で起動する。

---

## 外部 REST サービス（Cloud SDK + Destination）

**概要**  
OData 以外の REST エンドポイントは SAP Cloud SDK の `HttpClientAccessor` と BTP Destination を使って呼び出す。`DestinationAccessor` が認証・証明書管理を透過的に処理する。

**pom.xml（親 pom: pom.xml）**

```xml
<!-- BOM インポートでバージョンを一元管理 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.sap.cloud.sdk</groupId>
            <artifactId>sdk-bom</artifactId>
            <version>use-latest-version</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**pom.xml（子モジュール: srv/pom.xml）**

```xml
<!-- バージョン指定不要（親 BOM が管理） -->
<dependency>
    <groupId>com.sap.cloud.sdk.cloudplatform</groupId>
    <artifactId>cloudplatform-connectivity</artifactId>
</dependency>
<dependency>
    <groupId>com.sap.cloud.sdk.cloudplatform</groupId>
    <artifactId>scp-cf</artifactId>
</dependency>
```

**GET コード**

```java
import org.apache.http.client.methods.HttpGet;
import org.apache.http.util.EntityUtils;
import org.springframework.stereotype.Component;

import com.sap.cloud.sdk.cloudplatform.connectivity.DestinationAccessor;
import com.sap.cloud.sdk.cloudplatform.connectivity.HttpDestination;
import com.sap.cloud.sdk.http.HttpClient;
import com.sap.cloud.sdk.http.HttpClientAccessor;

@Component
public class ShippingClient {

    public String getShippingStatus(String orderId) {
        HttpDestination destination = DestinationAccessor.getDestination("SHIPPING_API").asHttp();
        HttpClient httpClient = HttpClientAccessor.getHttpClient(destination);

        HttpGet request = new HttpGet("/v1/shipments/" + orderId + "/status");
        return httpClient.execute(request, response ->
            EntityUtils.toString(response.getEntity()));
    }
}
```

**POST コード（CSRF トークン手動処理）**

`HttpClientAccessor` は汎用 HTTP クライアントのため、CSRF トークンの取得・付与は手動で行う。
なお、CAP Java RemoteService（OData）や Cloud SDK Java の型付き VDM クライアントでは CSRF は自動処理される。

```java
import java.nio.charset.StandardCharsets;

import org.apache.http.Header;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.util.EntityUtils;
import org.springframework.stereotype.Component;

import com.sap.cloud.sdk.cloudplatform.connectivity.DestinationAccessor;
import com.sap.cloud.sdk.cloudplatform.connectivity.HttpDestination;
import com.sap.cloud.sdk.http.HttpClient;
import com.sap.cloud.sdk.http.HttpClientAccessor;

@Component
public class ShippingClient {

    public String createShipment(String requestBody) {
        HttpDestination destination = DestinationAccessor.getDestination("SHIPPING_API").asHttp();
        HttpClient httpClient = HttpClientAccessor.getHttpClient(destination);

        // Step 1: CSRF トークンを取得（HEAD または GET + X-CSRF-Token: Fetch）
        HttpGet fetchToken = new HttpGet("/v1/shipments");
        fetchToken.setHeader("X-CSRF-Token", "Fetch");

        String[] csrfToken = {null};
        String[] cookie = {null};
        httpClient.execute(fetchToken, response -> {
            Header tokenHeader = response.getFirstHeader("X-CSRF-Token");
            Header cookieHeader = response.getFirstHeader("Set-Cookie");
            if (tokenHeader != null) csrfToken[0] = tokenHeader.getValue();
            if (cookieHeader != null) cookie[0] = cookieHeader.getValue();
            return null;
        });

        // Step 2: CSRF トークンをリクエストヘッダーに付与して POST
        HttpPost post = new HttpPost("/v1/shipments");
        post.setHeader("Content-Type", "application/json");
        post.setHeader("X-CSRF-Token", csrfToken[0]);
        if (cookie[0] != null) post.setHeader("Cookie", cookie[0]);
        post.setEntity(new StringEntity(requestBody, StandardCharsets.UTF_8));

        return httpClient.execute(post, response ->
            EntityUtils.toString(response.getEntity()));
    }
}
```

**ローカル開発：`cds bind` で Destination サービスにバインド**

CAP 開発では `cds bind` で BTP の Destination サービスインスタンスにローカルからバインドするのが標準的な方法。認証方式に関わらず BTP 上の Destination 定義をそのまま使えるため、ローカル専用のコードは不要。

```bash
# Destination サービスインスタンスにバインド（~/.cdsrc-private.json に保存される）
cds bind -2 <destination-service-instance-name>

# ハイブリッドプロファイルで起動（cds bind --exec で VCAP_SERVICES を注入）
cds bind --exec -- mvn spring-boot:run -Dspring-boot.run.profiles=hybrid
```

**補足**  
- `HttpClientAccessor.getHttpClient(destination)` が返す `com.sap.cloud.sdk.http.HttpClient` は Apache `HttpClient` ではない。Apache の `HttpUriRequest` は使えるが、型は Cloud SDK のものを使う。
- CSRF 自動処理の比較: CAP Java RemoteService（OData）と Cloud SDK Java VDM 型付きクライアントは自動。`HttpClientAccessor`（汎用 REST）のみ手動対応が必要。
- REST エンドポイントが CSRF を要求しない場合（例: API Key 認証の社内 API）は Step 1 を省略できる。

---

## 外部呼び出し結果を内部エンティティにマージして返す

**概要**  
`@On READ` ハンドラで内部 DB と外部サービスのデータを結合して返すパターン。外部サービスへの呼び出しを `IN` 句でまとめることで N+1 問題を回避する。

**コード**

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import com.sap.cds.ql.Select;
import com.sap.cds.services.EventHandler;
import com.sap.cds.services.cds.CdsReadEventContext;
import com.sap.cds.services.cqn.CqnService;
import com.sap.cds.services.handler.annotations.On;
import com.sap.cds.services.handler.annotations.ServiceName;
import com.sap.cds.services.persistence.PersistenceService;

import cds.gen.api_business_partner.ABusinessPartner;
import cds.gen.api_business_partner.ABusinessPartner_;
import cds.gen.api_business_partner.ApiBusinessPartner;
import cds.gen.api_business_partner.ApiBusinessPartner_;
import cds.gen.catalogservice.CatalogService_;
import cds.gen.catalogservice.Orders;
import cds.gen.catalogservice.Orders_;

@Component
@ServiceName(CatalogService_.CDS_NAME)
public class OrdersHandler implements EventHandler {

    private final PersistenceService db;
    private final ApiBusinessPartner bupa;

    // Orders に businessPartnerId フィールドがある前提
    OrdersHandler(
            PersistenceService db,
            @Qualifier(ApiBusinessPartner_.CDS_NAME) ApiBusinessPartner bupa) {
        this.db = db;
        this.bupa = bupa;
    }

    @On(event = CqnService.EVENT_READ, entity = Orders_.CDS_NAME)
    public List<Orders> readOrders(CdsReadEventContext context) {
        // 1. 内部 DB から Orders を取得
        List<Orders> orders = db.run(context.getCqn()).listOf(Orders.class);
        if (orders.isEmpty()) {
            return orders;
        }

        // 2. businessPartnerId を収集して外部サービスを一括呼び出し（N+1 回避）
        List<String> partnerIds = orders.stream()
            .map(Orders::getBusinessPartnerId)
            .distinct()
            .collect(Collectors.toList());

        Map<String, String> partnerNames = bupa.run(
            Select.from(ABusinessPartner_.class)
                  .columns(ABusinessPartner.BUSINESS_PARTNER,
                           ABusinessPartner.BUSINESS_PARTNER_FULL_NAME)
                  .where(bp -> bp.get(ABusinessPartner.BUSINESS_PARTNER).in(partnerIds))
        ).listOf(ABusinessPartner.class).stream()
         .collect(Collectors.toMap(
             ABusinessPartner::getBusinessPartner,
             ABusinessPartner::getBusinessPartnerFullName
         ));

        // 3. マージ: 内部エンティティに外部データをセット
        orders.forEach(order ->
            order.setBusinessPartnerName(
                partnerNames.getOrDefault(order.getBusinessPartnerId(), ""))
        );

        return orders;
    }
}
```

**補足**  
- `listOf(ABusinessPartner.class)` で外部サービスの結果を型付きリストに変換する。`Map<String, Object>` にキャストしない。
- `Collectors.toMap` で ID → 名前のマップを作り、ループで O(1) ルックアップする。
- `getOrDefault` を使うと外部サービスに存在しない ID があっても NPE を防げる。
- 外部サービスが障害の場合、`bupa.run(...)` は `ServiceException` をスローする。ダウングレードして内部データのみ返す場合は try-catch でラップする。

> 参照: [Consuming Remote Services – CAP Java](https://cap.cloud.sap/docs/java/cqn-services/remote-services#consuming-remote-services)
