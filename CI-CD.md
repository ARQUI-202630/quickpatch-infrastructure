# CI/CD de QUICKPATCH

Los pipelines viven en este repo como **workflows reutilizables** (`.github/workflows/`). Cada repo de componente solo tiene un archivo corto que los llama (`plantillas/ci/`). Un cambio en el pipeline se hace aquí una vez y aplica a los 10 repos.

## Qué corre en cada rama

| Evento | Servicios (.NET y Java) | Panel web (Angular) | App móvil (Flutter) |
|---|---|---|---|
| PR a `develop` | Lint y pruebas unitarias | Lint, pruebas y build | Formato, análisis y pruebas |
| Push a `develop` | Además, pruebas de integración (Testcontainers) | Lo mismo | Lo mismo |
| Push a `release/*` | Además, imagen en GHCR y **despliegue en QA** (k3s de VM2) | Además, **publica el panel en QA** (VM2) | Además, APK de release |
| Push a `main` | **Despliegue en producción** (k3s de VM3) con la misma imagen de QA | **Publica el panel en producción** (VM1) | APK de release |

QA solo cambia cuando se prepara una versión (`release/*`), no con cada push a `develop`.

### Producción usa la imagen probada en QA

`imagen.yml` etiqueta cada imagen con el hash del contenido del repo (`t-<árbol>`), no con el del commit. Cuando `release/x.y.z` se fusiona en `main` sin otros cambios, el contenido es idéntico, así que la imagen ya existe y se despliega sin construirla de nuevo. Si `main` trae algo que no pasó por `release/*` (por ejemplo un hotfix directo), el workflow construye una imagen nueva y deja un aviso en la ejecución.

## Dónde corre cada cosa

- **Runners de GitHub** (`ubuntu-latest`): lint, pruebas, build y publicación de imágenes.
- **Runner self-hosted de VM1** (etiquetas `self-hosted`, `vm1`, `red-laboratorio`): todo lo que tiene que llegar a la red del laboratorio, es decir los despliegues. Los runners de GitHub no alcanzan `10.43.x.x`.

El runner usa los kubeconfig de `~quickpatch/.kube/config-qa` y `config-produccion` que deja `deploy-runner.yml`, así que no hay credenciales de Kubernetes en GitHub. Para el panel de QA usa una llave SSH de VM1 a VM2 que también deja ese playbook.

## Workflows

| Archivo | Qué hace |
|---|---|
| `ci-dotnet.yml` | Formato (`dotnet format`), build y pruebas. Integración con `Category=Integration` |
| `ci-java.yml` | Maven o Gradle: `test`, o `verify`/`check` con integración |
| `ci-angular.yml` | `npm run lint`, `npm run test:ci`, `npm run build`; guarda el build como artefacto `panel` |
| `ci-flutter.yml` | `dart format`, `flutter analyze`, `flutter test`; opcionalmente el APK |
| `imagen.yml` | Construye y publica en `ghcr.io/arqui-202630/<repo>`, o reutiliza la imagen si ya existe |
| `deploy-k3s.yml` | `kubectl set image` y espera el rolling update; si falla, `kubectl rollout undo` |
| `deploy-panel.yml` | Reemplaza los archivos del panel en VM1 (producción) o VM2 (QA) |
| `infra.yml` | CI de este repo: actionlint y sintaxis de los playbooks |

Si un repo todavía no tiene código, cada workflow lo detecta, deja un aviso y termina en verde. Así las plantillas se pueden instalar desde ya.

## Convenciones para los equipos

- **.NET:** una solución (`.sln` o `.slnx`) en la raíz del repo. Las pruebas de integración llevan `[Trait("Category", "Integration")]`.
- **Java (Matching):** `pom.xml` o `build.gradle` en la raíz, preferiblemente con su wrapper (`mvnw`, `gradlew`). En Maven, las pruebas de integración se llaman `*IT.java` (Failsafe).
- **Angular:** los scripts `lint`, `test:ci` (sin modo watch, navegador headless) y `build` en `package.json`.
- **Imagen:** un `Dockerfile` en la raíz del repo del servicio.
- **Deployment:** el nombre del Deployment en k3s es el nombre del repo sin `quickpatch-` (por ejemplo `identity`), en el namespace `quickpatch`.

## Instalar en un repo

1. Copiar la plantilla que corresponda a `.github/workflows/ci-cd.yml` del repo:
   - `servicio-dotnet.yml`: identity, actors, catalog, service-request, ranking, payments, communication
   - `servicio-java.yml`: matching
   - `web.yml`: quickpatch-web
   - `mobile.yml`: quickpatch-mobile
2. Subirla por PR a `develop`, como cualquier cambio.

Las plantillas llaman los workflows en `@main` de este repo: este repo tiene que estar fusionado en `main` antes de instalarlas.

## Antes del primer despliegue

1. **Registrar el runner de VM1.** En GitHub: organización → Settings → Actions → Runners → New runner, copiar el token, guardarlo como `vault_github_runner_token` en el vault y correr `deploy-runner.yml`. El token dura una hora.
2. **Restringir el runner.** Los repos son públicos, así que hay que evitar que un PR desde un fork ejecute código en VM1:
   - Los workflows de despliegue solo se disparan con `push`, nunca con `pull_request` (ya está así en las plantillas).
   - Organización → Settings → Actions → General → "Fork pull request workflows": exigir aprobación para todos los colaboradores externos.
   - En Settings → Actions → Runner groups, el grupo del runner debe tener activado "Allow public repositories" para que los repos públicos lo usen. Si el plan de la organización lo permite, limitar además "Workflow access" a `deploy-k3s.yml` y `deploy-panel.yml` de este repo; hay que confirmarlo en la configuración, porque algunas de estas opciones dependen del plan de GitHub.
3. **Aprobación para producción:** en cada repo, Settings → Environments → `produccion` → Required reviewers. El ambiente se crea solo la primera vez que corre un despliegue.
4. **Manifiestos de k3s:** `deploy-k3s.yml` actualiza un Deployment que ya existe; no lo crea. Los manifiestos de cada servicio todavía no están escritos (`k8s/`).
5. **Visibilidad de las imágenes:** las imágenes quedan enlazadas a su repo. Si GHCR las deja privadas, k3s no las puede descargar: hay que hacerlas públicas (página del paquete → Package settings) o agregar un `imagePullSecret`.

## Pendiente

- Manifiestos de k3s por servicio (Deployment, Service, recursos de la sección 5.6 del Documento de Infraestructura).
- Pruebas de sistema del repo principal en `release/*`: E2E, OWASP ZAP y k6 contra QA, en el runner de VM1.
- Prueba de humo después de desplegar en producción, cuando los servicios tengan un endpoint de salud.
- Pruebas de contrato (Pact) y validación de `quickpatch-contracts`.
