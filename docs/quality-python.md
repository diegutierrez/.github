# Guia de Calidad - Python

Aplica a componentes Python del proyecto (flexipaas-rules-python y futuros).

## Estructura de Proyecto

```
mi-componente/
  src/
    __init__.py
    main.py
    models.py
    services/
      __init__.py
      query_service.py
  tests/
    __init__.py
    test_main.py
    test_query_service.py
  requirements.txt       # dependencias pinned
  pyproject.toml          # metadata del proyecto (alternativa moderna)
  .python-version         # version de Python
  README.md
```

### Dependencias

Usar `requirements.txt` con versiones fijadas para reproducibilidad:

```
websockets==12.0
pydantic==2.5.0
```

Para proyectos mas complejos, preferir `pyproject.toml`:

```toml
[project]
name = "flexipaas-rules-python"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
    "websockets>=12.0",
    "pydantic>=2.5.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "ruff>=0.1.0",
]
```

## Type Hints

Usar type hints en funciones publicas y parametros de entrada:

```python
from typing import Optional

def execute_query(sql: str, params: Optional[dict] = None) -> list[dict]:
    """Ejecuta una query SQL y retorna los resultados como lista de diccionarios."""
    ...

def connect(host: str, port: int = 8080) -> bool:
    """Establece conexion WebSocket."""
    ...
```

No es necesario anotar cada variable local, pero si los contratos de funciones.

## Manejo de Excepciones

### No silenciar excepciones

```python
# Correcto
try:
    result = db.execute(query)
except DatabaseError as e:
    logger.error("Error ejecutando query: %s", e)
    raise

# Incorrecto
try:
    result = db.execute(query)
except Exception:
    pass  # NO - silencia el error
```

### Excepciones custom cuando agreguen valor

```python
class QueryExecutionError(Exception):
    """Error al ejecutar una query contra la base de datos."""

    def __init__(self, query: str, original_error: Exception):
        self.query = query
        self.original_error = original_error
        super().__init__(f"Error ejecutando query: {original_error}")
```

### Context managers para recursos

```python
# Usar context managers para cleanup automatico
with open(filepath, "r") as f:
    data = json.load(f)

# Para conexiones de base de datos
with db.connection() as conn:
    result = conn.execute(query)
```

## Virtual Environments

Usar siempre un virtual environment. No instalar dependencias globalmente:

```bash
# Crear
python -m venv .venv

# Activar
source .venv/bin/activate   # Linux/macOS
.venv\Scripts\activate      # Windows

# Instalar dependencias
pip install -r requirements.txt

# Para desarrollo
pip install -r requirements.txt -r requirements-dev.txt
```

Agregar `.venv/` al `.gitignore`.

## Logging

Usar el modulo `logging` estandar, no `print()`:

```python
import logging

logger = logging.getLogger(__name__)

def process_message(msg: dict) -> dict:
    logger.info("Procesando mensaje tipo=%s", msg.get("type"))
    try:
        result = execute(msg)
        logger.debug("Resultado: %d filas", len(result))
        return result
    except Exception as e:
        logger.error("Error procesando mensaje: %s", e, exc_info=True)
        raise
```

### Configuracion basica

```python
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
```

Para logs estructurados (JSON), usar `python-json-logger`:

```python
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter())
logger.addHandler(handler)
```

## Testing

### pytest

```python
# tests/test_query_service.py
import pytest
from src.services.query_service import execute_query, QueryExecutionError

def test_execute_select_returns_results():
    result = execute_query("SELECT 1 as value")
    assert len(result) == 1
    assert result[0]["value"] == 1

def test_execute_invalid_query_raises():
    with pytest.raises(QueryExecutionError):
        execute_query("INVALID SQL")

@pytest.fixture
def mock_db(mocker):
    return mocker.patch("src.services.query_service.db")

def test_query_uses_connection(mock_db):
    execute_query("SELECT 1")
    mock_db.connection.assert_called_once()
```

### Ejecutar tests

```bash
# Todos los tests
pytest

# Con coverage
pytest --cov=src --cov-report=term-missing

# Un archivo especifico
pytest tests/test_query_service.py -v
```

## Linting y Formateo

### Ruff (recomendado)

Ruff reemplaza flake8, isort, y black en una sola herramienta:

```toml
# pyproject.toml
[tool.ruff]
target-version = "py39"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "UP"]
```

```bash
# Verificar
ruff check .

# Formatear
ruff format .
```

### Pre-commit (opcional)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
      - id: ruff-format
```

## Seguridad

- No hardcodear credenciales. Usar variables de entorno:
  ```python
  import os
  db_password = os.environ.get("DB_PASSWORD", "")
  ```
- No loguear datos sensibles.
- Validar inputs de fuentes externas.
- Usar parametros bindeados en queries SQL, nunca string concatenation:
  ```python
  # Correcto
  cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

  # Incorrecto - SQL injection
  cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
  ```
