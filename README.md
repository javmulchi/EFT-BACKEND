# Banco XYZ – Migración a microservicios (Desarrollo Backend III, EFT)

Modernización del sistema legacy del Banco XYZ: procesos batch con Spring Batch,
división en microservicios, patrón BFF por canal, seguridad con tokens JWT validados
en cada servicio y mensajería asíncrona con Kafka. Todo se despliega con Docker.

## Servicios

| Servicio | Rol | Tecnologías |
|---|---|---|
| `gateway` (8080, único puerto público) | Punto de entrada y balanceo entre réplicas del BFF | Spring Cloud Gateway + LoadBalancer |
| `auth-service` | Valida usuarios de canal y emite el JWT | Spring Security, Nimbus JOSE |
| `batch-service` | 3 jobs batch, datos de cuentas, consume eventos de transacciones y fraude | Spring Batch, JPA, PostgreSQL, Kafka |
| `bff-service` (2 réplicas) | BFF para Web, Mobile y ATM | OAuth2 Resource Server, Resilience4j, Kafka, Caffeine |
| `kafka` / `postgres` | Mensajería / base de datos | Apache Kafka 3.8, PostgreSQL 16 |

Diagrama y justificaciones: ver `ARQUITECTURA.md`. Informe: `INFORME_TECNICO.md`.

## Cómo ejecutar (necesitas solo Docker)

```bash
cp .env.example .env        # en Windows: copy .env.example .env
# edita .env: pon claves propias (JWT_SECRET de 32+ caracteres)
docker compose up --build -d
docker compose ps           # espera a que todos estén "running"
docker compose logs -f batch-service
```

La primera vez demora varios minutos (descarga imágenes y dependencias Maven).

> Si cambias algo del esquema de base de datos y quieres partir de cero: `docker compose down -v`.

## Probar el flujo completo (curl)

```bash
# 1) Login del canal WEB
curl -s -X POST localhost:8080/api/auth/login -H "Content-Type: application/json" \
  -d '{"usuario":"web","password":"<AUTH_WEB_PASSWORD de tu .env>"}'
# copia el valor de "token" y guárdalo:  TOKEN=<token>

# 2) Ejecutar los 3 jobs batch (solo rol WEB)
curl -s -X POST localhost:8080/api/batch/intereses       -H "Authorization: Bearer $TOKEN"
curl -s -X POST localhost:8080/api/batch/transacciones   -H "Authorization: Bearer $TOKEN"
curl -s -X POST localhost:8080/api/batch/estados-cuenta  -H "Authorization: Bearer $TOKEN"
#   La respuesta indica leídos / escritos / saltados. Repetirlos NO duplica datos.

# 3) BFF Web: vista completa de cuentas
curl -s localhost:8080/api/web/cuentas -H "Authorization: Bearer $TOKEN"

# 4) Canal ATM: login con usuario "atm" y retiro
curl -s -X POST localhost:8080/api/auth/login -H "Content-Type: application/json" \
  -d '{"usuario":"atm","password":"<AUTH_ATM_PASSWORD>"}'            # TOKEN_ATM=<token>
curl -s -X POST localhost:8080/api/atm/retiro -H "Authorization: Bearer $TOKEN_ATM" \
  -H "Content-Type: application/json" -d '{"numeroCuenta":"001","monto":50000}'    # HTTP 202
curl -s localhost:8080/api/atm/saldo/001 -H "Authorization: Bearer $TOKEN_ATM"     # saldo ya descontado

# 5) Canal Mobile: login con "mobile"
curl -s localhost:8080/api/mobile/saldo/001 -H "Authorization: Bearer $TOKEN_MOBILE"
```

Pruebas de seguridad útiles para el video: sin token → 401; token `mobile` en `/api/atm/**` → 403;
retiro mayor al saldo → 422; monto negativo → 400; retiro ≥ 1.000.000 → alerta en
`docker compose logs batch-service`.

Pruebas de resiliencia: `docker compose stop batch-service` y consultar `/api/web/cuentas`
→ 503 controlado (circuit breaker); `docker compose start batch-service` y se recupera.
Para ver la escalabilidad, detén una réplica (`docker compose stop bff-service-1`) y el
gateway sigue respondiendo con la otra.

## Procesos batch

| Job | Archivo | Qué hace |
|---|---|---|
| `calculoInteresesMensuales` | `cuentas.csv` | Valida saldo e interés, aplica el interés y guarda la cuenta |
| `reporteTransaccionesDiarias` | `transacciones.csv` | Valida id, cuenta, tipo (DEPOSITO/RETIRO) y monto > 0 |
| `generacionEstadosCuentaAnuales` | `estado_cuenta.csv` | Valida año/montos, recalcula saldo final y publica un evento Kafka por estado |

- **Paralelismo:** 4 hilos, chunk de 10; la lectura se sincroniza (`SynchronizedItemStreamReader`).
- **Errores:** se reintentan solo errores transitorios de BD (3 veces); se saltan solo registros
  inválidos o líneas mal formadas (máx. 10). Cualquier otro error hace fallar el job.
- **Idempotencia:** los writers hacen `INSERT ... ON CONFLICT DO UPDATE` sobre la clave de negocio.
- Los CSV traen registros inválidos a propósito (saldo negativo, línea incompleta, monto no numérico,
  tipo inválido, año 0...). Se espera ver esos registros saltados en el log.

## Eventos Kafka

| Tópico | Publica | Consume | Efecto |
|---|---|---|---|
| `transaccion-creada` | bff-service (retiro ATM) | batch-service | Guarda la transacción y descuenta del saldo disponible |
| `alerta-fraude` | bff-service (retiro ≥ umbral) | batch-service | Registra la alerta (WARN) |
| `estado-cuenta-generado` | batch-service (fin del job) | bff-service (todas las réplicas) | Invalida la caché |

## Tests

Son tests unitarios (no requieren Docker). Con Maven y Java 17 instalados:

```bash
cd auth-service && mvn test
cd ../batch-service && mvn test
cd ../bff-service && mvn test
```

## Limitaciones conocidas (por honestidad académica)

- La autenticación es un servicio propio que emite JWT (HS256, clave compartida) y los demás
  servicios lo validan como *Resource Server*. **No es un proveedor OAuth2/OIDC completo**
  (Keycloak, etc.).
- Los usuarios son de **canal** (web/mobile/atm), no clientes finales; por eso no se valida que
  un cliente consulte solo sus cuentas.
- `batch-service` se ejecuta en una sola instancia (los jobs no coordinan entre réplicas).
- Los tests de integración (Kafka + Postgres con Testcontainers) no están implementados.
- No se ha ejecutado un despliegue en la nube desde este repositorio: ver `DESPLIEGUE_NUBE.md`.
