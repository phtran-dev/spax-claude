# SPEC-01: Project Setup (pom.xml + application.yml + SpaxApplication)

## Mục tiêu
Tạo cấu trúc Maven project từ đầu cho SPAX. Không có code nào trước đó. Đây là task đầu tiên mọi task khác đều phụ thuộc.

## Files cần tạo

### 1. `pom.xml` (project root)

**Group/Artifact**: `com.spax` / `spax` / version `1.0.0-SNAPSHOT`
**Java**: 21
**Spring Boot**: 3.4.x (dùng spring-boot-starter-parent)

**Dependencies bắt buộc:**
```xml
<!-- Web -->
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-data-redis

<!-- Database -->
org.postgresql:postgresql (runtime)
org.flywaydb:flyway-core
org.flywaydb:flyway-database-postgresql

<!-- DICOM -->
<dcm4che.version>5.31.0</dcm4che.version>
org.dcm4che:dcm4che-core:${dcm4che.version}
org.dcm4che:dcm4che-json:${dcm4che.version}
org.dcm4che:dcm4che-imageio:${dcm4che.version}
org.dcm4che:dcm4che-imageio-opencv:${dcm4che.version}

<!-- Storage -->
org.apache.jclouds:jclouds-blobstore:2.6.0
org.apache.jclouds.provider:aws-s3:2.6.0
org.apache.jclouds.provider:google-cloud-storage:2.6.0
org.apache.jclouds.provider:azureblob:2.6.0

<!-- Test -->
spring-boot-starter-test
```

**dcm4che repository** (không có trên Maven Central):
```xml
<repository>
  <id>dcm4che</id>
  <url>https://www.dcm4che.org/maven2</url>
</repository>
```

**Compiler plugin**: Java 21, enable virtual threads:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <release>21</release>
    <compilerArgs>--enable-preview</compilerArgs>
  </configuration>
</plugin>
```

### 2. `src/main/java/com/spax/SpaxApplication.java`

```java
@SpringBootApplication
public class SpaxApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpaxApplication.class, args);
    }
}
```

### 3. `src/main/resources/application.yml`

```yaml
server:
  port: 8080
  servlet:
    multipart:
      max-file-size: 2GB
      max-request-size: 10GB

spring:
  threads:
    virtual:
      enabled: true
  datasource:
    url: ${SPAX_DB_URL:jdbc:postgresql://localhost:5432/spax}
    username: ${SPAX_DB_USER:spax}
    password: ${SPAX_DB_PASSWORD:spax}
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 5000
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  data:
    redis:
      host: ${SPAX_REDIS_HOST:localhost}
      port: ${SPAX_REDIS_PORT:6379}
  flyway:
    enabled: true
    locations: classpath:db/migration

spax:
  storage:
    disk-space-threshold-mb: 5120
    default-path-template: "{now,date,yyyy/MM/dd}/{0020000D,hash}/{0020000E,hash}/{00080018,hash}"
  ingest:
    batch-size: 200
    flush-interval-ms: 2000
    consumer-threads: 4
  queue:
    type: redis
  partition:
    months-ahead: 12
```

### 4. `src/main/resources/application-dev.yml`

```yaml
spring:
  jpa:
    show-sql: false
logging:
  level:
    com.spax: DEBUG
```

## Package Structure cần tạo (empty packages, chỉ cần tồn tại)
```
src/main/java/com/spax/
├── SpaxApplication.java
├── config/
├── tenant/
├── storage/
├── lifecycle/
├── queue/
├── transfer/
├── ingest/
├── selfmanage/
├── model/
├── repository/
├── dicomweb/
├── admin/
└── compression/
src/main/resources/
├── application.yml
├── application-dev.yml
└── db/migration/          (empty, Flyway migration files sẽ thêm ở SPEC-02)
src/test/java/com/spax/
```

## Kiểm tra thành công
- `mvn clean compile` chạy không lỗi
- Không cần start app (chưa có DB config thật)
