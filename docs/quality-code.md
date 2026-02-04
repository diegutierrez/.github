# Principios de Calidad de Codigo

Guia agnostica al lenguaje. Aplica a todos los proyectos.

## SOLID

### Single Responsibility

Cada clase/modulo tiene una unica razon para cambiar. Separar responsabilidades en capas claras: configuracion, handlers, servicios, observabilidad, utilidades.

### Open/Closed

Extender comportamiento sin modificar codigo existente. Usar interfaces y composicion sobre herencia.

### Dependency Inversion

Depender de abstracciones, no de implementaciones concretas. Inyectar dependencias via constructor.

## Manejo de Errores y Resiliencia

### Principios

- **No swallow exceptions**: nunca atrapar una excepcion sin hacer nada. Loguear, re-lanzar, o manejar.
- **Propagar con contexto**: cuando re-lanzas, agregar informacion relevante (que operacion fallo, con que parametros).
- **Fail fast**: si un recurso critico no esta disponible al inicio, terminar la aplicacion. No esperar a que falle en runtime.

### Patrones de resiliencia

- **Retry con backoff**: para fallos transitorios (red, timeouts). Usar backoff exponencial (ej. 2s, 4s, 8s).
- **Circuit breaker**: para servicios externos que pueden estar caidos. Evitar saturar un servicio que ya fallo.
- **Fallback**: cuando un componente falla, degradar gracefully. Acumular en buffer si el destino no responde.
- **Timeouts**: toda operacion de I/O debe tener un timeout. Conexiones DB, HTTP, WebSocket.

## Seguridad

- **No hardcodear credenciales**: usar variables de entorno o vaults.
- **No loguear datos sensibles**: passwords, tokens, datos personales. Mascarar en logs si es necesario.
- **Encriptar datos sensibles**: usar algoritmos actuales (AES/GCM, PBKDF2). No usar MD5/SHA1 para hashing de passwords.
- **Validar inputs**: toda entrada externa debe ser validada antes de usarse.
- **Principio de menor privilegio**: conectar a servicios con los permisos minimos necesarios.

## Observabilidad

### Metricas

- Instrumentar operaciones clave: queries, conexiones, errores, latencia.
- Usar naming conventions consistentes: `componente_operacion_unidad` (ejemplo: `db_queries_duration`, `ws_messages_received`).
- Agregar tags relevantes de baja cardinalidad: `query_type`, `error_type`, `status`.
- No crear metricas con cardinalidad infinita (no usar valores dinamicos como IDs o timestamps en tags).

### Logs

- Usar logs estructurados (JSON) para facilitar busqueda y analisis.
- Niveles correctos:
  - `ERROR`: algo fallo y requiere atencion.
  - `WARN`: algo inesperado pero la aplicacion puede continuar.
  - `INFO`: eventos importantes del ciclo de vida (inicio, conexion, reconexion).
  - `DEBUG`: detalle de operaciones para troubleshooting.
- Incluir contexto relevante en cada log (IDs de correlacion, tipo de operacion, duracion).
- En produccion, usar `INFO` o `WARN` como default. Nunca `TRACE` o `DEBUG`.

### Tracing

- Correlacionar operaciones end-to-end cuando sea posible (request ID, transaction UUID).

## Conexiones y Recursos

- **Connection pooling**: usar pools para conexiones de base de datos. Configurar min, max, idle timeout.
- **Cleanup**: cerrar recursos en bloques `finally` o usar try-with-resources / context managers.
- **Health checks**: validar conexiones periodicamente.
- **Timeouts de checkout**: no esperar indefinidamente por una conexion del pool.

## Configuracion Externalizada

- Toda configuracion debe ser externalizable via variables de entorno.
- Proporcionar valores por defecto sensatos para desarrollo local.
- Agrupar configuraciones relacionadas con prefijos.
- Documentar cada variable de entorno.

## Idempotencia y Durabilidad

- Las operaciones de escritura deben ser idempotentes cuando sea posible.
- Usar UUIDs para identificar transacciones.
- Persistir operaciones pendientes en disco para recovery ante reinicios.
- Implementar recovery automatico de operaciones pendientes.

## Naming Conventions

- Nombres descriptivos que revelen la intencion.
- Clases: `PascalCase` (Java), `snake_case` (Python), `camelCase` (JavaScript).
- Metodos/funciones: verbos que describan la accion.
- Variables: sustantivos que describan el contenido.
- Constantes: `SCREAMING_SNAKE_CASE`.
- No abreviar excepto convenciones ampliamente conocidas (`db`, `ws`, `tx`).
