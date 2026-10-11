# Despliegue en la nube (pasos)

Este repositorio **no** incluye un despliegue ya realizado: la pauta pide mostrar el despliegue en la
nube, así que debes ejecutarlo tú y capturar evidencias. Opción simple, válida en cualquier proveedor
(AWS EC2, Azure VM, Google Compute Engine, Oracle Cloud, etc.):

1. Crea una máquina virtual Linux (Ubuntu) con al menos 4 GB de RAM (Kafka + 4 servicios Java + PostgreSQL).
2. Abre en el firewall/grupo de seguridad **solo el puerto 8080** (y 22 para SSH). No abras 5432 ni 9092.
3. Instala Docker y el plugin Compose (guía oficial de Docker para tu distribución).
4. Sube el proyecto (git clone de tu repositorio o `scp`).
5. `cp .env.example .env` y define claves propias y robustas.
6. `docker compose up --build -d` y verifica con `docker compose ps`.
7. Prueba desde tu PC: `curl http://IP_PUBLICA:8080/api/auth/login ...` (flujo del README).
8. Toma capturas: consola del proveedor con la VM, `docker compose ps`, y las pruebas desde tu PC.

Para producción real faltaría: HTTPS (proxy/Load Balancer con certificado), gestor de secretos,
respaldos de PostgreSQL y varias instancias de la VM o un orquestador (Kubernetes / ECS / AKS).
