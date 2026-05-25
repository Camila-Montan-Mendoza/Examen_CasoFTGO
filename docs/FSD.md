# FSD — FTGO Functional Specification Document

## 1. Introducción

Este Functional Specification Document detalla los casos de uso funcionales de FTGO requeridos para soportar la arquitectura objetivo de microservicios. El FSD establece los requisitos funcionales derivados del PRD y las 3 user stories semilla del brief, extendiendo con 2 UCs adicionales justificables desde la arquitectura y el contexto de migración. Cada UC está formalizado en formato Given/When/Then (BDD) con trazabilidad explícita a capacidades del PRD y referencias a Richardson.

Origen: [PRD] [Brief §A.5]

---

## 2. Resumen de Casos de Uso

| ID | Título | Actor primario | Capacidad PRD | Origen |
|---|---|---|---|---|
| UC-01 | Tomar pedido desde el menú | Consumidor | Order Taking | [US-01] |
| UC-02 | Aceptar/rechazar ticket en restaurante | Restaurante | Order Fulfillment/Kitchen | [US-02] |
| UC-03 | Asignar pedido a courier | Sistema (Delivery) | Delivery | [US-03] |
| UC-04 | Procesar pago del pedido | Sistema (Billing) | Billing & Accounting | [Brief §A.3] [Richardson Cap.3] |
| UC-05 | Tracking en tiempo real del pedido | Consumidor | Delivery | [PRD NFR-01] |
| UC-06 | Notificación de estado del pedido | Sistema (Notifications) | Notifications | [Brief §A.3] |

---

## 3. Detalle de Casos de Uso

---

### UC-01: Tomar pedido desde el menú

| Campo | Valor |
|---|---|
| Actor primario | Consumidor |
| Capacidad PRD | Order Taking |
| Origen | [US-01] [PRD Capacidad 3] |

**Precondiciones**
- El consumidor está autenticado en la app móvil.
- El consumidor ha seleccionado un restaurante disponible.
- El restaurante tiene menú cargado y disponible.
- El restaurante está dentro del radio de cobertura de FTGO.

**Flujo principal**
1. El consumidor visualiza el menú del restaurante seleccionado (categorías, precios, descripción).
2. El consumidor agrega uno o más items al carrito.
3. El sistema valida disponibilidad de cada item en stock.
4. El consumidor ingresa dirección de entrega (o selecciona una guardada).
5. El sistema calcula subtotal, impuestos, delivery fee, total final.
6. El consumidor selecciona método de pago y confirma el pedido.
7. El sistema valida datos de pago (sin procesamiento en este UC, delegado a UC-04).
8. El sistema genera número de pedido único (PO-XXXXXXXXX).
9. Se registra el pedido con estado = "PENDING_PAYMENT".
10. El consumidor recibe confirmación con número de pedido.

**Flujos alternativos**
- **FA-01**: Restaurante no disponible.
  - Pasos 1-2: Si el restaurante está cerrado o fuera de horario, mostrar mensaje "Restaurante cerrado" y ofrecer alternativas.
  
- **FA-02**: Item sin stock.
  - Paso 3: Si un item fue agotado recientemente, notificar al consumidor y ofrecerle remover o reemplazar.
  
- **FA-03**: Dirección fuera de cobertura.
  - Paso 4: Si la dirección está fuera del radio de entrega, mostrar error y pedir ajuste.

**Postcondiciones**
- El pedido está registrado en la base de datos con estado PENDING_PAYMENT.
- Se ha generado un número de pedido único.
- El consumidor recibe confirmación en la app con detalles del pedido.
- Se publica evento `OrderCreated` para iniciar pipeline de ejecución.

**Given/When/Then**
- **Given**: Un consumidor autenticado con carrito válido (2 items, dirección válida, restaurante disponible).
- **When**: El consumidor confirma el pedido con método de pago válido.
- **Then**: El sistema registra el pedido con PO único, estado PENDING_PAYMENT, y publica evento OrderCreated con correlation ID.

Origen: [US-01] [Richardson Cap.2]

---

### UC-02: Aceptar/rechazar ticket en restaurante

| Campo | Valor |
|---|---|
| Actor primario | Restaurante |
| Capacidad PRD | Order Fulfillment / Kitchen |
| Origen | [US-02] [PRD Capacidad 4] |

**Precondiciones**
- El restaurante está autenticado en su dashboard.
- Existe un pedido en estado PENDING_CONFIRMATION (recién creado desde UC-01).
- El restaurante está operando dentro de su horario habitual.
- El pedido contiene items que el restaurante puede preparar.

**Flujo principal**
1. El sistema envía notificación del nuevo pedido al restaurante (push, audio, visual).
2. El restaurante visualiza el ticket con detalles: items, cantidad, especiales, dirección entrega.
3. El restaurante estima tiempo de preparación (p.ej., 20 minutos).
4. El restaurante acepta el pedido con tiempo estimado o rechaza con motivo.
5. Si acepta: el sistema actualiza estado a CONFIRMED_BY_RESTAURANT, inicia timer de preparación.
6. Si rechaza: el sistema actualiza estado a REJECTED_BY_RESTAURANT, notifica consumidor y libera recursos.

**Flujos alternativos**
- **FA-01**: Timeout de aceptación.
  - Paso 4: Si el restaurante no responde en 120 segundos, el pedido se auto-asigna con tiempo estimado por defecto y se notifica al restaurante.
  
- **FA-02**: Aceptación parcial (algunos items no disponibles).
  - Paso 3: El restaurante puede aceptar pero marcar ciertos items como "sin stock", ofreciendo alternativas al consumidor.

**Postcondiciones**
- El pedido está en estado CONFIRMED_BY_RESTAURANT o REJECTED_BY_RESTAURANT.
- Si aceptado: el restaurante ve timer de cocina contando atrás.
- Si rechazado: el consumidor es notificado automáticamente.
- Se publica evento `RestaurantTicketAccepted` o `RestaurantTicketRejected` con correlation ID.

**Given/When/Then**
- **Given**: Un pedido en estado PENDING_CONFIRMATION, restaurante disponible.
- **When**: El restaurante acepta el pedido con tiempo estimado = 20 minutos.
- **Then**: El sistema actualiza estado a CONFIRMED_BY_RESTAURANT, inicia timer, y publica `RestaurantTicketAccepted` con PO y ETA.

Origen: [US-02] [Richardson Cap.4 - Saga Pattern]

---

### UC-03: Asignar pedido a courier

| Campo | Valor |
|---|---|
| Actor primario | Sistema (Delivery Service) |
| Capacidad PRD | Delivery |
| Origen | [US-03] [PRD Capacidad 5] |

**Precondiciones**
- El pedido está en estado CONFIRMED_BY_RESTAURANT.
- Existe al menos 1 courier disponible en la app (estado = AVAILABLE).
- El courier está dentro de un radio razonable del restaurante (p.ej., 2 km).
- El pedido tiene ubicación del restaurante y consumidor disponible.

**Flujo principal**
1. El sistema identifica couriers disponibles cercanos al restaurante (usando Google Maps).
2. El sistema calcula distancia y ruta optimizada restaurante → consumidor para cada courier.
3. El sistema ordena couriers por distancia al restaurante (cercano primero).
4. El sistema envía oferta a courier más cercano: ubicación restaurante, ubicación entrega, tarifa estimada, timeout = 30 seg.
5. Si el courier acepta en < 30 seg:
   - Estado del pedido = ASSIGNED_TO_COURIER.
   - Courier visualiza ruta optimizada.
   - Sistema publica evento `CourierAssigned`.
6. Si el courier rechaza o timeout:
   - Sistema intenta con siguiente courier más cercano (volvemos a paso 4).
7. Si se agotan couriers disponibles sin aceptación:
   - Estado del pedido = WAITING_FOR_COURIER.
   - Sistema reintenta automáticamente cada 10 segundos.

**Flujos alternativos**
- **FA-01**: Múltiples rechazos consecutivos.
  - Paso 7: Después de 3 reintentros sin asignación, notificar al consumidor "Demora en búsqueda de courier" y ofrecer cancelación.
  
- **FA-02**: Cambio de ubicación del courier.
  - Durante pasos 1-4: Si un courier marcado como disponible se desconecta, excluirlo y pasar al siguiente.

**Postcondiciones**
- El pedido está en estado ASSIGNED_TO_COURIER o WAITING_FOR_COURIER.
- Si asignado: el courier ve ruta en su mapa, consumidor puede ver tracking.
- Se publica evento `CourierAssigned` con courier ID, ETA de recogida, correlación end-to-end.

**Given/When/Then**
- **Given**: Un pedido confirmado por restaurante, 2+ couriers disponibles a <2 km, restaurante y consumidor con ubicaciones válidas.
- **When**: El sistema asigna el pedido al courier más cercano, quien acepta en < 30 segundos.
- **Then**: El sistema actualiza estado a ASSIGNED_TO_COURIER, calcula ETA, y publica `CourierAssigned` con routing info.

Origen: [US-03] [Richardson Cap.5 - API Gateway + Service Discovery]

---

### UC-04: Procesar pago del pedido

| Campo | Valor |
|---|---|
| Actor primario | Sistema (Billing Service) |
| Capacidad PRD | Billing & Accounting |
| Origen | [Brief §A.3] [Richardson Cap.3 - Saga Pattern] |

**Precondiciones**
- El pedido está en estado PENDING_PAYMENT (desde UC-01).
- El consumidor ha proporcionado método de pago válido (tarjeta, wallet, etc.).
- Stripe API está disponible o hay mecanismo de retry async.
- El monto total es válido (no negativo, dentro de límites).

**Flujo principal**
1. El Billing Service recibe evento `OrderCreated` con detalles del pedido y método de pago.
2. El Billing Service valida que el monto sea sensato y que el consumidor no esté en "lista negra".
3. El Billing Service envía solicitud de pago a Stripe con idempotency key = PO (para evitar duplicados).
4. Stripe responde con CHARGE_SUCCESSFUL o CHARGE_FAILED en tiempo real (u horaio de bajo).
5. Si CHARGE_SUCCESSFUL:
   - El Billing Service registra la transacción.
   - Se publica evento `PaymentProcessed` con PO, monto, transacción ID.
   - Pedido avanza a estado PAYMENT_CONFIRMED.
6. Si CHARGE_FAILED:
   - Se publica evento `PaymentFailed` con motivo (fondos insuficientes, tarjeta expirada, etc.).
   - El Billing Service intenta notificar al consumidor via SMS/push.
   - Pedido entra en estado PAYMENT_FAILED.

**Flujos alternativos**
- **FA-01**: Stripe no responde (timeout o 5xx).
  - Paso 3: El Billing Service encola el pago para retry async (exponential backoff).
  - Consumidor recibe mensaje: "Procesando pago, por favor espera...".
  - Si se confirma después: evento `PaymentProcessed` se publica con delay.
  
- **FA-02**: Fraud detection por Stripe.
  - Paso 4: Stripe responde con CHARGE_FLAGGED_FOR_REVIEW.
  - El Billing Service notifica a back-office FTGO para revisión manual.
  - Pedido entra en estado PAYMENT_UNDER_REVIEW.

**Postcondiciones**
- El pedido está en estado PAYMENT_CONFIRMED, PAYMENT_FAILED, o PAYMENT_UNDER_REVIEW.
- Si PAYMENT_CONFIRMED: se inicia flujo UC-02 (restaurante recibe ticket).
- Si PAYMENT_FAILED: consumidor es notificado con opción de reintento o cancelación.
- Todas las transacciones están auditadas con correlation ID.

**Given/When/Then**
- **Given**: Un pedido con monto = $25.00, consumidor con tarjeta válida pre-registrada, Stripe disponible.
- **When**: El Billing Service procesa el pago con idempotency key = PO.
- **Then**: Stripe responde CHARGE_SUCCESSFUL, evento `PaymentProcessed` se publica, pedido pasa a PAYMENT_CONFIRMED.

Origen: [Richardson Cap.3] [Brief §A.4 Tolerancia a fallos externos]

---

### UC-05: Tracking en tiempo real del pedido

| Campo | Valor |
|---|---|
| Actor primario | Consumidor |
| Capacidad PRD | Delivery |
| Origen | [Brief §A.5 - Aceptación esperada] [PRD NFR-01] |

**Precondiciones**
- El pedido está en estado ASSIGNED_TO_COURIER o posterior (siendo entregado).
- El consumidor está en la app móvil.
- El courier está actualmente transportando el pedido (tiene ubicación).
- Google Maps API está disponible.

**Flujo principal**
1. El consumidor abre la pantalla de tracking del pedido (PO conocido).
2. El sistema consulta ubicación actual del courier (via Delivery Service, actualizada cada 5-10 segundos desde GPS del courier).
3. El sistema consulta destino (dirección del consumidor) y calcula ETA en tiempo real (Google Maps).
4. El sistema renderiza mapa interactivo:
   - Ubicación del courier (marcador dinámico).
   - Dirección del consumidor (marcador fijo).
   - Ruta estimada.
   - ETA actual.
   - Estado: "El courier está a 2.3 km, ETA 8 minutos".
5. El sistema actualiza la pantalla cada 5-10 segundos con nueva ubicación del courier.
6. Cuando el courier llega a destino (< 50 metros), notificar al consumidor "El courier está aquí".

**Flujos alternativos**
- **FA-01**: Courier pierde conexión GPS/internet.
  - Paso 2: El sistema muestra última ubicación conocida con timestamp. ETA se marca como "Estimada, puede variar".
  - Si la desconexión dura > 60 segundos, notificar al consumidor: "Perdimos el contacto, reintentando...".
  
- **FA-02**: Google Maps está degradado.
  - Paso 3: El sistema muestra ETA estática basada en cálculo anterior (sin refresco).
  - Mostrar advertencia: "Información de rutas no disponible, ETA puede no ser exacta".

**Postcondiciones**
- El consumidor ve mapa con ubicación del courier en tiempo real o degradado.
- ETA se actualiza continuamente (o con fallback si es necesario).
- Se publican eventos de ubicación con correlation ID para auditoría.

**Given/When/Then**
- **Given**: Un pedido en estado ASSIGNED_TO_COURIER, courier está a 2 km del consumidor, app del consumidor abierta.
- **When**: El consumidor abre pantalla de tracking.
- **Then**: El sistema muestra mapa con ubicación del courier, ruta, y ETA = 8 minutos, actualizado cada 10 segundos.

Origen: [Brief §A.5] [Richardson Cap.5]

---

### UC-06: Notificación de estado del pedido

| Campo | Valor |
|---|---|
| Actor primario | Sistema (Notification Service) |
| Capacidad PRD | Notifications |
| Origen | [Brief §A.3 Capacidad 7] [Richardson Cap.3 - Async communication] |

**Precondiciones**
- El consumidor tiene preferencias de notificación configuradas (push sí, SMS sí, email sí/no).
- El sistema ha capturado canales de contacto: teléfono móvil, email, push tokens.
- Existen eventos de cambio de estado del pedido (UC-01, UC-02, UC-03, UC-04, UC-05).

**Flujo principal**
1. El Notification Service escucha eventos sobre el pedido:
   - `OrderCreated` (Paso UC-01).
   - `PaymentProcessed` (Paso UC-04).
   - `RestaurantTicketAccepted` (Paso UC-02).
   - `CourierAssigned` (Paso UC-03).
   - `CourierArrived` (Paso UC-05).
   - `OrderDelivered`.

2. Para cada evento, el Notification Service:
   - Consulta preferencias del consumidor (push/SMS/email).
   - Genera mensaje personalizado con estado del pedido.
   - Envía a través de SendGrid (email) y Twilio (SMS) u otro proveedor.
   - Publica push notification en app móvil via Firebase Cloud Messaging.

3. Ejemplos de mensajes:
   - Orden creada: "Tu pedido PO-12345 fue confirmado. Total: $25.00".
   - Restaurante aceptó: "Tu pedido está siendo preparado. Tiempo estimado: 20 min".
   - Courier asignado: "Courier asignado, llega en ~12 min. Ver tracking".
   - Entrega completada: "Tu pedido fue entregado. ¡Gracias!".

4. El Notification Service registra cada intento de notificación (éxito, rechazo, bouncing email).

**Flujos alternativos**
- **FA-01**: SendGrid/Twilio no disponible.
  - Paso 2: El Notification Service encola la notificación para retry async.
  - Consumidor puede ver estado en app sin recibir SMS/email hasta que se recupere.
  
- **FA-02**: Consumidor ha optado por no recibir notificaciones.
  - Paso 2: El Notification Service respeta preferencias, solo registra en sistema.

**Postcondiciones**
- El consumidor ha recibido al menos una notificación (push, SMS, o email) para cada cambio de estado importante.
- Todas las notificaciones están correlacionadas con el PO y correlation ID.

**Given/When/Then**
- **Given**: Un consumidor con preferencias (push: sí, SMS: sí, email: no), pedido pasa a estado CONFIRMED_BY_RESTAURANT.
- **When**: El Notification Service recibe evento `RestaurantTicketAccepted`.
- **Then**: Se envía push notification y SMS al consumidor con mensaje: "Tu pedido está siendo preparado. ETA: 20 min".

Origen: [Brief §A.3] [Richardson Cap.3 - Event-driven architecture]

---

## 4. Relación con el PRD y capacidades

| UC | Capacidad PRD | NFR asociado | Origen |
|---|---|---|---|
| UC-01 | Order Taking | NFR-01 (Latencia), NFR-03 (Escalabilidad) | [US-01] |
| UC-02 | Order Fulfillment/Kitchen | NFR-02 (Disponibilidad), NFR-03 (Escalabilidad) | [US-02] |
| UC-03 | Delivery | NFR-03 (Escalabilidad), NFR-05 (Eventual consistency) | [US-03] |
| UC-04 | Billing & Accounting | NFR-04 (Tolerancia a fallos), NFR-05 (Eventual consistency) | [Brief §A.3] |
| UC-05 | Delivery | NFR-01 (Latencia), NFR-06 (Trazabilidad) | [Brief §A.5] |
| UC-06 | Notifications | NFR-06 (Trazabilidad) | [Brief §A.3] |

---

## 5. Referencias

- [PRD] — Product Requirements Document FTGO
- [Brief §A.1] — Contexto de negocio FTGO
- [Brief §A.2] — Stakeholders
- [Brief §A.3] — Capacidades de negocio
- [Brief §A.4] — Restricciones técnicas y NFRs
- [Brief §A.5] — User stories semilla
- [Richardson Cap.2] — Decomposition by Business Capability
- [Richardson Cap.3] — Interprocess Communication in a Microservice Architecture
- [Richardson Cap.4] — Managing Transactions with Sagas
- [Richardson Cap.5] — API Gateway, Service Discovery
- [US-01] — Toma de pedido por consumidor
- [US-02] — Aceptación de tickets por restaurante
- [US-03] — Asignación de entrega al courier
