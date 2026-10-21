---
name: mrs-curly-agenda
description: "Consultar y gestionar citas, disponibilidad, lista de espera y fichas de clientas de Mrs Curly mediante MCP."
---

# Agenda y clientas

Aplica [el contrato común](references/operacion.md).

- Busca por nombre u otro dato aportado con `clients_search`; resuelve coincidencias antes de usar `client_get`. Lee perfil capilar e historial cuando afecten al servicio solicitado.
- Para un día usa `calendar_get_day`; para un periodo `calendar_get_range`, cuyo máximo es 62 días inclusive. Amplía por intervalos si se pide un periodo mayor.
- Usa `services_list` o `service_get` para identificar variante y duración; consulta `availability_find` o `availability_search` con los IDs vigentes. Un hueco visual no demuestra disponibilidad: hay horarios, ausencias, bloqueos y dependencias.
- Reutiliza literalmente los IDs devueltos; si puedes conservar resultados en variables, extrae el ID de la variante elegida en vez de reescribir el UUID. Nunca reconstruyas un ID de memoria. Ante `VARIANT_UNAVAILABLE` (o el antiguo `NOT_FOUND` de variante), relee el servicio y repite una vez con su ID activo exacto. Si persiste, consulta a la usuaria. Ese error no significa «sin huecos».
- Antes de modificar una cita lee `booking_get`. Propón alternativas cuando se soliciten; ejecuta `booking_create`, `booking_reschedule`, `booking_cancel` o `booking_status_update` únicamente dentro de lo autorizado.
- Revisa `waitlist_list` cuando el objetivo incluya espera; una falta de huecos no autoriza crear una entrada por sí sola.

Completar el servicio o registrar su cobro presencial con `booking_checkout` aplica consumos de stock. No repitas el consumo manualmente. `booking_checkout` recibe `amountEuros`; el esquema de cada operación determina sus unidades.

Si el servidor anuncia garantías Stripe, lee `booking_payment_get`. `booking_guarantee_propose` genera la autorización que la clienta debe aceptar; un cambio de precio, fecha o condiciones puede exigir nueva aceptación. No confirmes una reserva pendiente de tarjeta ni aceptes condiciones en su nombre. Marcar `no_show` no cobra. La penalización se ejecuta aparte con `booking_penalty_charge`, solo con permiso y autorización explícita de la propietaria para esa cita e importe. No completa el servicio ni consume stock; actualmente solo funciona en sandbox.

Para reservas del portal usa `booking_portal_catalog`, `booking_portal_availability` y `booking_portal_submit`. Conserva preguntas y consentimientos reales: no marques consentimientos ni inventes respuestas para pasar la validación.

Si la reserva nace de un chat, incluye `conversationId` en `booking_create` para vincularla atómicamente a esa conversación y su clienta. Enviar la confirmación requiere autorización explícita de la usuaria para ese mensaje o lote; autorizar la reserva por sí solo no autoriza el envío. Un fallo de envío no requiere recrear la reserva.

## Convivencia con Treatwell

Mientras Treatwell Pro siga siendo la agenda responsable del salón, la disponibilidad real está en Treatwell, no en poliHair. Sigue [la guía de convivencia](references/treatwell-convivencia.md): comprueba los huecos en Treatwell con el script de lectura antes de ofrecer una hora, registra la cita primero en Treatwell y después refléjala en poliHair con `booking_create`. No respondas «hay hueco» usando solo `availability_find`. Si no dispones de control de navegador con la sesión de Treatwell, dilo y pide que se compruebe allí; no confirmes.

Para dos servicios seguidos, verifica ambos IDs, profesionales compatibles, duraciones y huecos consecutivos, incluidos los márgenes que aplique el motor. No sustituyas servicios por un pack distinto para esquivar un error de consulta. Las reservas individuales no constituyen una operación atómica conjunta: si solo se autoriza el conjunto, acuerda con la usuaria cómo gestionar una creación parcial antes de escribir; no canceles ni confirmes el conjunto por iniciativa propia. Informa «reservado» únicamente tras comprobar el éxito de todos los servicios.
