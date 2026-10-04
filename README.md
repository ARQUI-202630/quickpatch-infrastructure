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
