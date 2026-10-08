# Ansible de QUICKPATCH

Aprovisiona las 7 VMs según el Documento de Infraestructura (secciones 5, 7, 8, 9 y 10).

Producción usa VM1 y VM3 a VM7. **VM2 es el ambiente de QA** (ADR-015), y el panel Angular vive en el gateway de VM1.

## Ejecutar sin instalar nada

`./ap` corre `ansible-playbook` dentro de un contenedor (`Dockerfile`), con la misma versión para todo el equipo:

```bash
docker build -t quickpatch-ansible .
./ap playbooks/diagnostico.yml -k -K
```

## Requisitos (si se instala Ansible localmente)

- Estar conectado a la VPN de la Javeriana (las VMs están en `10.43.x.x`).
- Ansible y las colecciones:

```bash
python3 -m venv ~/.venvs/ansible
~/.venvs/ansible/bin/pip install "ansible-core==2.18.*"
~/.venvs/ansible/bin/ansible-galaxy collection install -r requirements.yml
```

## Antes del primer despliegue

1. Poner los secretos. `inventory/group_vars/all/vault.yml` está cifrado y Git lo ignora: se copia desde un lugar seguro, o se crea desde la plantilla:

```bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
# llenar los valores
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

2. Guardar la contraseña del vault en `~/.config/quickpatch/vault-pass`, fuera del repositorio. `./ap` la usa sola si existe; si no, hay que agregar `--ask-vault-pass` a cada comando.

El usuario SSH (`vm_admin_user`) y la red del equipo (`team_networks`) ya están configurados en `inventory/group_vars/all/main.yml`. El paso a paso completo, desde un computador nuevo, está en el Anexo A del Documento de Infraestructura.

## Orden recomendado

Primero solo lectura, después en modo simulación y al final de verdad:

```bash
# 1. Conexión y estado de las VMs (no cambia nada)
./ap playbooks/diagnostico.yml -k -K

# 2. Simular la base en una sola VM y revisar qué cambiaría
./ap playbooks/setup-base.yml --limit vm5 --check --diff -k -K

# 3. Aplicar en una VM, comprobar que SSH sigue funcionando, y luego en el resto
./ap playbooks/setup-base.yml --limit vm5 -k -K
./ap playbooks/setup-base.yml -k -K

# 4. Todo lo demás, en el orden de site.yml
./ap playbooks/site.yml -k -K
```

`-k` pide la contraseña SSH y `-K` la de sudo. El acceso a las VMs es con contraseña: el equipo decidió no exigir llaves públicas (Documento de Infraestructura, sección 10.4).

## Playbooks

| Playbook | VM | Qué hace |
|---|---|---|
| `diagnostico.yml` | Todas | Solo lectura: sistema, recursos, Docker y ufw |
| `setup-base.yml` | Todas | Paquetes, usuario de despliegue, SSH, Docker, firewall, node_exporter y Promtail |
| `deploy-observabilidad.yml` | VM1 | Prometheus, Loki y Grafana de los dos ambientes (ADR-022); dashboards y alertas |
| `deploy-garage.yml` | VM6 | Garage de producción (S3), con una llave por servicio; buckets y llaves de Garage |
| `deploy-db.yml` | VM4 y VM5 (QA) | PostgreSQL + PostGIS, una base por servicio con roles `_migrator` y `_app`, y respaldo diario a Garage (VM6) |
| `aplicar-roles-db.yml` | VM4 y VM5 (QA) | Aplica el `db/roles.sql` de cada servicio; va después de sus migraciones |
| `primer-despliegue-servicios.yml` | VM1 (hacia el k3s de un ambiente) | Secret de cada servicio desde el vault y sus manifiestos, con la IP de Kafka del ambiente (`-e ambiente=qa` o `produccion`) |
| `deploy-cache.yml` | VM4 | Redis con un usuario ACL por servicio, restringido a sus claves (tope de memoria `redis_maxmemory`) |
| `deploy-kafka.yml` | VM6 | Kafka en modo KRaft y Kafka UI |
| `deploy-k3s.yml` | VM3 | k3s de producción (la instalación compartida está en `tasks/k3s.yml`) |
| `deploy-gateway.yml` | VM1 | Nginx con HTTPS autofirmado: producción (`/api/` a VM3 y `/` al panel Angular), QA (VM2) y Grafana (VM7) |
| `deploy-runner.yml` | VM1 | Runner self-hosted de la organización, kubeconfig de QA y producción, y k6 |

## Entrar a producción, QA y Grafana

Desde la VPN, el firewall perimetral de la universidad solo deja pasar el 443 de VM1. Por eso VM1 reenvía cada nombre a su destino (ADR-015):

| Dirección | Destino | Quién entra |
|---|---|---|
| `https://quickpatch.internal` (o la IP de VM1) | Producción: panel y `/api/` | Cualquiera |
| `https://qa.quickpatch.internal` | QA en VM2 | Solo la red del equipo (`team_networks`) |
| `https://grafana.quickpatch.internal` | Grafana en VM7 | Solo la red del equipo, con login |

No hay DNS, así que cada persona agrega esta línea una vez a su archivo hosts (`C:\Windows\System32\drivers\etc\hosts` en Windows, abierto como administrador; `/etc/hosts` en Linux y macOS):

```
10.43.100.168  quickpatch.internal  qa.quickpatch.internal  grafana.quickpatch.internal
```

El certificado es autofirmado: el navegador muestra una advertencia la primera vez. Prometheus no se publica; sus datos se ven desde Grafana. Si VM1 se cae, el acceso de respaldo es un túnel SSH a otra VM, por ejemplo `ssh -L 3000:10.43.99.8:3000 estudiante@10.43.99.8`.

## Decisiones de implementación

- **Contenedores con red de host.** Con puertos publicados, Docker se salta las reglas de `ufw`; con red de host, el firewall de la sección 10.2 sí aplica.
- **El firewall abre el 22 antes de activarse**, para no perder la conexión.
- **SSH con contraseña, sin login de `root`.** Se decidió no exigir llaves (Documento de Infraestructura, sección 10.4). Si algún día se decide, `deploy_user_pubkeys` y `ssh_disable_password_auth` ya lo permiten.
- **El escritorio remoto (3389) queda abierto para la VPN del equipo**, porque el laboratorio lo usa y activar `ufw` lo bloquearía.
- **QA no puede llegar a producción:** PostgreSQL, Redis, Kafka y Garage de producción solo aceptan conexiones desde VM3 (y Garage, también desde VM4 para el respaldo).
- **Garage con una llave por uso:** `servicios` solo accede a `evidencias` y `backups` solo a `backups-postgres`. Las llaves se definen en el vault y el playbook las importa (`tasks/garage.yml`).
- **En VM2 y VM3, `ufw` permite el tráfico interno de k3s** (rangos de pods y servicios); sin eso los pods no se comunican.

## Migración al reparto del ADR-022

Rige el reparto del ADR-022 (revisión del profesor): VM1 herramientas, producción en VM3, VM4 y VM6, y QA en VM2, VM5 y VM7. Se migra por pasos y se comprueba cada uno antes del siguiente.

**Paso 1 — Observabilidad a VM1** (este orden evita dejar producción sin monitoreo ni respaldo):

1. `./ap playbooks/setup-base.yml --limit vm1`: abre el 3100 de VM1 a las 7 VMs.
2. `./ap playbooks/deploy-observabilidad.yml`: Prometheus, Loki y Grafana en VM1, con los dashboards y las alertas.
3. `./ap playbooks/setup-base.yml`: las 7 VMs mandan logs a VM1 y dejan que VM1 lea `node_exporter`.
4. `./ap playbooks/deploy-gateway.yml`: el Nginx de VM1 apunta a su propio Grafana.
5. Comprobar en Grafana que llegan métricas y logs de las 7 VMs.
6. `./ap playbooks/deploy-garage.yml`: VM7 queda solo con Garage (quita los contenedores de observabilidad).
7. `./ap playbooks/migrar-adr-022.yml --tags obs-vm7,fw-9100 -e confirmar_borrado=true`: quita lo viejo (volúmenes, configuración y reglas de firewall).

Prometheus (9090) y Grafana (3000) no tienen puerto abierto en el firewall: se llega a Grafana por el Nginx de VM1 (`grafana.quickpatch.internal`), o con un túnel SSH a VM1 si el proxy falla.

**Paso 2 — Garage y Kafka de producción en VM6:**

1. `./ap playbooks/migrar-adr-022.yml --tags qa-vm6 -e confirmar_borrado=true`: vacía el QA que estaba en VM6 (usa los mismos puertos 9092 y 9000).
2. `./ap playbooks/setup-base.yml`: firewall de VM6 (9092 y 9000 desde VM3; 9000 también desde VM4).
3. `./ap playbooks/deploy-kafka.yml` y `./ap playbooks/deploy-garage.yml`: Kafka con su UI y Garage en VM6, con las llaves `service-request` y `backups` y los buckets `service-request-evidencias` y `backups-postgres`.
4. `./ap playbooks/deploy-db.yml`: el respaldo diario de VM4 apunta a VM6. Se prueba corriéndolo una vez a mano.
5. `./ap playbooks/migrar-adr-022.yml --tags kafka-vm4,garage-vm7,kafka-ui -e confirmar_borrado=true`: quita Kafka de VM4, Garage de VM7 y cierra el 8080 de Kafka UI en VM6.

Kafka UI no tiene puerto abierto: `ssh -L 8080:10.43.99.12:8080 estudiante@10.43.99.12` y luego `http://localhost:8080`.

**Paso 3 — Redis con usuarios por servicio y PostGIS en VM4:**

1. `./ap playbooks/setup-base.yml`: firewall (VM4 acepta el 6379 desde VM3).
2. `./ap playbooks/deploy-cache.yml`: Redis con el archivo de usuarios (`default` apagado, `admin` para operación y un usuario por servicio restringido a `<servicio>:*`).
3. `./ap playbooks/deploy-db.yml`: habilita PostGIS también en `db_service_request`.
4. `./ap playbooks/migrar-adr-022.yml --tags redis-vm5 -e confirmar_borrado=true`: quita en VM5 el acceso de VM3 al 6379.

Los servicios reciben su usuario y su contraseña en su propio `Secret` de Kubernetes (Documento de Infraestructura, 3.4).

**Paso 5 — QA en VM2, VM5 y VM7:**

Los playbooks de producción sirven también a QA: `deploy-db`, `deploy-cache`, `deploy-kafka`, `deploy-garage` y `deploy-k3s` eligen su grupo de producción y su grupo de QA, y lo que cambia (secretos, memoria, retención, capacidad) está en `inventory/group_vars/produccion.yml` y `qa.yml`. `deploy-qa.yml` ya no existe. Para aplicarlo solo a QA se usa `--limit qa`.

1. `./ap playbooks/migrar-adr-022.yml --tags qa-vm2 -e confirmar_borrado=true`: quita el Nginx del QA anterior de VM2.
2. `./ap playbooks/setup-base.yml`: firewall (VM2: 30080 y 6443 desde VM1; VM5: 5432 y 6379 desde VM2; VM7: 9092 y 9000 desde VM2).
3. `./ap playbooks/deploy-garage.yml --limit qa`, `deploy-kafka.yml --limit qa`, `deploy-db.yml --limit qa` y `deploy-cache.yml --limit qa`: Garage y Kafka en VM7, PostgreSQL y Redis en VM5.
4. `./ap playbooks/deploy-k3s.yml --limit qa`: k3s de QA en VM2.
5. `./ap playbooks/deploy-gateway.yml` y `deploy-runner.yml`: el proxy de VM1 reenvía QA al Traefik de VM2 y el runner toma el kubeconfig nuevo.
6. `./ap playbooks/migrar-adr-022.yml --tags qa-vm5 -e confirmar_borrado=true`: quita el k3s de QA de VM5.

`qa.quickpatch.internal` entra por el proxy de VM1 al Traefik de VM2. Mientras no haya imágenes del gateway y del panel, `/` y `/api/` de QA responden 404.
