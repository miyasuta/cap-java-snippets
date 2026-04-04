# ログ出力

## SLF4J ロガーの定義と基本出力

**概要**  
CAP Java では SLF4J + Logback を使ってログを出力する。クラスごとにロガーを定義する。

**コード**  

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class BooksHandler implements EventHandler {

    private static final Logger logger = LoggerFactory.getLogger(BooksHandler.class);

    @Before(event = CdsService.EVENT_READ, entity = Books_.CDS_NAME)
    public void beforeRead(CdsReadEventContext context) {
        logger.debug("READ request: {}", context.getCqn().toJson());
        logger.info("User {} is reading Books", context.getUserInfo().getName());
    }

    @On(event = CdsService.EVENT_CREATE, entity = Books_.CDS_NAME)
    public void onCreate(CdsCreateEventContext context) {
        try {
            // 処理
        } catch (Exception e) {
            logger.error("Failed to create book", e);
            throw new ServiceException(ErrorStatuses.SERVER_ERROR, "CREATE_FAILED", e);
        }
    }
}
```

**補足**  
- ログレベルは `src/main/resources/application.yaml` で設定する:

```yaml
logging:
  level:
    com.example.myapp: DEBUG
    com.sap.cds: INFO
```

- テスト時は `src/test/resources/application.yaml` で `DEBUG` に上書きすると詳細ログを確認できる。
- `logger.debug("value: {}", obj)` のように `{}` プレースホルダを使うと、DEBUG が無効なときに `toString()` が呼ばれず効率的。
