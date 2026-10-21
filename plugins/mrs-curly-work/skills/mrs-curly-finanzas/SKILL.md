---
name: mrs-curly-finanzas
description: "Gestionar presupuestos, facturas, tesorería, registros de cobros y adjuntos de Mrs Curly mediante MCP."
---

# Finanzas

Aplica [el contrato común](references/operacion.md).

Empieza por `finance_get_summary` filtrado por clase, estado o búsqueda. Para detalle usa `financial_document_get`; para movimientos, `financial_movements_list`. Usa los totales del servidor y distingue importe del documento, saldo pendiente y dinero registrado. Una página de documentos no basta para sumar un periodo completo.

Antes de editar, cobrar o convertir un presupuesto, lee el documento y su estado. Reutiliza las relaciones devueltas por `financial_estimate_convert`; los anticipos se trasladan según las reglas del ERP y no se registran de nuevo. `financial_movement_register` anota un pago recibido externamente; no cobra una tarjeta.

Cuando el servidor anuncie pagos Stripe, usa `payments_list` para movimientos y pendientes, y `booking_payment_get` antes de gestionar una cita. `booking_penalty_charge` ejecuta un cargo en sandbox: requiere permiso de pagos y autorización explícita de la propietaria para esa cita e importe. Marcar `no_show` no autoriza cobrar. No aumentes el máximo aceptado ni cobres una cita completada o dispensada.

`booking_penalty_reduce` reduce el límite; con `amountCents: 0` perdona la penalización y registra el motivo. `payment_settings_save` cambia la política de nuevas garantías. `booking_guarantee_propose` prepara un enlace: la clienta debe aceptar personalmente las condiciones y preparar su tarjeta. No simules su consentimiento ni introduzcas tarjetas por ella.

Ante resultado incierto consulta `operation_get`, `booking_payment_get` y `penalty_charge_refresh`. Conserva el PaymentIntent y no abras otro cargo. Rechazo o autenticación pendiente no son ingresos. Obtén el enlace privado mediante lectura; no lo publiques ni lo envíes sin autorización. Una devolución o disputa modifica Tesorería, sin completar el servicio ni consumir stock. La integración actual está restringida a sandbox; no afirmes que está activada para dinero real.

Para PDF usa `financial_document_pdf`; las facturas de compra conservan el original adjunto. En lecturas por bloques, decodifica cada base64, concatena bytes y sigue `nextOffset` con `expectedSha256`. Comprueba tamaño y hash al terminar. Si devuelve una URL temporal, úsala como acceso privado, sin prometer permanencia.

`financial_attachment_upload` admite PDF/JPG/PNG/WebP hasta 4 MB y base64 sin prefijo data. El almacenamiento privado de producción puede responder por bloques autenticados. No descargues archivos cuando basta con consultar sus metadatos.

No borres facturas emitidas ni edites movimientos si esa operación no existe en el MCP. Explica la limitación en lugar de proponer cambios directos a la base de datos.
