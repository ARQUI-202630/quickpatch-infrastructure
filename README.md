# Infraestructura QUICKPATCH

Implementación de `docs/infrastructure/INFRASTRUCTURE.md`.

- VM1: Gateway / Nginx / runner de despliegue según diseño.
- VM2: Angular Admin Web servido como artefactos estáticos mediante Nginx en contenedor.
- VM3: k3s con 8 microservicios.
- VM4: PostgreSQL + PostGIS.
- VM5: Redis.
- VM6: Apache Kafka.
- VM7: MinIO + Prometheus + Loki + Grafana.
