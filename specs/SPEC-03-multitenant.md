# SPEC-03: Multi-Tenant Infrastructure

## Mục tiêu
Implement schema-per-tenant isolation. Mỗi request tự động chạy SQL trong đúng schema của tenant mà không cần filter tenant_id trong query.

## Dependencies
- SPEC-01 (project setup)
- SPEC-02 (database schema — biết format schema name)

## Files cần tạo

### 1. `src/main/java/com/spax/tenant/TenantContext.java`

ThreadLocal holder cho tenant code hiện tại của request.

```java
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setTenantCode(String tenantCode) { CURRENT_TENANT.set(tenantCode); }
    public static String getTenantCode() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); }

    // Schema name = "tenant_" + code
    public static String getSchemaName() {
        String code = getTenantCode();
        if (code == null) throw new IllegalStateException("No tenant in context");
        return "tenant_" + code;
    }
}
```

### 2. `src/main/java/com/spax/tenant/TenantInterceptor.java`

Spring `HandlerInterceptor`. Extract tenant từ:
1. Path variable: `/api/v1/{tenant}/...` hoặc `/dicomweb/{tenant}/...`
2. Header: `X-Tenant-ID`

Priority: path variable > header.

```java
@Component
public class TenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String tenantCode = extractTenantCode(request);
        if (tenantCode != null) {
            TenantContext.setTenantCode(tenantCode);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        TenantContext.clear();
    }

    private String extractTenantCode(HttpServletRequest request) {
        // Try to extract from URI path: /api/v1/{tenant}/... or /dicomweb/{tenant}/...
        String uri = request.getRequestURI();
        // Pattern: /api/v1/{tenant}/ or /dicomweb/{tenant}/
        // Extract segment at position 3 for /api/v1/{tenant} or position 2 for /dicomweb/{tenant}
        String[] parts = uri.split("/");
        if (parts.length >= 4 && "api".equals(parts[1]) && "v1".equals(parts[2])) {
            return parts[3]; // /api/v1/{tenant}/...
        }
        if (parts.length >= 3 && "dicomweb".equals(parts[1])) {
            return parts[2]; // /dicomweb/{tenant}/...
        }
        // Fallback to header
        return request.getHeader("X-Tenant-ID");
    }
}
```

**Lưu ý**: Paths `/api/v1/admin/...` (system admin không cần tenant) và `/api/v1/health` không cần tenant context — TenantContext sẽ là null, không sao vì các endpoint này không query tenant schema.

### 3. `src/main/java/com/spax/tenant/TenantSchemaResolver.java`

Implement Hibernate `CurrentTenantIdentifierResolver` và `MultiTenantConnectionProvider` để tự động chạy `SET search_path` trên mỗi connection.

**Approach**: Dùng Spring `AbstractRoutingDataSource` với connection wrapper:

```java
@Component
public class TenantSchemaResolver {
    // Called by DataSourceConfig to wrap connections
    public Connection wrapConnection(Connection connection) throws SQLException {
        return new ConnectionWrapper(connection) {
            // Override prepareStatement/createStatement để set search_path trước khi execute
        };
    }
}
```

**Recommended approach** (simpler): Dùng Hibernate's `MultiTenantConnectionProvider`:

```java
@Component
public class TenantConnectionProvider implements MultiTenantConnectionProvider<String> {
    @Autowired private DataSource dataSource;

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection conn = dataSource.getConnection();
        String schema = "tenant_" + tenantIdentifier + ", public";
        conn.createStatement().execute("SET search_path TO " + schema);
        return conn;
    }
    // ...
}

@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver<String> {
    @Override
    public String resolveCurrentTenantIdentifier() {
        String code = TenantContext.getTenantCode();
        return code != null ? code : "public"; // fallback for system operations
    }
    @Override
    public boolean validateExistingCurrentSessions() { return false; }
}
```

**DataSourceConfig** (`src/main/java/com/spax/config/DataSourceConfig.java`):
```java
@Configuration
public class DataSourceConfig {
    @Bean
    public JpaVendorAdapter jpaVendorAdapter() { ... }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            TenantConnectionProvider connectionProvider,
            TenantIdentifierResolver identifierResolver) {

        Map<String, Object> props = new HashMap<>();
        props.put(AvailableSettings.MULTI_TENANT_CONNECTION_PROVIDER, connectionProvider);
        props.put(AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER, identifierResolver);
        props.put(AvailableSettings.MULTI_TENANT, MultiTenancyStrategy.SCHEMA);
        // ...
    }
}
```

### 4. `src/main/java/com/spax/tenant/TenantManagementService.java`

Service tạo tenant mới:

```java
@Service
@Transactional
public class TenantManagementService {

    @Autowired private JdbcTemplate jdbcTemplate;  // uses public schema connection

    public void createTenant(String code, String name) {
        // 1. Insert into public.tenant
        jdbcTemplate.update("INSERT INTO public.tenant (code, name) VALUES (?, ?)", code, name);

        // 2. Create schema
        jdbcTemplate.execute("CREATE SCHEMA IF NOT EXISTS tenant_" + sanitize(code));

        // 3. Execute V1__init.sql against new schema
        String ddl = loadTenantInitSql();
        jdbcTemplate.execute("SET search_path TO tenant_" + sanitize(code));
        // Split by ";" and execute each statement
        Arrays.stream(ddl.split(";"))
              .map(String::trim)
              .filter(s -> !s.isEmpty())
              .forEach(jdbcTemplate::execute);

        // 4. Reset search_path
        jdbcTemplate.execute("SET search_path TO public");

        // 5. Create initial partitions (12 months ahead)
        createInitialPartitions(code);
    }

    private String loadTenantInitSql() {
        // Load from classpath: com/spax/schema/migration/V1__init.sql
        return new ClassPathResource("com/spax/schema/migration/V1__init.sql")
                   .getContentAsString(StandardCharsets.UTF_8);
    }

    private void createInitialPartitions(String tenantCode) {
        // Create instance partitions for next 12 months
        // Pattern: instance_YYYY_MM
        LocalDate now = LocalDate.now();
        for (int i = 0; i <= 12; i++) {
            LocalDate month = now.plusMonths(i);
            String partitionName = "instance_" + month.format(DateTimeFormatter.ofPattern("yyyy_MM"));
            String firstDay = month.withDayOfMonth(1).toString();
            String firstDayNextMonth = month.plusMonths(1).withDayOfMonth(1).toString();

            jdbcTemplate.execute("""
                SET search_path TO tenant_%s;
                CREATE TABLE IF NOT EXISTS %s
                    PARTITION OF instance
                    FOR VALUES FROM ('%s') TO ('%s');
                """.formatted(sanitize(tenantCode), partitionName, firstDay, firstDayNextMonth));
        }
    }

    private String sanitize(String code) {
        // Only allow alphanumeric + underscore to prevent SQL injection
        if (!code.matches("[a-zA-Z0-9_]+")) {
            throw new IllegalArgumentException("Invalid tenant code: " + code);
        }
        return code;
    }
}
```

### 5. `src/main/java/com/spax/config/WebConfig.java`

Đăng ký `TenantInterceptor`:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private TenantInterceptor tenantInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(tenantInterceptor)
                .addPathPatterns("/api/v1/**", "/dicomweb/**");
    }
}
```

## Lưu ý quan trọng
- `TenantManagementService.createTenant()` cần raw JDBC connection để `SET search_path` (JPA/Hibernate sẽ dùng tenant connection có search_path đã đặt sẵn)
- `sanitize()` là bắt buộc — tenant code được dùng trong `SET search_path` (dynamic SQL)
- `V1__init.sql` phải được đặt trong `src/main/resources/com/spax/schema/migration/` để loadable từ classpath
- Khi schema không tìm thấy → trả lỗi 404 "Tenant not found" thay vì 500

## Kiểm tra thành công
- `POST /api/v1/admin/tenants {"code": "test", "name": "Test"}` → tạo schema `tenant_test`
- `psql -c "\dn"` → thấy schema `tenant_test`
- `psql -c "SET search_path TO tenant_test; \dt"` → thấy tables patient, study, series, instance, ...
- Request với `X-Tenant-ID: test` → query chạy trong schema `tenant_test`
