# .github

Repositorio central con guias de calidad de codigo, templates, convenciones y **workflows reutilizables**.

## Workflows Reutilizables

| Workflow | Descripcion | Uso |
|----------|-------------|-----|
| [python-test.yml](.github/workflows/python-test.yml) | CI: lint (ruff, mypy), tests (pytest), security | `uses: diegutierrez/.github/.github/workflows/python-test.yml@main` |
| [python-deploy.yml](.github/workflows/python-deploy.yml) | CD: deploy a Render, health check, notificacion Telegram | `uses: diegutierrez/.github/.github/workflows/python-deploy.yml@main` |
| [dependabot-auto.yml](.github/workflows/dependabot-auto.yml) | Auto-merge de PRs de Dependabot (patch/minor) | `uses: diegutierrez/.github/.github/workflows/dependabot-auto.yml@main` |

### Ejemplo: CI/CD completo

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    uses: diegutierrez/.github/.github/workflows/python-test.yml@main
    with:
      python-version: "3.11"
      coverage-threshold: 80
```

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  ci:
    uses: diegutierrez/.github/.github/workflows/python-test.yml@main

  deploy:
    needs: ci
    uses: diegutierrez/.github/.github/workflows/python-deploy.yml@main
    with:
      service-name: "my-app"
    secrets:
      RENDER_DEPLOY_HOOK_URL: ${{ secrets.RENDER_DEPLOY_HOOK_URL }}
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
```

---

## Guias de Calidad

| Guia | Descripcion |
|---|---|
| [quality-code.md](docs/quality-code.md) | Principios universales: SOLID, resiliencia, seguridad, observabilidad |
| [quality-java.md](docs/quality-java.md) | Java / Spring Boot: DI, excepciones, logging, Micrometer, C3P0 |
| [quality-javascript.md](docs/quality-javascript.md) | JavaScript / Node.js: error handling, WebSocket, estructura |
| [quality-python.md](docs/quality-python.md) | Python: type hints, pytest, ruff, venvs |

## Templates

| Archivo | Descripcion |
|---|---|
| [PULL_REQUEST_TEMPLATE.md](PULL_REQUEST_TEMPLATE.md) | Template estandar para PRs con checklist de calidad |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Branching strategy, formato de commits, proceso de PR |

## Uso

Referenciar desde cualquier repositorio:

```markdown
Ver [guias de calidad](https://github.com/diegutierrez/.github/tree/main/docs)
```
