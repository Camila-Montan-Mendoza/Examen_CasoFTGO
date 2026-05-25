# ADR-0002 — Comunicación entre servicios e integración async

**Status**: Accepted  
**Fecha**: 2026-05-24  
**Autor**: Arquitecto FTGO  
**Revisores**: Equipo de arquitectura, Leads técnicos  

---

## 1. Contexto

El ADR-0001 decidió que FTGO migrará hacia **microservicios por capacidad de negocio** usando **Strangler Fig** [ADR-0001]. Ahora debe decidirse **cómo se comunican los servicios** entre sí y con sistemas externos.

Las capacidades identificadas en el PRD (Consumer, Restaurant, Order, Delivery, Billing, Notifications) necesitan coordinar:

- **UC-01 (Order Taking)**: Order Service debe validar contra Restaurant Service (¿restaurante disponible?).
- **UC-02 (Restaurant Ticket Acceptance)**: Kitchen Service notifica a Order Service del estado.
- **UC-03 (Courier Assignment)**: Delivery Service consulta Maps externo y notifica al consumidor.
- **UC-04 (Payment Processing)**: Billing Service intenta cobrar en Stripe y notifica Order Service del resultado.
- **UC-05 (Tracking)**: Delivery Service publica ubicación; Notification Service la retransmite al consumidor.
- **UC-06 (Notifications)**: Notification Service escucha cambios de estado de múltiples servicios.

**El problema arquitectónico**:
- ¿Usamos **REST síncrono** entre servicios? Rápido pero acoplado, requiere que ambos servicios estén up.
- ¿Usamos **mensajería async (Kafka, RabbitMQ)**? Desacoplado pero más complejo, eventual consistency.
- ¿Usamos **híbrido** (REST para queries, async para eventos)? Flexible pero requiere justificación clara.

La decisión debe considerar explícitamente:
- **Latencia p95 < 200 ms** [Brief §A.4 Latencia UX]
- **Tolerancia a fallos**: Stripe caído no debe bloquear toma de pedidos [Brief §A.4 Tolerancia a fallos externos]
- **Escalabilidad horizontal**: Delivery escalando a 100 instancias sin sobrecargar Billing [Brief §A.4 Escalabilidad]
- **Eventual consistency aceptada para reportes, fuerte en pedidos** [Brief §A.4 Consistencia]
- **Trazabilidad distribuida end-to-end** [Brief §A.4 Trazabilidad]

---

## 2. Opciones consideradas

### Opción 1: REST síncrono directo (choreography)

**Descripción**  
Los servicios se comunican directamente vía HTTP/JSON o gRPC. Cada servicio tiene la lógica de orquestación:
- Order Service llama a Restaurant Service (bloquea hasta respuesta).
- Order Service llama a Billing Service (bloquea hasta respuesta).
- Billing Service llama a Stripe (bloquea hasta respuesta).
- Si una llamada falla, el servicio retorna error.

Ejemplo flujo:
```
POST /orders (Order Service)
  → GET /restaurants/{id}/availability (Restaurant Service) [bloqueante]
  → POST /charges (Stripe via Billing Service) [bloqueante]
  → Response 200 OK
```

**Pros**
- **Simplicidad de implementación**: REST es bien conocido, SDK de Stripe es directo.
- **Latencia predecible en happy path**: 3 llamadas síncronas = ~30-50 ms (asumiendo 10 ms por call).
- **Consistencia inmediata**: conocemos el resultado de inmediato. Si Stripe acepta, Order está listo.
- **Debugging fácil**: stack trace en el mismo proceso.
- **Monitoreo simple**: HTTP status codes estándar.

**Contras**
- **Acoplamiento temporal**: si Restaurant Service está down (lentitud, restart), Order Service bloquea y sus usuarios ven timeout.
- **Cascada de fallos**: si Stripe es lento (100 ms), cada Order tarda mínimo 100 ms, escalabilidad sufre.
- **Escalabilidad limitada**: si Delivery Service hace 100 llamadas/segundo a Billing Service (para reportes), Billing debe manejar 100 req/seg síncronos. [Brief §A.4 Escalabilidad horizontal]
- **Tolerancia a fallos débil**: Stripe caído = toma de pedidos caída. No hay retry elegante. [Brief §A.4 Tolerancia a fallos externos]
- **No hay desacoplamiento**: cambios en contrato de Restaurant Service rompen Order Service.
- **Timeout hell**: ¿cuánto timeout? 5s? 30s? Si es corto, errores frecuentes; si es largo, UX mala.

**Impacto en NFRs**
- Latencia UX < 200 ms p95: ⚠ (depende de latencia de servicios dependientes)
- Disponibilidad 99.9%: ✗ (cascada de fallos reduce disponibilidad)
- Escalabilidad: ✗ (acoplamiento temporal limita escalado independiente)
- Tolerancia a fallos: ✗ (dependencias síncronas fuerzan fallos en cascada)
- Trazabilidad: ✓ (stack trace directo, fácil seguir)

**Trazabilidad**
- [Brief §A.4 Latencia UX] — Puede cumplirse en happy path, pero riesgo con dependencias lentes.
- [Brief §A.4 Tolerancia a fallos externos] — No soluciona (Stripe lento = pedidos lentos).
- [Richardson Cap.3] — Síncrono es patrón básico, pero desaconsejado para systems críticos.

---

### Opción 2: Mensajería async con event sourcing

**Descripción**  
Los servicios **no se comunican directamente**. Cada cambio importante es un **evento** publicado en un broker (Kafka, RabbitMQ). Otros servicios suscriben a eventos que los interesan:

Flujo Order:
```
POST /orders (Order Service)
  → valida consumidor localmente [BD Consumer Service replicada]
  → genera Order en state PENDING_PAYMENT
  → publica evento OrderCreated {order_id, amount, payment_method}
  → responde al cliente 200 OK (orden registrada)

Kafka topic: orders.created
  Subscribers:
  - Billing Service: escucha, intenta cobro async
  - Notification Service: escucha, envía confirmación
  - Kitchen Service: escucha preparación

Billing Service (async):
  → recibe OrderCreated
  → intenta Stripe.charge() con retry exponencial
  → si éxito: publica PaymentProcessed {order_id, transaction_id}
  → si fail: publica PaymentFailed {order_id, reason}

Kitchen Service (async):
  → recibe PaymentProcessed
  → publica TicketCreated {order_id}

Order Service:
  → escucha PaymentProcessed
  → actualiza estado Order a PAID
  → escucha TicketCreated
  → notifica: "Tu pedido fue aceptado"
```

**Pros**
- **Desacoplamiento completo**: servicios no conocen dirección HTTP de otros. Agregar nuevo servicio que escuche OrderCreated es trivial.
- **Tolerancia a fallos robusta**: si Billing Service está down, eventos se acumulan en Kafka, Order Taking sigue funcionando. Cuando Billing recupera, procesa backlog. [Brief §A.4 Tolerancia a fallos externos]
- **Escalabilidad horizontal óptima**: Delivery Service con 100 instancias consume eventos de Kafka (particionado por order_id), Billing puede tener 5 instancias. Independientes.
- **Retry elegante**: Kafka reintenta automáticamente, Billing puede implementar exponential backoff sin afectar a Order.
- **Eventual consistency manejable**: Order sabe que Billing está procesando, puede mostrar estado "pago pendiente" al usuario.
- **Event sourcing nativo**: cada evento es inmutable, perfecto para auditoría y debugging.
- **Escalado del broker**: Kafka escala a millones de eventos/segundo.

**Contras**
- **Complejidad operacional**: gestionar Kafka (replication, partitions, consumer groups, monitoring).
- **Eventual consistency requiere handling**: no sabes si el pago fue aceptado cuando retornas 200 OK. Consumidor debe polling o websockets. [Brief §A.4 Consistencia]
- **Debugging más difícil**: evento publicado a las 10:00, procesado a las 10:05 (si Kafka estuvo caído). Correlación de eventos compleja.
- **Latencia no determinística**: happy path puede ser 200 ms (evento llega rápido a Billing), pero en caso de reintento puede ser minutos.
- **Cambios de contrato complejos**: agregar nuevo campo a evento requiere planning (versioning, migration).
- **Duplicación de datos**: si Delivery Service necesita nombre del restaurante, debe replicarlo (de Restaurant Service) o consultarlo (REST síncrono, acoplamiento). [Brief §A.4 Consistencia]

**Impacto en NFRs**
- Latencia UX < 200 ms p95: ✓ (Order devuelve 200 sin esperar Billing)
- Disponibilidad 99.9%: ✓ (servicios independientes, Kafka es resiliente)
- Escalabilidad horizontal: ✓ (particionado por event key)
- Tolerancia a fallos: ✓ (servicios desacoplados, retry automático)
- Eventual consistency: ✓ (totalmente soportado)
- Trazabilidad: ⚠ (requiere correlation IDs propagados en eventos)

**Trazabilidad**
- [Brief §A.4 Latencia UX] — Cumplido (Order devuelve inmediatamente).
- [Brief §A.4 Tolerancia a fallos externos] — Cumplido (Stripe caído no afecta Order).
- [Brief §A.4 Escalabilidad horizontal] — Cumplido (Kafka escalable).
- [Brief §A.4 Consistencia] — Cumplido (eventual consistency explícita).
- [Richardson Cap.3] — Messaging and publish-subscribe.
- [Richardson Cap.4] — Saga Pattern con messaging.

---

### Opción 3: Arquitectura híbrida (REST para queries + async para eventos)

**Descripción**  
Combinar lo mejor de ambas:
- **Queries síncronas vía REST**: cuando necesitas respuesta inmediata, consulta vía HTTP (p.ej., Order Service consultando Restaurant Service para disponibilidad).
- **Eventos async vía Kafka**: para notificaciones y cambios de estado (p.ej., OrderCreated, PaymentProcessed).

Flujo Order híbrido:
```
POST /orders (Order Service)
  → GET /restaurants/{id}/availability (Restaurant Service REST) [bloqueante, 10 ms]
  → crea Order en PENDING_PAYMENT
  → publica evento OrderCreated a Kafka
  → responde 200 OK

[Async]
Billing Service escucha OrderCreated:
  → intenta Stripe.charge()
  → publica PaymentProcessed (async)

[Async]
Order Service escucha PaymentProcessed:
  → actualiza Order a PAID
```

**Pros**
- **Flexibilidad**: queries críticas (¿restaurante disponible?) son síncronas (rápido, consistencia inmediata). Integraciones lentas (pagos) son async.
- **Latencia mejorada**: no esperas a Stripe; respuesta de Order es rápida.
- **Acoplamiento moderado**: consultas REST permiten cache en cliente, fallback graceful. Eventos permiten desacoplamiento de notificaciones.
- **Debugging mixto**: queries síncronas tienen stack trace, eventos tienen correlation IDs.
- **Transiciones realistas**: migración incremental de REST → async es posible (primero async eventos, después mover queries a async si necesario).

**Contras**
- **Complejidad de decisión**: ¿cuándo usar REST, cuándo async? Requiere guía clara.
- **Riesgo de "worst of both"**: si usas REST para todo, vuelves a Opción 1. Si usas async para todo, vuelves a Opción 2, pero con REST innecesario.
- **Synchronize en boundaries**: servicios deben sincronizar datos cuando consultan vía REST (p.ej., disponibilidad de restaurante); si cambia mientras Order crea, inconsistencia.
- **Testing complejo**: tests deben cobertura REST + async, con timing.
- **Observabilidad split**: algunos flujos son síncronos (fácil tracing), otros async (complejo).

**Impacto en NFRs**
- Latencia UX < 200 ms p95: ✓ (queries rápidas, eventos async no bloquean)
- Disponibilidad 99.9%: ✓ (async tolera fallos)
- Escalabilidad: ✓ (queries pueden cacharse, eventos escalables)
- Tolerancia a fallos: ✓ (async tolera, REST puede fallar pero con circuit breaker)
- Eventual consistency: ✓ (si se diseña bien)
- Trazabilidad: ✓ (correlation IDs en REST y async)

**Trazabilidad**
- [Brief §A.4 Latencia UX] — Cumplido.
- [Brief §A.4 Tolerancia a fallos] — Cumplido con circuit breakers en REST.
- [Richardson Cap.3] — Combina messaging + REST calls.

---

## 3. Decisión

**Se selecciona: Opción 3 — Arquitectura híbrida con REST para queries y async/eventos para notificaciones.**

**Justificación**

La Opción 1 (REST puro) **incumple tolerancia a fallos** [Brief §A.4 Tolerancia a fallos externos]: si Stripe está caído, Order Taking no puede bloquear. Escalabilidad también sufre [Brief §A.4 Escalabilidad horizontal]: si Delivery escala a 100 instancias, cada una consultando Billing síncronamente, colapsamos Billing.

La Opción 2 (async puro) **es más resiliente y escalable**, pero introduce **eventual consistency** que requiere handling complejo en la UX. Aunque [Brief §A.4 Consistencia] acepta eventual consistency, algunos flujos **requieren consistencia inmediata**: cuando Order Taking confirma un pedido, debe validar que el restaurante existe y está disponible **ahora**, no "eventualmente" [UC-01 FSD].

La Opción 3 (híbrida) **balancea ambos mundos**:

1. **REST síncrono para queries de validación crítica**:
   - Order Service → Restaurant Service (¿restaurante disponible?)
   - Order Service → Consumer Service (¿consumidor existe?)
   - Delivery Service → Maps (¿qué couriers están cerca?)
   
   Estos son **queries**, respuestas rápidas (<50 ms), no generan cambios de estado, pueden cachearse. Si fallan, ofrecemos fallback graceful.

2. **Async/Eventos para cambios de estado y notificaciones**:
   - OrderCreated (Billing, Notification, Kitchen escuchan)
   - PaymentProcessed (Order, Notification escuchan)
   - CourierAssigned (Notification escucha)
   - OrderDelivered (Notification escucha)
   
   Estos son **eventos**, desacoplados, tolerantes a fallos. Stripe caído no bloquea Order Taking.

3. **Kafka como bus de eventos central**:
   - Todos los servicios publican eventos importantes a Kafka.
   - Topics: `orders.created`, `payments.processed`, `delivery.assigned`, etc.
   - Consumers: múltiples servicios pueden escuchar el mismo evento.
   - Particiones: por `order_id` para garantizar orden de eventos del mismo pedido.

4. **Circuit breakers en REST**:
   - Si Restaurant Service no responde en 1s, Circuit Breaker abre.
   - Fallback: asumir disponibilidad por defecto (optimista), O rechazar pedido (pesimista).
   - Monitorear y alertar.

**Decisión técnica concreta**:

| Tipo de comunicación | Mecanismo | Usado para | Justificación |
|---|---|---|---|
| **Query** | REST síncrono | Consumer lookup, Restaurant availability, Maps API | Rápido, inmediato, cacheble |
| **Comando** | REST síncrono + async fallback | Crear Order (respuesta inmediata), procesar pago (fallback async) | Inmediatez + resiliencia |
| **Evento** | Kafka (pub/sub async) | OrderCreated, PaymentProcessed, CourierAssigned, OrderDelivered | Desacoplamiento, tolerancia a fallos |
| **Notificación** | Kafka (async) | Notificaciones a consumidores (email, SMS, push) | Desacoplado, resiliente |

**Kafka como backbone**:
- **Broker**: Apache Kafka o AWS MSK.
- **Topics**: 
  - `orders.created` (partition key: order_id)
  - `orders.payment_processed` (partition key: order_id)
  - `orders.confirmed` (partition key: order_id)
  - `delivery.assigned` (partition key: order_id)
  - `delivery.arrived` (partition key: order_id)
  - Otros según demanda.
- **Consumer groups**: cada servicio es consumer group (Notification Service, Order Service, Kitchen Service).
- **Replication factor**: 3 (tolerancia a 2 fallos de broker).
- **Retention**: 7 días (para replay, debugging).

---

## 4. Consecuencias

### Consecuencias positivas

1. **Latencia UX optimizada**  
   Order devuelve 200 OK en < 100 ms (sin esperar a Stripe). Consumidor ve pedido confirmado inmediatamente, luego Billing procesa async. [Brief §A.4 Latencia UX ≤ 200 ms]

2. **Tolerancia a fallos Stripe/Maps**  
   Si Stripe está caído:
   - Order Taking sigue aceptando pedidos.
   - Billing Service reintenta con backoff exponencial.
   - Cuando Stripe se recupera, procesa queue de pagos pendientes.
   [Brief §A.4 Tolerancia a fallos externos]

3. **Escalabilidad horizontal independiente**  
   Delivery Service con 100 instancias consuming eventos del topic `delivery.assigned` (particionado). Billing Service con 3 instancias, ambas independientes. [Brief §A.4 Escalabilidad horizontal]

4. **Desacoplamiento de notificaciones**  
   Agregar nuevo canal de notificación (Slack, Discord) es tan fácil como agregar nuevo consumer a Kafka que escuche `orders.*` eventos.

5. **Auditoría y debugging mejorados**  
   Cada evento en Kafka es inmutable, timestamped, con correlation ID. Replay de eventos reproduce exactamente qué pasó.

6. **Resiliencia de cascada**  
   Fallos en servicios no propagaban. Si Kitchen Service está down, otros servicios siguen. Kitchen consume backlog de eventos cuando recupera.

### Consecuencias negativas

1. **Eventual consistency requiere cambios en UX**  
   Consumidor confirma orden, ve "orden creada". Pocos segundos después ve "pago procesado" o "pago fallido". No es instantáneo.
   - Mitigation: usar WebSockets o polling para updates en tiempo real.

2. **Complejidad operacional: Kafka**  
   Gestionar Kafka es responsabilidad operativa:
   - Replicación, rebalancing de consumers.
   - Monitoreo de consumer lag.
   - Alertas si topics están backed up.
   - Requiere expertise o SaaS (AWS MSK, Confluent Cloud).

3. **Duplicación de datos**  
   Si Delivery Service necesita saber si restaurante está abierto (para decidir si aceptar pedido), debe:
   - Opción A: replicar datos de Restaurant Service (dual writes, eventual consistency).
   - Opción B: consultar REST síncrono a Restaurant Service (acoplamiento).
   - Opción C: publicar evento `RestaurantStatusChanged` y mantener réplica eventual. (Opción elegida)

4. **Testing más complejo**  
   Tests deben cobertura:
   - REST calls síncronas (fast).
   - Eventos async (requiere testcontainers, Kafka embebido, wait conditions).
   - Timing (qué pasa si Kafka tarda 2 minutos en entregar evento).

5. **Observabilidad distribuida obligatoria**  
   Sin distributed tracing (Jaeger, DataDog) es imposible debuguear. Correlation IDs deben propagarse en REST headers y en payloads de Kafka.
   - Inversión: APM tool (costo ~$500-5000/mes).

6. **Versionamiento de eventos**  
   Cambiar schema de evento requiere backward compatibility planning. Si agregamos campo a `OrderCreated`, servicios viejos deben ignorarlo, nuevos deben soportarlo.

---

## 5. Decisiones relacionadas y follow-ups

1. **ADR-0003: Estrategia de datos y consistencia**  
   - Database-per-service: cómo replicar datos de un servicio a otro.
   - CDC (Change Data Capture): del monolito a servicios nuevos.
   - Sagas distribuidas: cómo orquestar transacciones cross-service (Order + Payment + Delivery).

2. **ADR-0004: Versionamiento de APIs y eventos**  
   - Semantic versioning para REST APIs (v1, v2).
   - Schema versioning para eventos Kafka (compatible backwards).

3. **API Gateway y configuración**  
   - Seleccionar API Gateway (Kong, Spring Cloud Gateway).
   - Configurar rate limiting, authentication, routing por servicio.

4. **Observabilidad y APM**  
   - Seleccionar APM (Jaeger, DataDog, New Relic).
   - Instrumentar todas las REST calls y eventos Kafka.
   - Correlation ID propagation.

5. **Testeo de Kafka**  
   - Testcontainers para tests de integración.
   - Chaos engineering: qué pasa si Kafka broker cae.
   - Consumer lag monitoring en producción.

6. **POCs recomendados**:
   - Semana 1-2: Levantar Kafka local, crear topics.
   - Semana 3-4: Implementar productor de eventos en Order Service, consumer en Notification Service.
   - Semana 5: Medir latencia REST queries + async events.
   - Semana 6: Test de resiliencia (derribar servicios, validar que otros sigan funcionando).

---

## 6. Impacto en NFRs

| NFR | Cumplimiento | Justificación |
|---|---|---|
| **Latencia < 200 ms p95** | ✓ | REST queries rápidas, eventos async no bloquean. |
| **Disponibilidad 99.9%** | ✓ | Servicios desacoplados, tolerancia a fallos independiente. |
| **Escalabilidad horizontal** | ✓ | Kafka particionado, servicios escalables independientemente. |
| **Tolerancia a fallos externos** | ✓ | Stripe/Maps caídos no bloquean Order Taking. |
| **Eventual consistency** | ✓ | Eventos async soportan eventual consistency. |
| **Trazabilidad distribuida** | ⚠ | Requiere correlation IDs propagados (no automático). |
| **Migración incremental** | ✓ | Hybrid permite REST→async gradualmente. |

---

## 7. Referencias

- [Brief §A.4] — Latencia, tolerancia a fallos, escalabilidad, consistencia.
- [Brief §A.6] — Migración incremental.
- [ADR-0001] — Estilo arquitectónico: microservicios.
- [PRD NFR-01 a NFR-06] — Requisitos no funcionales.
- [FSD UC-01 a UC-06] — Functional specification, flujos.
- [Richardson Cap.3] — Interprocess Communication in a Microservice Architecture.
- [Richardson Cap.4] — Managing Transactions with Sagas.
- Kafka documentation: https://kafka.apache.org/
- Circuit Breaker pattern: https://martinfowler.com/bliki/CircuitBreaker.html
