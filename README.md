# Documento Maestro de Producto, Arquitectura y Operaciones — EventHub

**Versión:** 1.0 (Consolidado: Negocio, Arquitectura, Repositorios & Hoja de Ruta)

**Fecha de Última Actualización:** 8 de septiembre de 2026

**Estado:** Documento Vivo de Especificación Final

---

## 1. Modelo de Negocio

**EventHub** es una plataforma **SaaS B2B Multi-tenant** de gestión de eventos, ticketing y analítica en tiempo real. Permite a empresas organizadoras de eventos operar bajo su propia marca (White-Label / Custom Branding) mediante un esquema de suscripción recurrente (Tiered Pricing) sumado a una comisión parametrizable por cada entrada vendida.

### Flujos de Ingreso:

* **Suscripción Base (SaaS Tier):** Cobro mensual o anual por el uso de la infraestructura multi-tenant, según el volumen de eventos y límites de concurrencia.


* **Comisión por Transacción:** Porcentaje parametrizable por tenant (`config-service`) aplicado sobre el valor de cada ticket vendido.


* **Servicios Add-on:** Módulos avanzados como dominios personalizados, integraciones premium y soporte de transmisión de alto tráfico.



---

## 2. Clientes Objetivos y Usuarios Persona

### Clientes Objetivos (Tenants B2B):

* **Organizadores de Eventos Deportivos:** Maratones, carreras nocturnas, torneos de ciclismo (ej. carreras 5K/10K/Media Maratón). Requieren formularios adaptativos con tallas de camisetas, datos médicos y contactos de emergencia.


* **Productoras de Conciertos y Teatro:** Promotoras que gestionan eventos masivos con silletería numerada, mapas de asientos y control estricto de accesos.


* **Empresas de Conferencias y Talleres:** Organizadores B2B de charlas técnicas, meetups y seminarios con esquemas de admisión general y certificados de asistencia.



### Usuarios Persona:

* **Tenant Admin (Organizador):** Gerente o coordinador de eventos de la empresa cliente. Configura el branding, crea eventos, selecciona/personaliza plantillas de registro, define precios y analiza métricas de venta en tiempo real.


* **Comprador Final (Cliente B2C):** Asistente al evento que navega la oferta y adquiere entradas. Selecciona la cantidad de entradas o asientos, completa los atributos dinámicos (talla, club, datos de pago), paga de forma segura y descarga sus tickets.


* **Super Admin (EventHub):** Equipo interno de soporte y operaciones de EventHub. Gestiona la incorporación de nuevos tenants, monitorea la facturación global, supervisa límites de plataforma y controla la salud de los microservicios.



---

## 3. Problema y Oportunidad de Mercado

Las soluciones de boletaje tradicionales presentan graves falencias para las empresas medianas y grandes organizadoras de eventos:

* **Falta de Personalización y Branding:** Las plataformas existentes fuerzan al comprador a salir del ecosistema de la marca organizadora, debilitando la identidad del cliente B2B.


* **Incapacidad de Adaptar el Esquema de Entrada:** La mayoría de las boleterías están optimizadas únicamente para conciertos (asientos numerados) o entradas generales simples, fallando al capturar información específica requerida por eventos deportivos o corporativos (tallas, clubes, contactos de emergencia, requerimientos dietéticos).


* **Sobrevendido (Overbooking) por Concurrencia Masiva:** Durante ventas relámpago o lanzamientos de alta demanda, los sistemas tradicionales sufren condiciones de carrera y terminan vendiendo el mismo cupo o asiento a múltiples usuarios.


* **Acoplamiento Síncrono y Latencia Elevada:** Procesos pesados como la generación de PDFs o el envío de correos se ejecutan en la misma transacción HTTP de compra, provocando caídas del sistema y timeouts en la pasarela de pago.



---

## 4. Hipótesis de Valor

> *"Creemos que al ofrecer a las empresas organizadoras de eventos un SaaS multi-tenant verdaderamente configurable —que combine esquemas de registro dinámicos (plantillas e información personalizada) con una arquitectura distribuida que garantice cero overbooking mediante consistencia fuerte en base de datos y procesamiento asíncrono de tickets— lograremos reducir a cero los errores por venta duplicada, aumentar la conversión de compra en un 25% al acortar el tiempo de respuesta síncrono del checkout a menos de 500 ms, y ofrecer una plataforma 100% personalizable por tenant sin modificar una sola línea de código."*
> 

---

## 5. Primer Flujo Vertical Crítico (End-to-End Walkthrough)

```text
[Comprador / React Frontend] ──> (1. Solicita Reserva) ──> [API Gateway] ──> [booking-service (Spring Boot)]
                                                                                │
                                                                                ├── (2. Lock Fila / Transacción ACID en Postgres)
                                                                                ├── (3. Cambia Estado: AVAILABLE ──> HELD)
                                                                                │
                                                                                ▼
                                                                      [payment-service] ──> [Pasarela / Stripe Simulator]
                                                                                │
                                                                                ├── (4. Pago Exitoso)
                                                                                ▼
                                                                      [booking-service]
                                                                                │
                                                                                ├── (5. UPDATE: HELD ──> SOLD)
                                                                                ├── (6. COMMIT Transacción)
                                                                                │
                                                                                └──> Publica Evento: BookingConfirmed ──> [Kafka Topic]
                                                                                                                               │
                                                                                                                               ▼
                                                                                                                  [ticket-generator-service]
                                                                                                                               │
                                                                                                                               ├── (7. Genera PDF + QR)
                                                                                                                               ├── (8. Guarda en MinIO / S3)
                                                                                                                               └── (9. Notifica URL via Email)
```

### Paso a Paso del Flujo:
1. **Inicio de Reserva:** El comprador selecciona la entrada/asiento en la interfaz web (React + Vite) e ingresa los campos dinámicos solicitados (ej. Talla "M", Club de Running)[cite: 3]. El request atraviesa el *Spring Cloud API Gateway* validando el `tenant_id`[cite: 3].
2. **Bloqueo Concurrente y Hold Temporal:** `booking-service` (Spring Boot) inicia una transacción PostgreSQL e invoca un lock (`SELECT FOR UPDATE` o Lock Optimista sobre el asiento/cupo)[cite: 3]. Pasa el estado del cupo de `AVAILABLE` a `HELD` con un TTL (ej. 10 minutos) y realiza el `COMMIT`[cite: 3]. El bloqueo de fila en la DB dura solo milisegundos[cite: 3].
3. **Procesamiento de Pago Síncrono:** Fuera del lock de base de datos, `booking-service` invoca síncronamente a `payment-service` para cobro con la pasarela de pago[cite: 3].
4. **Confirmación de Reserva (Consistencia Fuerte):** Tras confirmarse el pago, `booking-service` ejecuta un `UPDATE seats SET status = 'SOLD' WHERE status = 'HELD' AND booking_id = ?` dentro de una breve transacción ACID[cite: 3].
5. **Desacoplamiento Event-Driven:** Inmediatamente después del `COMMIT`, `booking-service` emite el evento asíncrono `BookingConfirmed` al topic correspondiente en Apache Kafka[cite: 3].
6. **Generación y Entrega Asíncrona de Tickets:** El microservicio `ticket-generator-service` consume `BookingConfirmed`, construye el PDF con el código QR único y los atributos dinámicos del comprador, lo almacena de forma aislada en MinIO/S3 (`s3://eventhub-tickets/{tenant_id}/{booking_id}.pdf`) y dispara el envío del correo electrónico con la URL firmada de descarga[cite: 3].

---

## 6. Arquitectura y Factores Técnicos ($0 USD Budget Stack)

### 6.1 Principios Técnicos y Decisiones de Diseño
* **Frontend:** React 19 + Vite (Single Page Application responsiva, con renderizado dinámico de formularios mediante React Hook Form basado en el `formSchema` recibido del backend)[cite: 3].
* **Backend:** Java 21 + Spring Boot 3.x utilizando Arquitectura Hexagonal (Ports and Adapters) por microservicio[cite: 3].
* **Infraestructura Inmutable & IaC:** Terraform + Helm para orquestación de recursos de forma agnóstica a la nube y repetible[cite: 3].
* **Automatización Radical (CI/CD GitOps):** GitHub Actions para compilación, testing y construcción de imágenes Docker + ArgoCD para despliegue automatizado basado en repositorio de manifiestos[cite: 3].
* **Desacoplamiento de Configuración & Secretos:** `config-service` (MongoDB) para parámetros de negocio por tenant; HashiCorp Vault para gestión de secretos y claves JWT en tiempo de ejecución[cite: 3].
* **Elasticidad & Auto-Escalado (500+ Clientes Concurrentes):** Implementación de Kubernetes (`k3d` / `Minikube` local) con KEDA (Kubernetes Event-driven Autoscaling) y HPA (Horizontal Pod Autoscaler) para escalar pods basados en lag de Kafka y consumo métrico[cite: 3].
* **Observabilidad Centralizada:** LGTM Stack (Loki para logs, Grafana para tableros, Tempo para trazas distribuidas y Prometheus para métricas) con propagación de `trace_id` y `tenant_id` en headers HTTP y Kafka[cite: 3].

### 6.2 Matriz Tecnológica $0 USD

| Capa / Factor | Tecnología Adoptada ($0 USD / Open Source) | Propósito en la Arquitectura |
| :--- | :--- | :--- |
| **Frontend** | React 19 + Vite | Interfaz ultrarrápida para compradores y admins de tenant, con formularios dinámicos[cite: 3]. |
| **Backend Microservicios** | Java 21 + Spring Boot 3.x (Arquitectura Hexagonal) | Reglas de negocio aisladas, puertos/adaptadores limpios y alta eficiencia en runtime JVM[cite: 3]. |
| **Orquestación Local** | k3d / Minikube (Kubernetes sobre Docker) | Clúster K8s local multitarea de costo cero para simular producción con múltiples nodos[cite: 3]. |
| **Almacenamiento de Objetos** | MinIO (Self-hosted en K8s) | Reemplazo 100% compatible con AWS S3 para logos, marcas y PDFs de entradas con URLs firmadas[cite: 3]. |
| **Gestión de Secretos** | HashiCorp Vault (Community Edition) | Inyección dinámica de credenciales de base de datos y llaves de cifrado en tiempo de ejecución[cite: 3]. |
| **Bases de Datos & Broker** | PostgreSQL, MongoDB, Apache Kafka | Persistencia relacional ACID, persistencia NoSQL de esquemas flexibles y bus de eventos asíncrono[cite: 3]. |
| **IaC & GitOps** | Terraform + Helm + ArgoCD + GitHub Actions | Automatización completa desde el aprovisionamiento hasta el despliegue continuo inmutable[cite: 3]. |
| **Observabilidad** | OpenTelemetry + Prometheus + Grafana + Tempo + Loki | Trazabilidad distribuida end-to-end con filtrado multi-tenant y monitoreo de SLOs[cite: 3]. |

---

## 7. Estrategia y Organización de Repositorios

Para garantizar el desacoplamiento técnico y la integración continua (CI/CD) independiente por componente, la plataforma se estructura en una estrategia Multirrepo + Repositorios de Infraestructura:

### 7.1 Frontend
* `eventhub-frontend` (React + Vite): Portal público del comprador y panel administrativo del tenant.

### 7.2 Backend & Microservicios (Java 21 / Spring Boot)
* `eventhub-api-gateway`: Spring Cloud API Gateway, enrutamiento, validación JWT y propagación de `X-Tenant-Id`.
* `eventhub-auth-service`: Registro/autenticación, gestión de roles y emisión de JWT con claims de tenant.
* `eventhub-config-service`: Gestión de configuraciones por tenant, branding y esquemas de formularios (`formSchema`).
* `eventhub-catalog-service`: Catálogo de eventos, categorías, recintos y asociación con plantillas.
* `eventhub-booking-service`: Motor de reservas, gestión de estado (`AVAILABLE` → `HELD` → `SOLD`), locks ACID en Postgres y soporte `JSONB`.
* `eventhub-payment-service`: Integración / simulador síncrono de pasarela de pagos.
* `eventhub-ticket-generator-service`: Consumidor Kafka, generación de PDF con QR y almacenamiento en MinIO.
* `eventhub-analytics-service`: Consumidor de eventos para agregaciones CQRS y paneles de métricas.

### 7.3 Infraestructura y Operaciones (GitOps)
* `eventhub-dev-environment`: Archivo `docker-compose.yml` base y scripts de inicialización para desarrollo local.
* `eventhub-k8s-manifests`: Configuración de Terraform, Helm Charts de cada microservicio y manifiestos de Kubernetes observados por ArgoCD.

---

## 8. Plan de Inicio y Hoja de Ruta de Implementación

1. **Paso 1 — Entorno Local Base:** Crear el repositorio `eventhub-dev-environment` y configurar el `docker-compose.yml` con Postgres, MongoDB, Kafka y MinIO.
2. **Paso 2 — Primer Microservicio:** Crear `eventhub-config-service` con Java 21 / Spring Boot bajo Arquitectura Hexagonal para servir configuraciones y `formSchema`.
3. **Paso 3 — Seguridad & Routing:** Implementar `eventhub-auth-service` y `eventhub-api-gateway` para interceptar peticiones e inyectar el contexto del tenant (`X-Tenant-Id`).
4. **Paso 4 — Motor de Reservas & Pagos:** Construir `eventhub-catalog-service`, `eventhub-booking-service` (con locks en Postgres) y `eventhub-payment-service`.
5. **Paso 5 — Desacoplamiento Event-Driven:** Integrar el publicador Kafka en `booking-service` y construir `eventhub-ticket-generator-service` con soporte para MinIO.
6. **Paso 6 — Interfaz Web & Despliegue GitOps:** Desarrollar `eventhub-frontend` en React 19 y orquestar el clúster local con ArgoCD y Helm en `eventhub-k8s-manifests`.
