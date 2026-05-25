# ADR-0001 — Estilo arquitectónico y estrategia de migración

**Status**: Accepted  
**Fecha**: 2026-05-24  
**Autor**: Arquitecto FTGO  
**Revisores**: Equipo de arquitectura, Líderes técnicos  

---

## 1. Contexto

FTGO opera actualmente como una **aplicación monolítica Java** que ha crecido exponencialmente, acumulando síntomas del "infierno monolítico": escalado conflictivo entre módulos, builds lentos (30-45 minutos), falta de aislamiento de fallos y baja mantenibilidad [Brief §A.1] [Richardson Cap.1]. La base de código es de ~500K LOC con dependencias circulares entre capacidades de negocio (Consumer, Restaurant, Order, Delivery, Billing, Notifications) que impiden que equipos independientes avancen a velocidad [Brief §A.1].

El negocio necesita **sostener crecimiento** mediante:
- Escalado independiente de capacidades durante tráfico pico de 5x [Brief §A.4 Carga]
- Latencia < 200 ms p95 en acciones del consumidor [Brief §A.4 Latencia UX]
- Tolerancia a fallos de sistemas externos (Stripe, Google Maps) [Brief §A.4 Tolerancia a fallos externos]
- Despliegues independientes sin coordinación global [Brief §A.1]

La **decisión arquitectónica** que debe tomarse ahora es: **¿cuál es el estilo arquitectónico objetivo y la estrategia de migración desde el monolito?**

Opciones:
1. **Mantener monolito** con refactorización interna.
2. **Reemplazo big-bang** hacia microservicios nuevos.
3. **Migración incremental con Strangler Fig** hacia microservicios.

---

## 2. Opciones consideradas

### Opción 1: Mantener y refactorizar el monolito

**Descripción**  
Revertir al estado previo: aplicar principios de modularidad dentro del monolito (DDD estratégico, bounded contexts internos), mejorar tests, refactorizar dependencias circulares sin cambiar el despliegue monolítico. El equipo mantiene una única unidad de despliegue.

**Pros**
- Menor complejidad operacional inmediata.
- No hay overhead de IPC (Inter-Process Communication) de red.
- Transacciones ACID nativas en la BD monolítica.
- Equipo mantiene el mismo entorno de ejecución.
- Costo inicial más bajo.

**Contras**
- **No soluciona escalabilidad horizontal**: la Entrega no puede crecer independientemente de Billing. [Brief §A.4 Escalabilidad horizontal]
- **Builds siguen siendo lentos**: una refactorización interna no reduce el acoplamiento de compilación.
- **Aislamiento de fallos insuficiente**: una fuga de memoria en Notifications tira todo el sistema. [Brief §A.4 Tolerancia a fallos externos]
- **Lock-in tecnológico**: Java/Spring Boot sigue siendo obligatorio en todas las capacidades, imposibilitando freedom of technology en equipos satélite.
- **No resuelve el problema raíz**: el monolito sigue siendo un cuello de botella para equipos paralelos [Richardson Cap.1].
- **Eventual obsolescencia**: a medida que crece, los problemas monolíticos reaparecen.

**Impacto en NFRs**
- Escalabilidad horizontal ✗ (no se puede escalar Delivery sin escalar Billing)
- Latencia UX ⚠ (sin IPC de red, pero compilación lenta impacta ciclo desarrollo)
- Tolerancia a fallos ✗ (un componente puede tirar todo el sistema)
- Mantenibilidad ✗ (refactorización interna no resuelve complejidad acumulada)

**Trazabilidad**
- [Brief §A.1 Contexto de negocio] — necesidad de escalado independiente.
- [Brief §A.4 Escalabilidad horizontal] — X-axis y Y-axis del Scale Cube.
- [Richardson Cap.1] — Monolithic Hell y síntomas.

---

### Opción 2: Reemplazo big-bang hacia microservicios

**Descripción**  
Reescribir el sistema completo desde cero en una arquitectura de microservicios. Cada capacidad (Consumer, Restaurant, Order, Delivery, Billing, Notifications) se implementa como servicio independiente con su propia BD. El monolito legacy se desactiva completamente en un momento puntual ("cutover").

**Pros**
- **Escalabilidad horizontal completa**: cada servicio se escala independientemente en X-axis y Y-axis [Brief §A.4].
- **Libertad tecnológica**: Delivery Service puede ser Node.js, Notifications Service Python, etc.
- **Despliegues desacoplados**: cambios en Billing no requieren redeployment de Order Service.
- **Aislamiento de fallos**: Stripe caído no afecta a Order Taking (solo a pagos retry async).
- **DDD estratégico nativo**: cada servicio es un Bounded Context independiente.

**Contras**
- **Riesgo extremadamente alto**: reescritura completa significa duplicar todos los features del monolito.
- **Tiempo a mercado largo**: 18-36 meses típicamente antes del primer release productivo.
- **Problema de sincronización de datos**: cómo migrar datos de 7 capacidades sin corrupción. [Brief §A.4 Migración incremental]
- **Testing paralizado**: no hay sistema viable durante la reescritura, imposible iterar con clientes.
- **Equipo paralizado**: developers actuales siguen manteniendo el monolito + escribiendo nuevos servicios simultáneamente.
- **Decisiones arquitectónicas tomadas bajo incertidumbre**: sin pilotos reales, el diseño de IPC, consistencia y sagas es especulativo.
- **Gran inversión sin ROI a corto plazo**: FTGO no genera ingresos nuevos en meses/años.

**Impacto en NFRs**
- Escalabilidad horizontal ✓ (lograble, pero a alto costo)
- Latencia UX ⚠ (mejor potencial, pero riesgo de IPC ineficiente sin experiencia)
- Tolerancia a fallos ✓ (lograble con diseño correcto de circuit breakers)
- Trazabilidad ⚠ (observabilidad distribuida requiere inversión en APM)
- Riesgo operacional ✗ (sistema no productivo hasta el final)

**Trazabilidad**
- [Brief §A.1] — "no hay big-bang rewrite", migración incremental.
- [Brief §A.4 Migración incremental] — "Strangler Fig durante 18-24 meses".
- [Brief §A.6 Restricciones] — "Migración incremental como contexto".
- [Richardson Cap.2] — Opciones de descomposición, sin mencionar big-bang como recomendado.

---

### Opción 3: Migración incremental con Strangler Fig

**Descripción**  
El sistema objetivo es una arquitectura de **microservicios por capacidad de negocio** (Consumer, Restaurant, Order, Delivery, Billing, Notifications) [Brief §A.3]. La migración sigue el patrón **Strangler Fig** [Richardson Cap.2]: se crean servicios nuevos en paralelo con el monolito legacy, se redirige tráfico gradualmente, el monolito "se va extinguiendo" lentamente sin big-bang.

**Estrategia**:
1. Semana 1-2: Diseñar y levantar infraestructura base (API Gateway, Kafka, observabilidad).
2. Semana 3-8: Implementar primer servicio (Consumer Management) con réplica de datos del monolito.
3. Semana 9+: Redirigir tráfico de Consumer al nuevo servicio, validar en producción.
4. Semana 20+: Consumer completamente migrado, monolito se reduce a 6 capacidades.
5. Iteración: Siguiente servicio (Restaurant Management), luego Order, Delivery, Billing, Notifications.
6. Mes 18-24: Monolito legacy contiene solo lógica heredada, nuevas features en servicios.

**Pros**
- **Riesgo minimizado**: cada servicio se valida en producción antes de migrar el siguiente [Brief §A.6 Strangler Fig].
- **ROI incremental**: usuario final ve mejoras desde primer servicio migrado (latencia, escalabilidad en esa capacidad).
- **Equipo productivo**: una parte del equipo trabajar en nuevo servicio, otra mantiene monolito.
- **Toma de decisiones informada**: cada migración enseña lecciones que mejoran la siguiente.
- **Posibilidad de rollback**: si algo falla, redirigir tráfico de vuelta al monolito.
- **Coexistencia temporal manejable**: monolito y servicios nuevos coexisten, se comunican vía eventos/API.
- **Alineado con restricción de negocio**: Brief §A.4 menciona explícitamente Strangler Fig 18-24 meses.

**Contras**
- **Complejidad operacional inicial**: API Gateway, event bus, observabilidad distribuida requieren inversión.
- **Sincronización de datos**: durante coexistencia monolito-servicios, replicar datos puede ser complejo (dual writes, eventual consistency) [Brief §A.4 Consistencia].
- **Testing end-to-end más difícil**: flujos que cruzan monolito y servicios requieren coordinación.
- **Deuda técnica temporal**: el monolito sigue vivo, acumulando cambios heredados mientras migramos.
- **Latencia de red**: IPC entre servicios introduce latencia, debe compensarse con caching y optimización.

**Impacto en NFRs**
- Escalabilidad horizontal ✓ (cada nuevo servicio es independiente)
- Latencia UX ✓ (con IPC optimizado y caching, < 200 ms p95 es alcanzable)
- Tolerancia a fallos ✓ (servicios aislados, circuit breakers en integraciones externas)
- Consistencia ✓ (eventual consistency entre servicios, fuerte dentro de aggregates)
- Trazabilidad ✓ (correlation IDs y distributed tracing desde inicio)
- Migración incremental ✓ (Strangler Fig nativo)

**Trazabilidad**
- [Brief §A.1] — "equipos bloqueados, necesidad de escalado independiente".
- [Brief §A.3] — "7 capacidades de negocio estables".
- [Brief §A.4 Migración incremental] — "Strangler Fig 18-24 meses".
- [Brief §A.6] — "Migración incremental como contexto", "coexistencia temporal con monolito legacy".
- [Richardson Cap.2] — "Descomposición por capacidades de negocio", "Strangler Fig pattern".

---

## 3. Decisión

**Se selecciona: Opción 3 — Migración incremental con Strangler Fig hacia microservicios por capacidad.**

**Justificación**

La Opción 1 (mantener monolito) **no resuelve el problema raíz**: FTGO necesita escalabilidad horizontal independiente [Brief §A.4 Escalabilidad horizontal], aislamiento de fallos [Brief §A.4 Tolerancia a fallos externos] y velocidad de desarrollo [Brief §A.1]. Refactorización interna del monolito mantiene el acoplamiento de compilación y runtime que impide que equipos avancen en paralelo.

La Opción 2 (big-bang rewrite) **incumple la restricción explícita del brief** [Brief §A.4 Migración incremental]: "Strangler Fig durante 18-24 meses, no hay big-bang". Además, el riesgo es inaceptable: reescribir 500K LOC en paralelo mantener un monolito productivo típicamente fracasa [Richardson Cap.1]. FTGO no puede permitirse 18+ meses sin poder iterar con usuarios reales.

La Opción 3 (Strangler Fig) **cumple con restricciones explícitas** [Brief §A.4], **minimiza riesgo** mediante validación incremental, **habilita ROI temprano** (primer servicio en 2-3 meses), y **es probada en industria** [Richardson Cap.2]. Cada servicio migrado enseña lecciones arquitectónicas (IPC, consistencia, observabilidad) que mejoran las migraciones posteriores.

**Decisión técnica**:

El **sistema objetivo** es una arquitectura de **microservicios por capacidad de negocio**:
- Consumer Management Service (autenticación, perfiles, direcciones)
- Restaurant Management Service (catálogos, menús, disponibilidad)
- Order Taking Service (validación, cálculo de totales)
- Kitchen Service (tickets, estado de preparación)
- Delivery Service (asignación, tracking)
- Billing Service (procesamiento de pagos, comisiones)
- Notification Service (emails, SMS, push)

Cada servicio:
- Tiene su propia BD (database-per-service) [Brief §A.4 Escalabilidad horizontal].
- Escala independientemente en X-axis y Y-axis [Brief §A.4 Carga].
- Se comunica vía **eventos async** preferentemente, con REST síncrono para queries [Brief §A.4 Tolerancia a fallos externos].
- Implementa circuit breakers para integraciones externas (Stripe, Google Maps).
- Propaga correlation IDs para trazabilidad end-to-end [Brief §A.4 Trazabilidad].

---

## 4. Consecuencias

### Consecuencias positivas

1. **Escalabilidad horizontal independiente realizada**  
   Delivery Service escala a 100 instancias durante horarios pico (12:00-14:00) sin afectar a Billing Service. [Brief §A.4 Carga, Escalabilidad horizontal]

2. **Aislamiento de fallos mejorado**  
   Si Stripe está caído, Order Taking sigue aceptando pedidos con pagos en cola de retry. [Brief §A.4 Tolerancia a fallos externos]

3. **Velocidad de desarrollo mejorada**  
   Equipos de Consumer y Restaurant trabajan en paralelo sin ciclos de compilación global. Despliegues independientes reducen ciclo de entrega de semanas a horas.

4. **Riesgo minimizado**  
   Cada servicio se valida en producción antes de migrar el siguiente. Rollback es posible redirigiendo tráfico al monolito.

5. **Libertad tecnológica**  
   Billing Service puede ser Java/Spring Boot, Notification Service puede ser Node.js o Python. [Brief §A.4 Tecnología "libertad en servicios satélite"].

6. **DDD estratégico nativo**  
   Cada servicio es un Bounded Context independiente. Lenguaje ubicuo (ubiquitous language) clara por dominio.

7. **Costo y ROI gradual**  
   Inversión inicial en infraestructura (API Gateway, Kafka, APM), pero ROI desde primera migración en 2-3 meses.

### Consecuencias negativas

1. **Complejidad operacional inicial**  
   Se requiere:
   - API Gateway (p.ej., Kong o Spring Cloud Gateway)
   - Event bus (Kafka, RabbitMQ)
   - Distributed tracing (Jaeger, Datadog)
   - Service mesh opcional (Istio) para resiliencia
   - Orquestación de containers (Kubernetes) recomendado
   
   Inversión 3-6 meses en infraestructura antes del primer servicio en producción.

2. **Sincronización de datos compleja**  
   Durante coexistencia monolito-servicios, se requiere:
   - Dual writes (escritura simultánea en BD monolito y servicio nuevo)
   - O change data capture (CDC) desde monolito a servicios
   - O event sourcing en el monolito
   
   Risk: inconsistencias temporales [Brief §A.4 Consistencia]. Requiere testing exhaustivo.

3. **Latencia de red introducida**  
   IPC entre servicios es más lento que llamadas intraproceso:
   - Monolito: 1-5 ms (llamada a función)
   - Microservicios: 10-100 ms (HTTP/gRPC + red)
   
   Mitigation: caching agresiva, prefetching, optimización de payloads. [Brief §A.4 Latencia UX ≤ 200 ms p95]

4. **Testing end-to-end más difícil**  
   Flujos que cruzan monolito y servicios requieren:
   - Orquestación de múltiples contenedores
   - Stubs/mocks de servicios nuevos en ambiente monolito
   - Monitoreo de múltiples logs distribuidos
   
   Inversión: herramientas de testing distribuido (testcontainers, chaos engineering).

5. **Deuda técnica temporal**  
   El monolito sigue vivo 18+ meses, acumulando cambios heredados que deben aplicarse también en servicios nuevos (duplicación).

6. **Consistencia eventual implica reconciliación**  
   Entre servicios no hay transacciones ACID. Se requiere:
   - Sagas distribuidas para orquestación
   - Compensating transactions para rollbacks
   - Reconciliación periódica de datos
   
   [Brief §A.4 Consistencia] — "eventual consistency aceptada para reporting, fuerte dentro del aggregate de un pedido".

7. **Observabilidad distribuida obligatoria**  
   Sin correlation IDs y distributed tracing, debugging en producción es pesadilla. Inversión en APM es no negociable.

---

## 5. Decisiones relacionadas y follow-ups

Este ADR establece que FTGO migrará hacia **microservicios por capacidad**. Las decisiones relacionadas que deben tomarse son:

1. **ADR-0002: Comunicación entre servicios e integración async**  
   - Evaluar REST síncrono vs mensajería async.
   - Decidir sobre Kafka vs RabbitMQ vs AWS SQS.
   - Definir patrón de IPC (direct HTTP, message broker, event sourcing).

2. **ADR-0003: Estrategia de datos y consistencia**  
   - Database-per-service vs shared database.
   - Evento sourcing vs snapshot.
   - Sagas distribuidas vs compensating transactions.

3. **ADR-0004: API Gateway y orquestación**  
   - Seleccionar API Gateway (Kong, Spring Cloud Gateway, Ambassador, AWS API Gateway).
   - Definir punto de entrada único vs federated APIs.

4. **ADR-0005: Observabilidad y APM**  
   - Distributetracing (Jaeger, DataDog, New Relic, Prometheus).
   - Logs centralizados (ELK, Splunk, CloudWatch).
   - Métricas y alertas.

5. **Validaciones técnicas**:
   - Prototipo de IPC (latencia medida < 200 ms p95 con payload realista).
   - Prototipo de sincronización de datos (Consumer Service con dual writes).
   - Test de escalabilidad de primer servicio migrado.

6. **POCs recomendados**:
   - Semana 1-4: Levantar API Gateway + Kafka + Jaeger locales.
   - Semana 5-8: Implementar Consumer Management Service con BD separada.
   - Semana 9-12: Testing exhaustivo de interop monolito ↔ nuevo servicio.
   - Semana 13-16: Desplegar en ambiente staging, validar SLAs.
   - Semana 17+: Pilot production con 5-10% de tráfico en new Consumer Service.

---

## 6. Referencias

- [Brief §A.1] — Contexto de negocio, síntomas monolítico.
- [Brief §A.3] — 7 capacidades de negocio.
- [Brief §A.4] — Restricciones técnicas: escalabilidad, latencia, tolerancia a fallos, migración incremental.
- [Brief §A.6] — "Strangler Fig durante 18-24 meses", "no hay big-bang rewrite".
- [Richardson Cap.1] — Monolithic Hell, síntomas y soluciones.
- [Richardson Cap.2] — Decomposition by Business Capability, Strangler Fig pattern.
- [PRD] — Product Requirements Document FTGO.
- [FSD UC-01 a UC-06] — Functional Specification Document.
