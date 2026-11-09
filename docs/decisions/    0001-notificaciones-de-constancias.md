---
status: "proposed"
date: 2026-11-09
---

# Desacoplamiento del envío de notificaciones de constancias digitales mediante procesamiento asíncrono

## Context and Problem Statement

El sistema ConstanciaUV debe generar y almacenar constancias digitales para los estudiantes y notificarles por correo electrónico. Sin embargo, el servicio de correo electrónico es un proveedor externo no confiable (RT1) que puede experimentar caídas o latencia. Se necesita definir una arquitectura de comunicación entre el módulo de generación de constancias y el servicio de notificaciones que garantice la continuidad operativa del sistema de constancias (RT2), conserve las notificaciones para reintentarlas (RT3) y prevenga el envío de solicitudes duplicadas (RT4).

## Decision Drivers

- **Desacoplamiento y Alta Disponibilidad del servicio core (RT2):** La generación y almacenamiento de la constancia digital no debe bloquearse ni fallar si el proveedor externo de correo no está disponible.
- **Fiabilidad en la entrega e Idempotencia (RT3, RT4):** El sistema debe garantizar que todas las notificaciones se procesen posteriormente sin perder información y sin generar envíos duplicados al estudiante.

## Considered Options

- **Alternativa A:** Comunicación directa con recuperación de fallos (Sincrónico/Inmediato con Fallback).
- **Alternativa B:** Procesamiento asíncrono mediante eventos o registros persistidos (Desacoplado por Cola/Tabla de Pendientes).

## Decision Outcome

Chosen option: **"Alternativa B. Procesamiento asíncrono mediante eventos persistidos"**, porque desacopla completamente el flujo crítico de generación de constancias del estado del servicio externo. Al persistir el evento/registro de notificación dentro de la misma transacción o inmediatamente después de generar la constancia, se satisface RT2 por diseño. Un componente independiente (worker/scheduler) procesa la cola controlando el estado de envío, reintentos y bloqueos de deduplicación, cumpliendo de forma robusta con RT3 y RT4.

### Consequences

- **Good, because** la disponibilidad de la generación de constancias pasa a ser del 100% independiente del estado del proveedor de correo (cumple RT2).
- **Good, because** simplifica la gestión de idempotencia y reintentos en un único componente especializado sin recargar la lógica principal de la aplicación.
- **Bad, because** introduce una leve latencia eventual (el correo no se envía de forma estrictamente instantánea en la misma petición HTTP, aunque se procesa en segundos).
- **Bad, because** incrementa ligeramente la complejidad arquitectónica al requerir un componente worker/background job y almacenamiento persistente de estados de notificación.

### Confirmation

- **Resiliencia ante caídas (RT2/RT3):** Se ejecutará una prueba de integración donde se desactive o simule una falla (mock 503/timeout) en el proveedor externo de correo. Se verificará que el endpoint de solicitud de constancia responda exitosamente (HTTP 200/201), la constancia quede almacenada y el registro de la notificación permanezca con estado `PENDING`. Al restaurar el servicio externo, el worker debe procesar la notificación exitosamente.
- **Prevención de duplicados (RT4):** Se realizará una prueba de concurrencia enviando múltiples solicitudes simultáneas de procesamiento para la misma constancia. Se validará mediante registros de auditoría y restricciones de base de datos (`UNIQUE constraint` sobre `constancia_id` o patrón Outbox/Idempotency Key) que el proveedor externo reciba únicamente una llamada API de envío por constancia.

## Pros and Cons of the Options

### Alternativa A. Comunicación directa con recuperación de fallos

Después de almacenar una constancia, la aplicación intenta solicitar directamente el envío al servicio de correo. Si falla, captura la excepción y registra la notificación como pendiente.

- **Good, because** en escenarios donde el proveedor externo está en línea, la notificación se envía de manera casi instantánea sin intermediación de componentes extra.
- **Bad, because** expone la transacción o petición principal del usuario a latencias o timeouts del proveedor de correo antes de poder registrar el fallo.
- **Bad, because** acopla temporalmente el proceso de solicitud de constancias a la infraestructura de notificaciones, complicando el control fino de duplicados en escenarios de peticiones concurrentes o reintentos inmediatos.

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

La aplicación únicamente registra la notificación pendiente en un almacenamiento persistente (ej. patrón Outbox o cola de mensajes persistente) y retorna inmediatamente la respuesta exitosa. Un worker o proceso independiente consume y procesa los envíos.

- **Good, because** garantiza el aislamiento total de la generación de constancias frente a fallas del servicio de correo (RT2).
- **Good, because** facilita la implementación de políticas de reintento (`exponential backoff`) y control estricto de idempotencia/duplicados mediante cerrojos o estados explícitos (`PENDING`, `PROCESSING`, `SENT`, `FAILED`) antes de invocar al proveedor (RT3, RT4).
- **Bad, because** requiere monitoreo adicional para asegurar que el componente procesador independiente esté activo y consumiendo las notificaciones en tiempo y forma.

## More Information

- **Revisión Parte 4:** La presente propuesta formaliza el análisis realizado por el equipo de arquitectura de ConstanciaUV para resolver el desacoplamiento de servicios de terceros.
- **Mecanismo de deduplicación recomendado:** Se propone incluir una clave única idempotente (`constancia_id` + `tipo_notificacion`) en la tabla o cola de persistencia de notificaciones para evitar registrar o procesar doblemente el aviso correspondiente a una misma constancia.