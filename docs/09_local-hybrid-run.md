# ローカルハイブリッド実行

## 目次

- [hybrid プロファイル vs cloud プロファイルの使い分け](#hybrid-プロファイル-vs-cloud-プロファイルの使い分け)
- [VS Code Test Runner から実行する場合は `.env` ファイルが必要](#vs-code-test-runner-から実行する場合は-env-ファイルが必要)

---

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

---

## VS Code Test Runner から実行する場合は `.env` ファイルが必要

**概要（結論）**  
CAP Java の hybrid 実行に `.env` ファイルは **必須ではない**。`.hybrid.env` のようなファイルは「CAP Java hybrid の要件」ではなく「VSCode テストランナー連携の都合」で必要になるものにすぎない。

- **ターミナルから実行する場合**（`mvn test` / `mvn spring-boot:run`）は [前述](#hybrid-プロファイル-vs-cloud-プロファイルの使い分け)の `cds bind --exec` でラップすれば、資格情報をその場で取得して `VCAP_SERVICES` として注入するため **ファイル不要**（ディスクに残らない）。
- **VSCode の Java Test Runner UI から実行する場合のみ**、環境変数を動的注入できないという制約があるため、`.hybrid.env` に `VCAP_SERVICES` を書き出して読ませる回避策が必要になる。

**標準的なやり方: `cds bind --exec`（ファイル不要）**  
サービスバインディングをクラウドから解決し、`VCAP_SERVICES` 環境変数としてアプリケーションへ渡す。資格情報は ad-hoc に取得され、ディスクには残らない。

```bash
# アプリの hybrid 起動
cds bind --exec -- mvn spring-boot:run

# テストの hybrid 実行（ターミナルからならこれで十分）
cds bind --exec -- mvn test -Dtest=customer.cap_agent.AgentTest#testSdkPrompt
```

CAP Java のテストガイドでも、hybrid テストの手順として cds-bind の "Run CAP Java Apps with Service Bindings" を参照するよう案内されており、`cds bind --exec` が正規ルート。

**VSCode Test Runner UI の場合の回避策: `.hybrid.env`**  
VSCode の Java Test Runner（テスト実行ボタン）には環境変数を動的注入できない。UI からテストを起動したい場合に限り、`VCAP_SERVICES` を一旦ファイルへ書き出して `envFile` で読ませる。

```bash
# 1. VCAP_SERVICES をファイルに書き出す
cds bind --exec -- node -e "require('fs').writeFileSync('.hybrid.env', 'VCAP_SERVICES=' + process.env.VCAP_SERVICES, 'utf8')"
```

```gitignore
# 2. .gitignore に追加（資格情報をコミットしない）
*.env
```

```jsonc
// 3. .vscode/settings.json — テストランナーに .hybrid.env を読ませる
"java.test.config": [
  {
    "name": "cap-bound-tests",
    "workingDirectory": "${workspaceFolder}",
    "envFile": "${workspaceFolder}/.hybrid.env"
  }
]
```

> 注意: `.hybrid.env` には資格情報が平文で保存される。`.gitignore` 必須。バインディング先のローテーション後は書き出し直すこと。

**使い分け早見表**

| 実行方法 | `.env` ファイル | 備考 |
|---|---|---|
| ターミナル `cds bind --exec -- mvn test` | 不要 | 資格情報は ad-hoc 注入、ディスクに残らない |
| ターミナル `cds bind --exec -- mvn spring-boot:run` | 不要 | 同上（アプリ起動） |
| VSCode Java Test Runner UI | 必要 | 動的注入不可のため `.hybrid.env` + `envFile` |

**補足: `default-env.json`**  
もう一つの古典的なファイルベースの選択肢として `default-env.json`（`VCAP_SERVICES` を手書き／書き出して配置）もある。ただしこれも `cds bind --exec` を使うなら不要。ファイルに資格情報を残したくないなら `cds bind --exec` を優先する。

> 参照: [Testing CAP Java Applications（Hybrid Testing）](https://cap.cloud.sap/docs/java/developing-applications/testing) / [cds bind（Run CAP Java Apps with Service Bindings）](https://cap.cloud.sap/docs/tools/cds-bind)
