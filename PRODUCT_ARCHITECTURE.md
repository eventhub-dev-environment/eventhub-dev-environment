
# Documento Maestro de Producto, Arquitectura y Operaciones — EventHub

**Versión:** 2.0 (Consolidado: Ecosistema Microservicios, Async Event-Driven, AWS S3 & Neon DB)

**Fecha de Última Actualización:** Octubre de 2026

**Estado:** Documento Vivo de Especificación de Arquitectura

---

## 1. Modelo de Negocio

**EventHub** es una plataforma **SaaS B2B Multi-tenant** de gestión de eventos, ticketing y analítica en tiempo real. Permite a empresas organizadoras de eventos operar bajo su propia marca (White-Label / Custom Branding) mediante un esquema de suscripción recurrente sumado a una comisión parametrizable por entrada vendida.

---

## 2. Flujo Vertical Crítico (End-to-End Walkthrough)

```text
[Comprador / Frontend] ──> (1. Reserva Entrada) ──> [API Gateway (8000)]
                                                         │
                                                         ▼
                                             [booking-service (8084)]
                                                         │
                                                         ├── (2. Lock Fila / DB Transaction en Neon Postgres)
                                                         ├── (3. Estado: AVAILABLE ──> HELD)
                                                         │
                                                         ▼
                                             [payment-service (8085)]
                                                         │
                                                         ├── (4. Procesamiento de Pago / Webhook)
                                                         │
                                                         ▼
                                             [booking-service (8084)]
                                                         │
                                                         ├── (5. Estado: HELD ──> SOLD)
                                                         ├── (6. Publica evento AMQP: 'booking.confirmed')
                                                         │
                                                         ▼
                                               [RabbitMQ Broker]
                                                         │
                                                         ▼
                                         [notification-service (8086)]
                                                         │
                                                         ├── (7. Obtiene Branding & Assets de AWS S3)
                                                         ├── (8. Compila plantilla HTML con Thymeleaf)
                                                         └── (9. Genera PDF/QR y envía correo vía SMTP)

```

---

## 3. Matriz de Microservicios y Componentes

| Repositorio / Microservicio | Tecnología / Stack | Responsabilidad Principal | Persistencia / Storage |
| --- | --- | --- | --- |
| `eventhub-api-gateway` | Java 21, Spring Boot 3.x / WebFlux | Enrutamiento dinámico, verificación de JWT, inyección de `X-Gateway-Secret` y propagación de `X-Tenant-Id`. | Stateless |
| `eventhub-config-service` | Java 21, Spring Boot 3.x | Parámetros de negocio por tenant, esquemas dinámicos de formularios de registro (`formSchema`). | MongoDB (`eventhub_config_db`) |
| `eventhub-auth-service` | Java 21, Spring Boot 3.x | Autenticación, registro de usuarios/organizadores, emisión de JWT con claims de tenant. | Neon PostgreSQL |
| `eventhub-event-service` | Java 21, Spring Boot 3.x | Gestión de eventos, localidades/tiers, precios y subida de imágenes de branding (logos, banners). | MongoDB + AWS S3 (`eventhub-branding-prod`) |
| `eventhub-booking-service` | Java 21, Spring Boot 3.x | Motor de reserva de boletas, máquinas de estado (`AVAILABLE` → `HELD` → `SOLD`), control de expiración. | Neon PostgreSQL + RabbitMQ |
| `eventhub-payment-service` | Java 21, Spring Boot 3.x | Integración y procesamiento de pagos / simulador de pasarela y webhooks de confirmación. | Neon PostgreSQL + RabbitMQ |
| `eventhub-notification-service` | Java 21, Spring Boot 3.x | Consumidor AMQP, generación de tickets en PDF con código QR y envío de correos con branding. | MongoDB + AWS S3 |

---

## 4. Estrategia Multirrepo y Visión CI/CD

El proyecto sigue un enfoque **Multirrepo**:

* Un repositorio independiente por cada microservicio (`eventhub-api-gateway`, `eventhub-auth-service`, `eventhub-event-service`, etc.).
* Repositorio de orquestación `eventhub-dev-environment` para desarrollo local y orquestación con Docker Compose.

### Pipeline CI/CD Objetivo (por microservicio)

1. **Trigger:** `push` / `pull_request` sobre la rama `main`.
2. **Build & Test:** Compilación e integración continua utilizando Java 21 y Gradle.
3. **Containerization:** Construcción de imagen OCI (Docker) optimizada.
4. **Publishing:** Publicación automática de la imagen etiquetada en Docker Hub / GitHub Container Registry (GHCR).