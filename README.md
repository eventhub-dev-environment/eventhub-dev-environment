# EventHub — Dev Environment

Entorno de desarrollo local para la plataforma EventHub (SaaS B2B multi-tenant de gestión de eventos). Este repositorio contiene el `docker-compose.yml` base y la configuración necesaria para levantar toda la infraestructura y los microservicios en tu máquina.

> Para la visión de producto, modelo de negocio y arquitectura completa del sistema, ver `PRODUCT_ARCHITECTURE.md`.

---

## Servicios incluidos

| Servicio | Descripción | Puerto |
|---|---|---|
| `api-gateway` | Punto de entrada único (Spring Cloud Gateway, WebFlux) | 8000 |
| `config-service` | Configuración por tenant, branding, `formSchema` (MongoDB) | 8081 |
| `auth-service` | Registro/login, emisión de JWT (MongoDB) | 8082 |
| `postgres` | Base de datos relacional (para futuros `booking-service`, `payment-service`) | 5432 |
| `mongodb` | Base de datos NoSQL (`config-service`, `auth-service`, futuros `catalog-service`, `analytics-service`) | 27017 |
| `kafka` | Broker de eventos (modo KRaft) | 9092 |
| `kafka-ui` | UI web para inspeccionar topics de Kafka | 8080 |
| `minio` | Almacenamiento de objetos compatible con S3 (branding, tickets) | 9000 / 9001 (consola) |
| `zipkin` | Backend de trazas distribuidas | 9411 |

---

## Setup

### 1. Requisitos previos

- Docker y Docker Compose instalados.
- Puertos `5432, 8000, 8080, 8081, 8082, 9000, 9001, 9092, 9411, 27017` libres en tu máquina.

### 2. Configura las variables de entorno

Copia el archivo de ejemplo y rellena los valores reales:

```bash
cp .env.example .env
```

Crea y Edita `.env` con tus propios valores.

| Variable | Descripción |
|---|---|
| `MONGO_ROOT_USER` / `MONGO_ROOT_PASS` | Credenciales root de MongoDB (también reusadas por MinIO) |
| `GATEWAY_INTERNAL_SECRET` | Secreto compartido entre el Gateway y los microservicios internos, usado para rechazar tráfico que no pase por el Gateway |
| `JWT_SECRET` | Clave HMAC para firmar/verificar JWTs. Genera una nueva con `openssl rand -hex 32` |
| `ZIPKIN_ENDPOINT` | Endpoint interno de Zipkin para exportar trazas |

### 3. Levanta el stack

```bash
docker compose up -d --build
```

Verifica que todos los contenedores estén saludables:

```bash
docker compose ps
```

### 4. Accesos rápidos

- API Gateway: `http://localhost:8000`
- Config Service (directo, solo debug): `http://localhost:8081`
- Auth Service (directo, solo debug): `http://localhost:8082`
- Kafka UI: `http://localhost:8080`
- MinIO Console: `http://localhost:9001`
- Zipkin: `http://localhost:9411`

---

## Operación diaria

### Reconstruir un solo servicio tras cambios de código

```bash
docker compose build --no-cache <nombre-servicio>
docker compose up -d --force-recreate <nombre-servicio>
```

### Ver logs en vivo de un servicio

```bash
docker logs <nombre-contenedor> -f
```

### Reiniciar todo desde cero (incluyendo datos)

```bash
docker compose down -v
docker compose up -d --build
```

`-v` elimina también los volúmenes (Postgres, Mongo, MinIO). Úsalo si necesitas que Mongo vuelva a leer `MONGO_INITDB_ROOT_USERNAME`/`PASSWORD` del `.env` — estas variables solo se aplican en la inicialización de un volumen vacío, no en volúmenes ya existentes.

---

## Arquitectura de seguridad interna (Gateway ↔ Microservicios)

Todos los microservicios internos (`config-service`, `auth-service`) exponen un `GatewayAccessFilter` que exige el header `X-Gateway-Secret` en cada request, con el valor de `GATEWAY_INTERNAL_SECRET`. Esto impide que alguien le pegue directo a un microservicio (ej. `localhost:8081`) sin pasar por el Gateway.

El `api-gateway` inyecta ese header automáticamente vía su `JwtAuthenticationFilter` (`GlobalFilter`), tanto en rutas públicas (login/register) como en rutas protegidas por JWT. Si agregas un nuevo microservicio con este mismo patrón de seguridad, asegúrate de:

1. Implementar el mismo `GatewayAccessFilter` (o extraerlo a una librería compartida).
2. Agregar su ruta en el `application.yml` del `api-gateway`.
3. Usar la misma variable `GATEWAY_INTERNAL_SECRET` — nunca un valor hardcodeado por servicio.

---

## Notas de versión

Los microservicios `config-service`, `auth-service` y `api-gateway` corren sobre **Spring Boot 4.1.1**. Si vas a agregar un nuevo microservicio, revisa la sección "Notas de migración Spring Boot 4.x" en `PRODUCT_ARCHITECTURE.md`