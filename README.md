# ms-vehiculos – Pipeline CI/CD

**Evaluación Parcial N°2 – Ingeniería DevOps (DOY0101) – Duoc UC**

**Integrantes:** Kenyer Sandoval · Javier Valenzuela

Microservicio de Vehículos (CRUD con Swagger) desarrollado en Java 21 con Spring Boot y MySQL, al que se le automatizó todo el ciclo de vida con un pipeline CI/CD en GitHub Actions: pruebas, análisis de seguridad, contenedores, despliegue y orquestación.

## 1. Arquitectura

| Componente | Descripción |
|---|---|
| `ms-vehiculos` | API REST en el puerto 9004, empaquetada en una imagen Docker multi-etapa. |
| `mysql` | Base de datos MySQL 8.4 con volumen persistente. |
| Orquestación | Docker Compose: healthchecks, reinicio automático, límites de CPU y memoria, red propia. |
| Registro de imágenes | GitHub Container Registry (GHCR). |

## 2. Contenedores (IE1)

- `Dockerfile` multi-etapa: compila con `maven:3.9-eclipse-temurin-21` y ejecuta sobre `eclipse-temurin:21-jre-alpine`. La imagen final solo contiene el JRE y el `.jar`, por lo que es sumamente liviana.
- Se ejecuta con un usuario sin privilegios (`spring`) y define un `HEALTHCHECK` para validar la disponibilidad del servicio.
- `.dockerignore` configurado para evitar copiar archivos innecesarios al contexto de build.

## 3. Pipeline (`.github/workflows/ci-cd.yml`)

El flujo automatizado sigue la siguiente estructura:
`push / pull request → test → security (Snyk) → docker (build + push GHCR) → deploy (Compose + prueba de humo)`

| Job | Qué hace | Indicador |
|---|---|---|
| `test` | Ejecuta `mvn verify`: pruebas unitarias JUnit/Mockito, genera el reporte de cobertura JaCoCo (publicado como artefacto) y verifica el umbral mínimo de cobertura. | IE2 |
| `security` | Snyk analiza dependencias y código; falla y detiene el flujo si encuentra vulnerabilidades de severidad alta o crítica. | IE3 |
| `docker` | Construye la imagen y la publica en GHCR etiquetada con el SHA del commit. | IE1, IE4 |
| `deploy` | Levanta MySQL y el microservicio usando Docker Compose en el runner, para luego ejecutar una prueba de humo contra `GET /`. | IE4, IE5 |

Los jobs `docker` y `deploy` solo se ejecutan en la rama `main` y dependen estrictamente de que `test` y `security` finalicen con éxito. Las pruebas y el análisis de seguridad se ejecutan automáticamente en cada pull request.

## 4. Calidad, seguridad y bloqueos (IE2, IE3)

- **Pruebas y umbral de cobertura:** JaCoCo valida durante `mvn verify` que la cobertura de líneas de código sea de al menos 75%. Si es inferior, el build falla y el pipeline se detiene.
- **Dependabot** (`.github/dependabot.yml`): Revisa semanalmente Maven, Docker y GitHub Actions, abriendo pull requests con las actualizaciones sugeridas. Ignora saltos de versión mayor (ej. Java 21 a 25) para evitar rupturas de compatibilidad.
- **Snyk:** Comandos `snyk test` y `snyk code test` configurados con `--severity-threshold=high`. Previene la publicación de imágenes vulnerables.
- **Ruleset de la rama `main`** (`proteger-main`): Exige revisión mediante pull request y obliga a que los checks de `Build y pruebas unitarias` y `Análisis de seguridad (Snyk)` pasen antes de permitir la fusión. Bloquea el borrado de la rama y los force-push.
- **Secretos:** Gestionados en GitHub Secrets (`SNYK_TOKEN`, `DB_PASSWORD`). Las credenciales se inyectan en tiempo de ejecución; `application.yml` lee `DB_URL`, `DB_USERNAME` y `DB_PASSWORD` mediante variables de entorno.
- **Contenedor endurecido:** Ejecución sin usuario root, uso de imagen alpine y límites de recursos (CPU y memoria) definidos en el Compose.

## 5. Orquestación (IE5)

El archivo `docker-compose.yml` define dos servicios integrados:

- `mysql`: Cuenta con un volumen persistente `mysql_data` y un healthcheck basado en `mysqladmin ping`.
- `ms-vehiculos`: Configurado para arrancar únicamente cuando la base de datos esté lista (`depends_on` con `condition: service_healthy`). Incluye la directiva `restart: unless-stopped` y su propio healthcheck.

Ambos servicios están aislados en la red `vehiculos-net`. La imagen del microservicio se parametriza con la variable `IMAGE`, permitiendo que el pipeline inyecte la imagen de GHCR o que se construya localmente desde el `Dockerfile`.

## 6. Trazabilidad y calidad (IE4)

- **Trazabilidad:** Las imágenes se publican con el formato `ghcr.io/kensandoval-ing/evaluacion2-docker:<SHA del commit>` e incluyen labels OCI referenciando el commit y repositorio. El *Job Summary* de GitHub Actions documenta cada despliegue (commit, imagen, autor, ejecución).
- **Calidad Integral:** Automatización de pruebas en cada push/PR, umbrales de cobertura, análisis SAST/SCA con Snyk, políticas de bloqueo de merges y validación funcional post-despliegue (prueba de humo).
- **Auditoría de cambios:** Todo pase a `main` requiere un PR con checks aprobados, garantizando un registro histórico claro de quién implementó cada cambio y en qué momento.

## 7. Ejecución local

**Requisitos:** Tener instalado Docker Desktop.

```bash
# Clonar y configurar credenciales
cp .env.example .env          # Editar DB_PASSWORD en el archivo .env

# Levantar la infraestructura
docker compose up -d --build

# Prueba de humo manual
curl http://localhost:9004/

# Acceso a Swagger UI
# http://localhost:9004/swagger-ui.html

docker compose down -v
```

8. Uso de Inteligencia Artificial

   
Se utilizó Claude (Anthropic) como herramienta de asistencia técnica para:
Generar la estructura base del Dockerfile, docker-compose.yml, el flujo de GitHub Actions y la configuración de dependabot.yml.
Orientar las políticas de Snyk, gestión de secretos y protección de ramas.
Estructurar y redactar la documentación técnica de este README.
Todo el contenido generado fue iterado, probado localmente y ajustado por el equipo. Se corrigieron errores de sintaxis en el workflow observados en GitHub, se verificaron los despliegues locales y se extrajeron las credenciales hardcodeadas del código fuente. Las conclusiones presentadas a continuación fueron elaboradas exclusivamente por cada estudiante.

Cita: Anthropic. (2026). Claude Sonnet 5.5 [Modelo de lenguaje de gran tamaño]. https://claude.ai


9. Conclusiones


Kenyer Sandoval
La implementación de este pipeline CI/CD demostró la importancia crítica de adoptar una cultura DevSecOps. Automatizar las pruebas unitarias y el análisis de vulnerabilidades con Snyk desde las primeras etapas del desarrollo no solo evita que el código defectuoso llegue a producción, sino que también agiliza el trabajo del equipo. Hemos logrado un flujo continuo donde cada cambio se valida automáticamente, dándonos la confianza de que el microservicio mantendrá su estabilidad y seguridad de forma constante.

Javier Valenzuela
Diseñar esta arquitectura de integración y despliegue reafirmó el valor de aplicar metodologías de ingeniería estructuradas al ciclo de vida del software. Lograr que los contenedores, los escaneos de seguridad y la orquestación funcionen de forma autónoma y cohesionada en GitHub Actions elimina los cuellos de botella manuales. Además, asegurar la tolerancia a fallos mediante healthchecks y políticas estrictas de PR en la rama principal asegura que cada versión desplegada cumpla con estándares de calidad a nivel profesional.
