> **Repositorio archivado (SCRUM-338).** Su contenido se trasladó y ya no se mantiene aquí:
> - Ansible e inventario → [`quickpatch/infrastructure/`](https://github.com/ARQUI-202630/quickpatch/tree/develop/infrastructure)
> - Configuración de Nginx del gateway → [`quickpatch-api-gateway/nginx/`](https://github.com/ARQUI-202630/quickpatch-api-gateway/tree/develop/nginx)
> - Despliegue de Kafka y creación de topics → [`quickpatch-kafka/deploy/`](https://github.com/ARQUI-202630/quickpatch-kafka/tree/develop/deploy)
> - CI/CD → `.github/workflows/ci-cd.yml` de cada repositorio (los workflows reutilizables de este repo ya no se usan)

# Infraestructura QUICKPATCH

Implementación de `docs/infrastructure/INFRASTRUCTURE.md`.

- VM1: Gateway Nginx, panel Angular (archivos estáticos), runner de despliegue y k6.
- VM2: ambiente de QA (k3s, PostgreSQL, Redis, Kafka y Nginx propios).
- VM3: k3s con 8 microservicios.
- VM4: PostgreSQL + PostGIS.
- VM5: Redis.
- VM6: Apache Kafka.
- VM7: MinIO + Prometheus + Loki + Grafana.

## Contenido

- [`ansible/`](ansible/README.md): configuración de las VMs con Ansible.
- [`CI-CD.md`](CI-CD.md): pipelines de GitHub Actions; los workflows reutilizables están en `.github/workflows/` y las plantillas para cada repo en `plantillas/ci/`.
