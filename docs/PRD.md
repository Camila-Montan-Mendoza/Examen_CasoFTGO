# PRD — FTGO Platform (Arquitectura Objetivo)

## 1. Contexto y objetivos

FTGO enfrenta problemas clásicos del infierno monolítico: escalado conflictivo entre módulos, builds lentos, falta de aislamiento de fallos y baja mantenibilidad [Brief §A.1] [Richardson Cap.1]. Desde hace varios años opera como una aplicación Java monolítica empaquetada como WAR, con una base de código que ha crecido exponencialmente impidiendo que equipos independientes avancen a velocidad [Brief §A.1].

La dirección de FTGO ha decidido **migrar a una arquitectura de microservicios** para sostener el crecimiento mediante escalado independiente, aislamiento de fallos y despliegues desacoplados. Este PRD documenta la **arquitectura objetivo** que guiará la migración incremental usando el patrón **Strangler Fig** [Brief §A.1] [Richardson Cap.2].

El objetivo es proporcionar claridad arquitectónica suficiente para que los equipos de desarrollo comiencen la migración gradual sin interrumpir la operación del monolito legacy.

---

## 2. Stakeholders

| Stakeholder | Necesidad principal | Interés |
|---|---|---|
| **Consumidor** | UX rápida y confiable | Experiencia fluida, <200 ms p95 latencia, tracking en tiempo real [Brief §A.2] |
| **Restaurante** | Gestión eficiente de carga | Dashboard de tickets, control de disponibilidad, rechazo de pedidos [Brief §A.2] |
| **Courier** | Asignaciones optimizadas | Ofertas cercanas, rutas optimizadas, pago confiable [Brief §A.2] |
| **Empleado FTGO** | Operabilidad y reportes | Visibilidad del sistema, resolución de incidentes, reportes de negocio [Brief §A.2] |
| **Equipo de arquitectura** | Calidad arquitectónica | Migración incremental, trazabilidad, mantenibilidad a largo plazo [Brief §A.1] |
| **Sistemas externos** | Integración estable | Stripe (pagos), Google Maps (rutas), SendGrid/Twilio (notificaciones) [Brief §A.2] |

Origen: [Brief §A.2]

---

## 3. Capacidades de negocio

Las siguientes 7 capacidades estables fueron identificadas como candidatos a microservicios [Brief §A.3]:

### 1. Consumer Management
**Responsabilidad**: Registro, perfiles, direcciones y preferencias de consumidores.  
**Objetivo**: Mantener datos de consumidores centralizados, permitir self-service, integrar con notificaciones.  
Origen: [Brief §A.3] [Richardson Cap.2]

### 2. Restaurant Management
**Responsabilidad**: Restaurantes registrados, menús, horarios y disponibilidad.  
**Objetivo**: Permitir restaurantes controlar su catálogo, horarios y capacidad de cocina.  
Origen: [Brief §A.3] [Richardson Cap.2]

### 3. Order Taking
**Responsabilidad**: Validación de pedidos, cálculo de totales, confirmación.  
**Objetivo**: Permitir consumidores realizar pedidos confiables desde el menú de un restaurante.  
Origen: [Brief §A.3] [US-01]

### 4. Order Fulfillment / Kitchen
**Responsabilidad**: Tickets entregados al restaurante, estado de preparación.  
**Objetivo**: Notificar restaurantes de nuevos pedidos y permitir aceptación/rechazo según carga.  
Origen: [Brief §A.3] [US-02]

### 5. Delivery
**Responsabilidad**: Asignación de couriers, rutas, tracking en tiempo real.  
**Objetivo**: Optimizar entregas asignando couriers cercanos y permitiendo tracking al consumidor.  
Origen: [Brief §A.3] [US-03]

### 6. Billing & Accounting
**Responsabilidad**: Cobros a consumidores, comisiones, payouts a restaurantes y couriers.  
**Objetivo**: Procesar pagos de forma confiable, segregar ingresos y gastos.  
Origen: [Brief §A.3] [Richardson Cap.3]

### 7. Notifications
**Responsabilidad**: Emails, SMS, push a consumidores, restaurantes, couriers.  
**Objetivo**: Notificar estados de pedido en tiempo real, confirmaciones y alertas.  
Origen: [Brief §A.3]

---

## 4. Requisitos no funcionales (NFRs)

### NFR-01: Latencia UX
- **Métrica**: ≤ 200 ms p95 para acciones del consumidor en app.
- **Justificación**: La experiencia móvil durante horarios pico (12:00-14:00, 19:00-22:00) debe mantenerse fluida para no perder conversiones.
- **Impacto arquitectónico**: Minimizar hops síncronos innecesarios, usar cachés agresivas, evitar llamadas bloqueantes a sistemas externos.
- Origen: [Brief §A.4 Latencia UX]

### NFR-02: Disponibilidad de toma de pedidos
- **Métrica**: ≥ 99.9 % mensual mínimo en el flujo crítico de toma de pedidos.
- **Justificación**: La capacidad de tomar pedidos es el corazón del negocio; degradación temporal aceptable en tracking.
- **Impacto arquitectónico**: Aislamiento de fallos por capacidad, circuit breakers en integraciones externas, redundancia en Order Taking.
- Origen: [Brief §A.4 Disponibilidad]

### NFR-03: Escalabilidad horizontal independiente
- **Métrica**: Cada capacidad puede escalarse en X-axis y Y-axis sin afectar otras.
- **Justificación**: Tráfico pico de 5x durante horarios de almuerzo y cena requiere escalado sin-acoplado de Delivery y Kitchen.
- **Impacto arquitectónico**: Servicios desacoplados, databases separadas, comunicación async preferida.
- Origen: [Brief §A.4 Carga, Escalabilidad horizontal]

### NFR-04: Tolerancia a fallos externos
- **Métrica**: El sistema puede tomar pedidos aunque Stripe esté caído (queue de retry); mapas pueden degradarse a modo fallback.
- **Justificación**: Dependencias externas (pagos, mapas) tienen SLAs independientes; FTGO no debe sufrir completamente.
- **Impacto arquitectónico**: Implementar async pagos con retry, circuit breakers, degradación elegante.
- Origen: [Brief §A.4 Tolerancia a fallos externos]

### NFR-05: Consistencia eventual con límites claros
- **Métrica**: Eventual consistency aceptada entre servicios; consistencia fuerte requerida dentro del aggregate de un pedido.
- **Justificación**: Reportes y analytics pueden estar ligeramente atrasados; pero un pedido nunca debe estar en un estado inválido.
- **Impacto arquitectónico**: Event sourcing para pedidos, sagas distribuidas para orquestación, bases de datos separadas por servicio.
- Origen: [Brief §A.4 Consistencia de datos]

### NFR-06: Trazabilidad distribuida
- **Métrica**: Cada acción del consumidor debe rastrearse end-to-end con correlation IDs y distributed tracing.
- **Justificación**: En una arquitectura distribuida, el debugging y auditoría deben ser predecibles.
- **Impacto arquitectónico**: Integración con APM (p.ej., Jaeger/DataDog), propagación de correlation IDs, logging centralizado.
- Origen: [Brief §A.4 Trazabilidad]

---

## 5. Alcance del laboratorio

### Incluye:
- Documentación clara de la arquitectura objetivo basada en microservicios.
- Definición de las 7 capacidades de negocio como servicios candidatos.
- 6 NFRs trazables con métricas explícitas.
- Decisiones arquitectónicas clave (ADRs): estilo arquitectónico, IPC/async, consistencia.
- Diagramas C4 (Nivel 1 y 2) que representen la arquitectura objetivo.
- Functional Specification Document (FSD) con ≥ 5 casos de uso.

### Excluye:
- Implementación física de microservicios.
- Pipelines CI/CD o infrastructure-as-code.
- Evaluación de plataformas de operación (Kubernetes, etc.).
- Benchmarks de rendimiento del monolito legacy.
- Roadmap detallado de migración (timelines).

### Restricciones del laboratorio:
- La migración es **incremental** usando Strangler Fig [Brief §A.1] durante 18-24 meses.
- El monolito legacy **coexiste temporalmente** con la arquitectura objetivo.
- No hay "big-bang rewrite" [Brief §A.4].
- Java/Spring Boot preferido para reutilizar conocimiento del equipo [Brief §A.4].
- Trazabilidad obligatoria: cada decisión debe rastrearse al brief, libro o user story.

Origen: [Brief §A.6] [Richardson Cap.2]

---

## 6. Resumen ejecutivo

FTGO requiere migración desde un monolito Java legacy hacia una arquitectura de microservicios desacoplados. La arquitectura objetivo debe soportar:

- **Escalabilidad horizontal independiente** de capacidades (Order Taking, Delivery, Kitchen).
- **Latencia < 200 ms p95** en acciones del consumidor durante tráfico pico de 5x.
- **99.9 % disponibilidad** en flujo crítico de toma de pedidos.
- **Tolerancia a fallos** de sistemas externos (Stripe, Google Maps).
- **Migración incremental** sin interrupciones operacionales.

Este PRD sirve como entrada para FSD, ADRs y diagramas C4 que detallarán la estrategia técnica.

---

## Referencias

- [Brief §A.1] — Contexto de negocio FTGO
- [Brief §A.2] — Stakeholders
- [Brief §A.3] — Capacidades de negocio
- [Brief §A.4] — Restricciones técnicas y NFRs
- [Brief §A.6] — Restricciones del laboratorio
- [Richardson Cap.1] — Monolithic Hell
- [Richardson Cap.2] — Decomposition by Business Capability
- [Richardson Cap.3] — Interprocess Communication
- [US-01] — Toma de pedido por consumidor
- [US-02] — Aceptación de tickets por restaurante
- [US-03] — Asignación de entrega al courier
