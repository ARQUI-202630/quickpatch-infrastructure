# CI/CD de QUICKPATCH

Los pipelines viven en este repo como **workflows reutilizables** (`.github/workflows/`). Cada repo de componente solo tiene un archivo corto que los llama (`plantillas/ci/`). Un cambio en el pipeline se hace aquí una vez y aplica a los 10 repos.

**Qué se prueba** y con qué herramientas sigue el Documento de Pruebas (`docs/testing/TD.md` del repo principal). **Dónde y cuándo se despliega** sigue el ADR-015: QA en VM2 con `release/*` y producción con `main`, siempre desde el runner de VM1.

## Compuertas del Documento de Pruebas

| Compuerta | Dónde está implementada |
|---|---|
| 0. Pre-push local | `plantillas/hooks/pre-push`, que cada persona instala en su clon |
| 1. CI del componente: lint, unitarias, Testcontainers, cobertura ≥ 80% | `ci-dotnet.yml`, `ci-java.yml`, `ci-angular.yml`, `ci-flutter.yml` y la acción `cobertura`; contratos en `ci-contratos.yml` |
| 2. Pruebas de sistema: E2E, OWASP ZAP, escáner PCI-DSS | `pruebas-sistema.yml` del repo principal, contra la entrada de QA (VM2) en `release/*` |
| 3. Carga con k6 | `pruebas-sistema.yml` del repo principal, contra QA, después de E2E y seguridad |

La diferencia con el texto actual del Documento de Pruebas: las compuertas 2 y 3 corren contra QA (entrada en VM2, servicios en VM5), no en un staging efímero ni en producción (ADR-015).

## Qué corre en cada rama

| Evento | Servicios (.NET y Java) | Panel web (Angular) | App móvil (Flutter) |
|---|---|---|---|
| PR a `develop` | Lint, pruebas unitarias y cobertura | Lint, pruebas, cobertura y build | Formato, análisis, pruebas y cobertura |
| Push a `develop` | Además, pruebas de integración (Testcontainers) | Lo mismo | Lo mismo |
| Push a `release/*` | Además, imagen en GHCR y **despliegue en QA** (k3s de VM5) | Además, **publica el panel en QA** (VM2) | Además, APK de release |
| Push a `main` | **Despliegue en producción** (k3s de VM3) con la misma imagen de QA | **Publica el panel en producción** (VM1) | APK de release |

QA solo cambia cuando se prepara una versión (`release/*`), no con cada push a `develop`.

### Producción usa la imagen probada en QA

`imagen.yml` etiqueta cada imagen con el hash del contenido del repo (`t-<árbol>`), no con el del commit. Cuando `release/x.y.z` se fusiona en `main` sin otros cambios, el contenido es idéntico, así que la imagen ya existe y se despliega sin construirla de nuevo. Si `main` trae algo que no pasó por `release/*` (por ejemplo un hotfix directo), el workflow construye una imagen nueva y deja un aviso en la ejecución.

## Dónde corre cada cosa

- **Runners de GitHub** (`ubuntu-latest`): lint, pruebas, build y publicación de imágenes.
- **Runner self-hosted de VM1** (etiquetas `self-hosted`, `vm1`, `red-laboratorio`): todo lo que tiene que llegar a la red del laboratorio, es decir los despliegues y las pruebas de sistema contra QA. Los runners de GitHub no alcanzan `10.43.x.x`.

El runner usa los kubeconfig de `~quickpatch/.kube/config-qa` y `config-produccion` que deja `deploy-runner.yml`, así que no hay credenciales de Kubernetes en GitHub. Para el panel de QA usa una llave SSH de VM1 a VM2 que también deja ese playbook.

## Workflows

| Archivo | Qué hace |
|---|---|
| `ci-dotnet.yml` | `dotnet format`, build, pruebas de `tests/unit` y `tests/integration`, cobertura (coverlet) |
| `ci-java.yml` | Gradle (`test`, `integrationTest`, `jacocoTestReport`) o Maven (`test`, `verify`); cobertura con JaCoCo |
| `ci-angular.yml` | `npm run lint`, `npm run test:ci -- --code-coverage`, `npm run build`; guarda el build como artefacto `panel` |
| `ci-flutter.yml` | `dart format`, `flutter analyze`, `flutter test --coverage`; opcionalmente el APK |
| `ci-contratos.yml` | Spectral sobre `openapi/`, AJV sobre `events/` y forma de los tags (`vMAJOR.MINOR.PATCH`) |
| `actions/cobertura` | Acción compartida: falla si la cobertura de líneas está por debajo del mínimo (80% por defecto) |
| `imagen.yml` | Construye y publica en `ghcr.io/arqui-202630/<repo>`, o reutiliza la imagen si ya existe |
| `deploy-k3s.yml` | `kubectl set image` y espera el rolling update; si falla, `kubectl rollout undo` |
| `deploy-panel.yml` | Reemplaza los archivos del panel en VM1 (producción) o VM2 (QA) |
| `infra.yml` | CI de este repo: actionlint, sintaxis de los playbooks, ansible-lint (perfil `production`) y gitleaks |

Si un repo todavía no tiene código, cada workflow lo detecta, deja un aviso y termina en verde. Así las plantillas se pueden instalar desde ya.

## Convenciones para los equipos

Salen de la sección 2 del Documento de Pruebas; el pipeline depende de ellas.

- **Versiones por defecto:** .NET 8, Java 17, Node 20 (Angular 17) y Flutter estable. Cada plantilla puede cambiarlas con `with:`.
- **Cobertura:** mínimo 80% de líneas (compuerta 1). Sin reporte de cobertura, el pipeline falla.
- **.NET:** una solución (`.sln` o `.slnx`) en la raíz del repo; pruebas unitarias en `tests/unit/` y de integración en `tests/integration/`; los proyectos de pruebas referencian `coverlet.collector`.
- **Java (Matching):** `build.gradle` (o `pom.xml`) en la raíz con su wrapper. En Gradle, una tarea `integrationTest` para las pruebas de integración; en Maven, Failsafe con `*IT.java`. JaCoCo configurado con reporte XML.
- **Angular:** los scripts `lint`, `test:ci` (sin modo watch, navegador headless) y `build` en `package.json`.
- **Contratos:** OpenAPI en `openapi/` y esquemas JSON en `events/`. Las reglas propias de Spectral (CTR-002 y CTR-003) van en `.spectral.yaml` del repo de contratos.
- **Pruebas de sistema (repo principal):** Playwright en `tests/e2e/` (con `package.json` y `BASE_URL` como dirección de QA), colecciones de Postman en `tests/e2e/*.postman_collection.json`, scripts de k6 en `tests/performance/*.js` (con `BASE_URL` y umbrales propios) y reglas opcionales de ZAP en `tests/security/zap-baseline.conf`.
- **Imagen:** un `Dockerfile` en la raíz del repo del servicio.
- **Deployment:** el nombre del Deployment en k3s es el nombre del repo sin `quickpatch-` (por ejemplo `identity`), en el namespace `quickpatch`.

## Instalar en un repo

1. Copiar la plantilla que corresponda a `.github/workflows/ci-cd.yml` del repo:
   - `servicio-dotnet.yml`: identity, actors, catalog, service-request, ranking, payments, communication
   - `servicio-java.yml`: matching
   - `web.yml`: quickpatch-web
   - `mobile.yml`: quickpatch-mobile
   - `contracts.yml`: quickpatch-contracts (como `.github/workflows/ci.yml`)
2. Subirla por PR a `develop`, como cualquier cambio.
3. Hook pre-push (compuerta 0), una vez por clon: copiar `plantillas/hooks/pre-push` como `.githooks/pre-push` del repo y correr `git config core.hooksPath .githooks`. Es una ayuda local: `git push --no-verify` lo salta, y el CI corre todo igual.

Las plantillas llaman los workflows en `@main` de este repo: este repo tiene que estar fusionado en `main` antes de instalarlas.

## Antes del primer despliegue

1. **Registrar el runner de VM1.** En GitHub: organización → Settings → Actions → Runners → New runner, copiar el token, guardarlo como `vault_github_runner_token` en el vault y correr `deploy-runner.yml`. El token dura una hora.
2. **Restringir el runner.** Los repos son públicos, así que hay que evitar que un PR desde un fork ejecute código en VM1:
   - Los workflows de despliegue solo se disparan con `push`, nunca con `pull_request` (ya está así en las plantillas).
   - Organización → Settings → Actions → General → "Fork pull request workflows": exigir aprobación para todos los colaboradores externos.
   - En Settings → Actions → Runner groups, el grupo del runner debe tener activado "Allow public repositories" para que los repos públicos lo usen. Si el plan de la organización lo permite, limitar además "Workflow access" a `deploy-k3s.yml` y `deploy-panel.yml` de este repo; hay que confirmarlo en la configuración, porque algunas de estas opciones dependen del plan de GitHub.
3. **Aprobación para producción:** en cada repo, Settings → Environments → `produccion` → Required reviewers. El ambiente se crea solo la primera vez que corre un despliegue.
4. **Manifiestos de k3s:** el workflow `deploy-k3s.yml` (distinto del playbook del mismo nombre) actualiza un Deployment que ya existe; no lo crea. Los manifiestos irán en `k8s/` de este repo, porque se aplican en los dos k3s (QA en VM2 y producción en VM3); las carpetas `vm1-gateway/` a `vm7-storage-observability/` son del andamiaje inicial. Los manifiestos de cada servicio todavía no están escritos (`k8s/`).
5. **Visibilidad de las imágenes:** las imágenes quedan enlazadas a su repo. Si GHCR las deja privadas, k3s no las puede descargar: hay que hacerlas públicas (página del paquete → Package settings) o agregar un `imagePullSecret`.

## Pendiente

- Manifiestos de k3s por servicio (Deployment, Service, recursos de la sección 5.6 del Documento de Infraestructura).
- Prueba de humo después de desplegar en producción, cuando los servicios tengan un endpoint de salud.
- Pruebas de contrato con Pact entre servicios (el Documento de Pruebas las menciona en CAT-011 y ACT-013).
- Limpieza de los datos del tenant de prueba después de k6 (PRF-006): con una base por servicio, cada servicio limpia sus propias tablas.
- Validación de manifiestos de k3s con kubeconform (reemplazo de Kubeval, que ya no se mantiene) cuando existan los manifiestos.
