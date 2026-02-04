# Guia de Calidad - JavaScript / Node.js

Aplica a los scripts utilitarios en `tools/` y componentes Node.js del proyecto.

## Estructura de Scripts

Cada script debe tener un header descriptivo con proposito y uso:

```javascript
/**
 * Mock WebSocket server para testing local del agente.
 *
 * Uso:
 *   node ws-mock-server.js
 *   PORT=9090 node ws-mock-server.js
 *
 * Escucha en ws://localhost:8080/websocket por defecto.
 * Responde a comandos 'command' y 'commandRW' con datos mock.
 */
```

## Dependencias

### package.json

- Mantener las dependencias al minimo necesario.
- Fijar versiones mayores, permitir patches: `"ws": "^8.19.0"`.
- Incluir `node_modules/` en `.gitignore`.

### Instalacion

Documentar en el script o en un README local:

```bash
cd tools/
npm install
```

## Configuracion

Usar variables de entorno con defaults para configuracion:

```javascript
const PORT = process.env.PORT || 8080;
const WS_PATH = process.env.WS_PATH || '/websocket';
```

No hardcodear puertos, hosts, ni credenciales.

## Error Handling

### WebSocket Server

Manejar eventos de error y cierre de conexion:

```javascript
const wss = new WebSocketServer({ server, path: WS_PATH });

wss.on('connection', (ws, req) => {
    const clientId = req.headers['client-id'] || 'unknown';
    console.log(`Cliente conectado: ${clientId}`);

    ws.on('error', (err) => {
        console.error(`Error en conexion ${clientId}:`, err.message);
    });

    ws.on('close', (code, reason) => {
        console.log(`Cliente desconectado: ${clientId}, code=${code}`);
    });
});

wss.on('error', (err) => {
    console.error('Error en WebSocket server:', err.message);
    process.exit(1);
});
```

### Mensajes

Validar JSON antes de procesar:

```javascript
ws.on('message', (data) => {
    let msg;
    try {
        msg = JSON.parse(data);
    } catch (e) {
        console.warn('Mensaje no es JSON valido, ignorando');
        return;
    }

    // procesar msg
});
```

## Logging

Para scripts utilitarios, `console.log`/`console.error` es suficiente. Incluir contexto relevante:

```javascript
console.log(`[${new Date().toISOString()}] Servidor iniciado en puerto ${PORT}`);
console.log(`Cliente conectado: ${clientId}`);
console.error(`Error procesando mensaje: ${err.message}`);
```

- Usar `console.log` para eventos normales.
- Usar `console.warn` para situaciones inesperadas manejadas.
- Usar `console.error` para errores.

## Buenas Practicas

### Graceful shutdown

```javascript
process.on('SIGINT', () => {
    console.log('Cerrando servidor...');
    wss.close(() => {
        console.log('Servidor cerrado');
        process.exit(0);
    });
});
```

### Evitar callbacks anidados

Preferir `async/await` cuando sea posible:

```javascript
// Preferir
async function processMessage(msg) {
    const result = await executeQuery(msg.query);
    return { status: 'ok', data: result };
}

// Evitar
function processMessage(msg, callback) {
    executeQuery(msg.query, (err, result) => {
        if (err) return callback(err);
        callback(null, { status: 'ok', data: result });
    });
}
```

### Constantes

Agrupar constantes al inicio del archivo:

```javascript
const DEFAULT_PORT = 8080;
const WS_PATH = '/websocket';
const PING_MESSAGE = 'ping';
```

## Testing

Para scripts utilitarios simples no se requiere un framework de testing completo, pero al menos documentar como verificar que funciona:

```bash
# Iniciar el mock server
node ws-mock-server.js

# En otra terminal, probar conexion
wscat -c ws://localhost:8080/websocket -H "client-id: test"
```

Si los scripts crecen en complejidad, considerar:
- **Jest** o **Vitest** como framework de testing.
- Separar logica de negocio del setup del servidor para facilitar testing.
