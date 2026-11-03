---
name: mrs-curly-catalogo-equipo
description: "Consultar y actualizar servicios, variantes, precios, preguntas, profesionales, horarios y ausencias de Mrs Curly."
---

# Catálogo y equipo

Aplica [el contrato común](references/operacion.md). Usa `services_list` y `team_get_summary` para búsquedas generales, y `service_get` o `team_get` para configuración completa. `categories_list` permite resolver categorías antes de crearlas.

Una actualización sustituye la configuración editable: lee primero y conserva variantes, preguntas, requisitos, profesionales y consumos que no deban cambiar. No reconstruyas una ficha completa a partir de su tarjeta de resumen.

`service_update` crea nuevas versiones de variantes y preguntas. Reutiliza los nuevos IDs devueltos para reservas posteriores; las citas históricas conservan sus variantes anteriores. No reutilices un ID antiguo por estar en el contexto de una llamada previa.

Cada variante puede dividirse en `steps` ordenados (`name`, `durationMinutes`, `occupiesProfessional`). La duración es la suma de los pasos. Un paso que no ocupa a la profesional, como la exposición de un tinte, deja ese tiempo libre para otra cita suya; al menos un paso debe ocuparla. Al actualizar, conserva los pasos que devuelve `service_get` si no deben cambiar.

Para profesionales usa `professional_save` tras consultar la ficha y resolver servicios asignados. Distingue horarios habituales, `professional_absence_create` y `calendar_block_create`. Una ausencia no autoriza reagendar automáticamente citas afectadas: informa de ellas o ejecuta la reprogramación cuando forme parte del encargo.
