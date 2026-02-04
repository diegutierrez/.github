# Guia de Calidad - Java / Spring Boot

Aplica al agente Spring Boot 2.7.x con Java 11.

## Inyeccion de Dependencias

Usar **constructor injection**. Evitar `@Autowired` en campos.

```java
// Correcto
@Component
public class DatabaseService {
    private final JdbcTemplate jdbcTemplate;
    private final PrometheusMeterRegistry registry;

    public DatabaseService(JdbcTemplate jdbcTemplate, PrometheusMeterRegistry registry) {
        this.jdbcTemplate = jdbcTemplate;
        this.registry = registry;
    }
}

// Evitar
@Component
public class DatabaseService {
    @Autowired
    private JdbcTemplate jdbcTemplate;  // NO - field injection
}
```

Con Lombok se puede simplificar:

```java
@Component
@RequiredArgsConstructor
public class DatabaseService {
    private final JdbcTemplate jdbcTemplate;
    private final PrometheusMeterRegistry registry;
}
```

## Configuracion

### @ConfigurationProperties vs @Value

Usar `@ConfigurationProperties` para grupos de configuracion relacionada:

```java
@Component
@ConfigurationProperties(prefix = "fp.obs")
public class ObservabilityProperties {
    private Prom prom = new Prom();
    private int bufferSize = 1000;
    private int pushInterval = 30;
    // getters y setters
}
```

Usar `@Value` solo para configuraciones simples y aisladas:

```java
@Value("${app.ws.uri:ws://localhost:8080/websocket}")
private String wsUri;
```

### Valores por defecto

Toda configuracion debe tener un default para desarrollo local:

```properties
# En application.properties
fp.obs.prom.url=${PROMETHEUS_URL:${FP_OBS_PROM_URL:}}
fp.obs.push.interval=${METRICS_PUSH_INTERVAL_SECONDS:${FP_OBS_PUSH_INTERVAL:30}}
```

## Manejo de Excepciones

### No swallow exceptions

```java
// Correcto - loguear y re-lanzar
try {
    int result = jdbc.update(sql);
    return result;
} catch (DataAccessException ex) {
    log.error("Error ejecutando query RW: {}", ex.getMessage());
    queriesFailedRw.increment();
    throw ex;
}

// Incorrecto - silenciar
try {
    jdbc.update(sql);
} catch (Exception ex) {
    // silenciado - NO HACER ESTO
}
```

### Fail fast en errores criticos

Si un recurso critico no esta disponible al inicio, terminar la aplicacion:

```java
@EventListener(ApplicationReadyEvent.class)
public void validateResources() {
    try {
        FileSystemUtilities.save("test-fs.chk", "test");
        FileSystemUtilities.delete("test-fs.chk");
    } catch (IOException e) {
        log.error("No se puede escribir en el path configurado: {}", dirTmp);
        System.exit(0);
    }
}
```

### Instrumentacion de errores con AOP

Usar aspects para tracking global de errores sin contaminar el codigo de negocio:

```java
@Aspect
@Component
public class ErrorMetricsAspect {
    @Around("execution(* com.opendevpro.services..*(..)) || " +
            "execution(* com.opendevpro.handlers..*(..))")
    public Object countErrors(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            return joinPoint.proceed();
        } catch (Throwable ex) {
            errorsTotal.increment();
            Counter.builder("errors_by_type")
                .tag("error_type", ex.getClass().getSimpleName())
                .register(registry).increment();
            throw ex;
        }
    }
}
```

## Logging

### Usar Lombok @Slf4j

```java
@Slf4j
@Component
public class ClientWebSocketHandler extends TextWebSocketHandler {

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        log.debug("Mensaje recibido: tipo={}", messageType);
        log.info("Conexion WebSocket establecida, client_id={}", clientId);
        log.error("Error procesando mensaje: {}", ex.getMessage(), ex);
    }
}
```

### Niveles de log

| Nivel | Uso |
|-------|-----|
| `ERROR` | Fallo que requiere atencion (query fallo, conexion perdida sin recovery) |
| `WARN` | Situacion inesperada pero manejada (reconexion exitosa, buffer lleno) |
| `INFO` | Eventos del ciclo de vida (inicio, conexion, desconexion, shutdown) |
| `DEBUG` | Detalle de operaciones (queries ejecutadas, mensajes recibidos) |

### No loguear datos sensibles

```java
// Correcto
log.info("Conexion a base de datos establecida: url={}", jdbcUrl);

// Incorrecto
log.info("Conectando con usuario={} password={}", user, password);  // NO
```

## Metricas Micrometer

### Naming conventions

Formato: `componente_operacion_unidad`

```java
// Contadores
Counter.builder("db_queries").tag("query_type", "select").register(registry);
Counter.builder("ws_messages_received").register(registry);

// Gauges
Gauge.builder("ws_status", this, h -> h.isConnected() ? 1.0 : 0.0).register(registry);
Gauge.builder("c3p0_connections_active", ds, d -> d.getNumBusyConnectionsDefaultUser()).register(registry);

// Timers
Timer.builder("db_queries_duration").tag("query_type", "select").register(registry);
Timer.builder("ws_ping_latency").register(registry);
```

### Tags

- Usar tags de baja cardinalidad: `query_type`, `error_type`, `status`.
- No usar IDs, timestamps, o valores unicos como tags.
- Common tags se configuran una vez en el registry: `application`, `client_id`.

### Instrumentar operaciones clave

```java
long start = System.nanoTime();
try {
    Object result = jdbc.queryForObject(sql, Object.class);
    queryDuration.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
    queriesSelect.increment();
    return result;
} catch (Exception ex) {
    queryDuration.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
    queriesFailedSelect.increment();
    throw ex;
}
```

## Connection Pooling C3P0

### Configuracion recomendada

```java
pooledDataSource.setMaxPoolSize(maxPoolSize);
pooledDataSource.setMaxIdleTime(maxIdleTime);
pooledDataSource.setDebugUnreturnedConnectionStackTraces(true);  // Detectar connection leaks
pooledDataSource.setAutoCommitOnClose(false);                     // Control explicito de transacciones
pooledDataSource.setTestConnectionOnCheckin(true);                // Validar al devolver al pool
pooledDataSource.setCheckoutTimeout(15000);                       // No esperar mas de 15s
```

### Monitoreo

Instrumentar el pool con metricas:

```java
Gauge.builder("c3p0_connections_active", dataSource,
    d -> d.getNumBusyConnectionsDefaultUser()).register(registry);
Gauge.builder("c3p0_connections_idle", dataSource,
    d -> d.getNumIdleConnectionsDefaultUser()).register(registry);
```

## Scheduling

### fixedRate vs fixedDelay

- `fixedRate`: ejecutar cada N ms independientemente de la duracion. Usar para heartbeats y polling.
- `fixedDelay`: esperar N ms despues de que la ejecucion anterior termine. Usar para tareas que no deben solaparse.

```java
@Scheduled(fixedRate = 15000)   // Ping cada 15s, incluso si el anterior no termino
void sendPing() { }

@Scheduled(fixedRate = 60000)   // Check cada 60s
void checkPendingOperations() { }
```

### Error handling en tareas programadas

Las excepciones en `@Scheduled` no detienen el scheduler, pero deben loguearse:

```java
@Scheduled(fixedRate = 60000)
void checkPendingOperations() {
    try {
        // logica
    } catch (Exception e) {
        log.error("Error en check de operaciones pendientes", e);
        // el scheduler continua ejecutando
    }
}
```

## Seguridad

### Encripcion

Usar AES/GCM/NoPadding con PBKDF2 para derivacion de claves:

```java
// Parametros seguros
private static final String ENCRYPT_ALGO = "AES/GCM/NoPadding";
private static final int TAG_LENGTH_BIT = 128;
private static final int IV_LENGTH_BYTE = 12;
private static final int SALT_LENGTH_BYTE = 16;
private static final int KEY_LENGTH = 256;
private static final int ITERATION_COUNT = 65536;
```

- Generar IV y salt aleatorios para cada encripcion.
- Concatenar `IV + SALT + CIPHERTEXT` en el output.
- Usar `SecureRandom` para generar nonces.

### Passwords de base de datos

Los passwords se almacenan encriptados y se desencriptan al inicializar el DataSource:

```java
public void setPassword(String password) {
    this.password = Crypto.decrypt(password);
}
```

## Testing

### Stack recomendado

- **JUnit 5**: framework de testing
- **Mockito**: mocking de dependencias
- **Spring Boot Test**: tests de integracion

### Estructura

```
src/test/java/com/opendevpro/
  services/         # Tests de servicios
  handlers/         # Tests de WebSocket handler
  util/             # Tests de utilidades (Crypto, FileSystem)
  config/           # Tests de configuracion
```

### Ejemplo

```java
@ExtendWith(MockitoExtension.class)
class DatabaseServiceTest {

    @Mock
    private JdbcTemplate jdbcTemplate;

    @Mock
    private PrometheusMeterRegistry registry;

    @InjectMocks
    private DatabaseService databaseService;

    @Test
    void shouldExecuteSelectQuery() {
        when(jdbcTemplate.queryForObject(anyString(), eq(Object.class)))
            .thenReturn("result");

        Object result = databaseService.executeSelect("SELECT 1");

        assertNotNull(result);
        verify(jdbcTemplate).queryForObject("SELECT 1", Object.class);
    }
}
```
