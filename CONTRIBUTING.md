# Guia de Contribucion

## Branching Strategy

```
main        <- produccion, solo recibe merges de test
test        <- ambiente de QA, recibe merges de development
development <- rama de integracion, los PRs van aca
```

Para cada tarea:

```bash
git checkout development
git pull origin development
git checkout -b feat/PROJ-XXX-descripcion-corta
```

Prefijos de branch: `feat/`, `fix/`, `refactor/`, `hotfix/`, `chore/`

## Formato de Commits

Conventional commits:

```
tipo(scope): descripcion breve [PROJ-XXX]

- Detalle del cambio 1
- Detalle del cambio 2

Refs: PROJ-XXX
```

**Tipos:** `feat` | `fix` | `refactor` | `docs` | `style` | `test` | `chore`

## Proceso de PR

1. Crear branch desde `development`
2. Hacer los cambios y commits
3. Push de la branch: `git push -u origin feat/PROJ-XXX-descripcion`
4. Crear PR hacia `development`
5. Esperar review y aprobacion
6. Merge (squash o merge commit segun el caso)

## Guias de Calidad

Ver las guias detalladas en el repositorio [.github](https://github.com/diegutierrez/.github):

- [Calidad de codigo (universal)](https://github.com/diegutierrez/.github/blob/main/docs/quality-code.md)
- [Java / Spring Boot](https://github.com/diegutierrez/.github/blob/main/docs/quality-java.md)
- [JavaScript / Node.js](https://github.com/diegutierrez/.github/blob/main/docs/quality-javascript.md)
- [Python](https://github.com/diegutierrez/.github/blob/main/docs/quality-python.md)
