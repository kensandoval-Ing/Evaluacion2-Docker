# ms-vehiculos – Pipeline CI/CD

Microservicio de Vehículos (Java 21, Spring Boot, MySQL) con pipeline CI/CD en GitHub Actions.
Evaluación Parcial N°2 – Ingeniería DevOps (DOY0101).

**Integrantes:** _(completar)_

## 1. Arquitectura

| Componente | Descripción |
|---|---|
| `ms-vehiculos` | API REST (puerto 9004), imagen Docker multi-etapa |
| `mysql` | Base de datos MySQL 8.4 con volumen persistente |
| Orquestación | Docker Compose (healthchecks, reinicio automático, límites de CPU/memoria) |

## 2. Contenedores (IE1)

- `Dockerfile` multi-etapa: compila con `maven:3.9-eclipse-temurin-21` y ejecuta sobre `eclipse-temurin:21-jre-alpine`.
- Usuario sin privilegios (`spring`), `HEALTHCHECK` integrado y `.dockerignore`.

## 3. Pipeline (`.github/workflows/ci-cd.yml`)

```
push / PR → test → security (Snyk) → docker (build + push GHCR) → deploy (Compose + smoke test)
```

| Job | Qué hace | Indicador |
|---|---|---|
| `test` | `mvn verify`: pruebas JUnit/Mockito + reporte JaCoCo | IE2 |
| `security` | Snyk sobre dependencias y código; falla con severidad `high` o mayor | IE3 |
| `docker` | Construye y publica la imagen en GHCR etiquetada con el SHA del commit | IE1, IE4 |
| `deploy` | Levanta MySQL + microservicio con Compose en el runner y ejecuta prueba de humo | IE4, IE5 |

Los jobs `docker` y `deploy` solo corren en `main` y dependen de que `test` y `security` pasen.

## 4. Seguridad y bloqueos (IE3)

- **Dependabot** (`.github/dependabot.yml`): actualizaciones semanales de Maven, Docker y GitHub Actions.
- **Snyk**: si detecta vulnerabilidades altas el job falla y el pipeline se detiene antes de publicar o desplegar.
- **Protección de rama `main`**: exige que el pipeline pase antes de hacer merge.
- **Secretos** en GitHub Secrets: `SNYK_TOKEN`, `DB_PASSWORD`. Ninguna credencial va en el código.
- Contenedor sin root y límites de recursos en Compose.

## 5. Trazabilidad y calidad (IE4)

- Cada imagen se etiqueta con `ghcr.io/<repo>:<SHA del commit>`, y lleva labels OCI con el commit y el repositorio.
- El resumen de cada despliegue (commit, imagen, autor, ejecución) queda en el *Job Summary* de Actions.
- Calidad: pruebas automatizadas en cada push/PR, reporte de cobertura JaCoCo como artefacto, análisis de seguridad y smoke test post-despliegue.

## 6. Ejecución local

```bash
cp .env.example .env        # definir DB_PASSWORD
docker compose up -d --build
curl http://localhost:9004/
# Swagger: http://localhost:9004/swagger-ui.html
docker compose down -v
```

## 7. Uso de Inteligencia Artificial

_(Declarar aquí qué herramientas de IA se usaron y para qué, y citarlas según https://bibliotecas.duoc.cl/ia.
Ejemplo: Claude (Anthropic) se usó como apoyo para generar la estructura inicial de Dockerfile, docker-compose y workflow, que luego fueron revisados y validados por el equipo.)_

## 8. Conclusiones

_(Cada integrante debe escribir su reflexión personal **sin ayuda de IA**: qué aprendió y cuál fue su contribución.)_

### Integrante 1
### Integrante 2
