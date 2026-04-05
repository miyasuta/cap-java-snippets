# ローカルハイブリッド実行

## hybrid プロファイル vs cloud プロファイルの使い分け

**概要**  
`cds bind` でバインドしたクラウドサービスをローカルから使う方法は2種類ある。どちらを使うかは接続するサービスの種類によって決まる。

どちらのプロファイルも **`cds bind --exec`** でサービスバインディング情報（`VCAP_SERVICES`）を注入して起動する。

| サービス種別 | `cds bind --exec` + hybrid | `cds bind --exec` + cloud |
|---|---|---|
| 認証（XSUAA / IAS） | ✅ | ✅ |
| RemoteService（外部 OData / REST） | ✅ | ✅ |
| Messaging（Event Mesh など） | ✅ | ✅ |
| HANA Cloud（DB 接続） | ✅ | ✅ |

**`cds bind --exec` が必要な理由**  
`cds bind -2` はバインディングのメタ情報を `~/.cdsrc-private.json` に保存するだけで、サービスの認証情報は実行時に解決される。CAP Java ランタイムはこの情報を `VCAP_SERVICES` 環境変数経由で受け取ることを期待しており、`cds bind --exec` がその変換・注入を担う。`cds bind --exec` なしで起動すると `VCAP_SERVICES` が存在しないため、HANA には接続できず H2 にフォールバックし、認証もモックユーザーになる。

```
cds bind --exec -- mvn ...:
  .cdsrc-private.json → VCAP_SERVICES に変換して注入
  VCAP_SERVICES → CAP Java ランタイム → HANA / 認証 / RemoteService すべて解決

mvn ... のみ（cds bind --exec なし）:
  VCAP_SERVICES が存在しない
  → DB: H2 にフォールバック
  → 認証: モックユーザーにフォールバック
```

**コマンド**

```bash
# 事前: サービスインスタンスにバインド（初回のみ）
cds bind -2 <hana-service-instance>
cds bind -2 <xsuaa-service-instance>

# hybrid プロファイル（HANA 含む全サービス）
cds bind --exec -- mvn spring-boot:run -Dspring-boot.run.profiles=hybrid

# cloud プロファイル（本番環境に近い設定で検証したい場合）
cds bind --exec -- mvn clean install -Dspring.profiles.active=cloud
```

**補足**  
- `cds bind --exec` は `.cdsrc-private.json` の内容を `VCAP_SERVICES` 環境変数に変換してサブプロセスに注入する。HANA を含むすべてのサービスバインディングの解決に必要。
- `mvn spring-boot:run -Dspring-boot.run.profiles=hybrid`（`cds bind --exec` なし）では `VCAP_SERVICES` が注入されないため、HANA に接続できず H2 にフォールバックする。
- `cds bind -2 <instance>` はサービスキーが存在しない場合、自動で作成してから保存する。
- バインド情報は `~/.cdsrc-private.json` に保存される（プロジェクトルートではない）。コミットしない。

> 参照: [Hybrid Testing – CAP](https://cap.cloud.sap/docs/advanced/hybrid-testing) / [cds bind – CAP CLI](https://cap.cloud.sap/docs/tools/cds-bind)
