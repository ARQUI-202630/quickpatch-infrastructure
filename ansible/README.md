# Ansible de QUICKPATCH

Aprovisiona las 7 VMs según el Documento de Infraestructura (secciones 5, 7, 8, 9 y 10).

Reparto de las 7 VMs (ADR-021): **1 de herramientas, 3 de producción y 3 de QA**, y producción y QA tienen la misma forma (entrada, servicios y datos). El panel Angular de producción vive en el gateway de VM1.

| Ambiente | Entrada | Servicios | Datos |
|---|---|---|---|
| Producción | VM1 (grupo `gateway`) | VM3 (`backend`) | VM4 (`database`, `cache` y `messaging`) |
| QA | VM2 (`qa_entrada`) | VM5 (`qa_servicios`) | VM6 (`qa_datos`) |
| Herramientas | VM7 (`storage_observability`) | | |

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
| `deploy-storage-observability.yml` | VM7 | Garage (S3), Prometheus, Loki y Grafana; buckets y llaves de Garage |
| `deploy-db.yml` | VM4 | PostgreSQL + PostGIS, una base por servicio y respaldo diario a Garage (VM7) |
| `deploy-cache.yml` | VM4 | Redis con contraseña (tope de memoria `redis_maxmemory`, porque comparte VM) |
| `deploy-kafka.yml` | VM4 | Kafka en modo KRaft y Kafka UI (sin puerto abierto: se entra por túnel SSH) |
| `deploy-k3s.yml` | VM3 | k3s de producción (la instalación compartida está en `tasks/k3s.yml`) |
| `deploy-qa.yml` | VM6, VM5 y VM2 | QA en tres partes y en este orden: datos (PostgreSQL, Redis, Kafka y Garage, VM6), servicios (k3s, VM5) y entrada (Nginx y panel, VM2), con secretos propios |
| `migrar-reparto-vms.yml` | VM5, VM6 y VM2 | **Una sola vez, destructivo:** limpia lo del reparto anterior (ver abajo). Exige `-e confirmar_borrado=true` |
| `deploy-gateway.yml` | VM1 | Nginx con HTTPS autofirmado: producción (`/api/` a VM3 y `/` al panel Angular), QA (VM2) y Grafana (VM7) |
| `deploy-runner.yml` | VM1 | Runner self-hosted de la organización, kubeconfig de QA y producción, y k6 |

## Entrar a producción, QA y Grafana

Desde la VPN, el firewall perimetral de la universidad solo deja pasar el 443 de VM1. Por eso VM1 reenvía cada nombre a su destino (ADR-015):

| Dirección | Destino | Quién entra |
|---|---|---|
| `https://quickpatch.internal` (o la IP de VM1) | Producción: panel y `/api/` | Cualquiera |
| `https://qa.quickpatch.internal` | QA (entrada en VM2) | Solo la red del equipo (`team_networks`) |
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
- **QA no puede llegar a producción:** PostgreSQL, Redis y Kafka de producción (VM4) solo aceptan conexiones desde VM3, y Garage (VM7), desde VM3 y VM4 para el respaldo. Los datos de QA (VM6) solo aceptan a VM5.
- **Garage con una llave por uso:** `servicios` solo accede a `evidencias` y `backups` solo a `backups-postgres`. Las llaves se definen en el vault y el playbook las importa (`tasks/garage.yml`).
- **En VM3 y VM5, el firewall permite el tráfico interno de k3s** (rangos de pods y servicios); sin eso los pods no se comunican.

## Cómo migrar al reparto del ADR-021

Se hace una sola vez, por fases y **en este orden**, porque VM5 y VM6 eran producción (Redis y Kafka) y pasan a ser QA. Todo estaba vacío: no hay datos que mover. Cada paso se prueba antes con `--check`.

1. `./ap playbooks/setup-base.yml --check --diff` y luego sin `--check`: firewall nuevo en las 7 VMs.
2. Producción: `deploy-db.yml`, `deploy-cache.yml` y `deploy-kafka.yml` (los tres ahora en VM4).
3. `./ap playbooks/migrar-reparto-vms.yml --tags vm5,vm6 -e confirmar_borrado=true`: quita el Redis de VM5 y el Kafka de VM6 (QA usa el mismo puerto 9092), y las reglas de firewall viejas (entre ellas la que dejaba a VM3 entrar al Kafka de VM6).
4. `./ap playbooks/deploy-qa.yml`: datos (VM6), k3s (VM5) y entrada (VM2). Después `deploy-runner.yml`, para que el runner tome el kubeconfig de QA nuevo (ahora apunta a VM5).
5. `./ap playbooks/migrar-reparto-vms.yml --tags vm2 -e confirmar_borrado=true`: quita el k3s y los volúmenes del QA anterior de VM2.

Kafka UI de producción no tiene puerto abierto (el perímetro de la universidad solo dejaba pasar el 8080 de VM6): se entra con `ssh -L 8080:10.43.98.209:8080 estudiante@10.43.98.209` y luego `http://localhost:8080`.
