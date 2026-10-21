---
name: mrs-curly-operar
description: "Operar Mrs Curly mediante su MCP cuando una petición combina módulos, necesita lotes o requiere recuperar una operación interrumpida."
---

# Operar Mrs Curly

Lee [el contrato común](references/operacion.md) una vez en la sesión. Selecciona solo las skills y herramientas que necesita el resultado solicitado:

- Resumen y prioridades: `mrs-curly-dia`.
- Reservas, disponibilidad e historial de clientas: `mrs-curly-agenda`.
- Documentos, cobros y archivos: `mrs-curly-finanzas`.
- Productos, proveedores y movimientos: `mrs-curly-inventario`.
- Conversaciones y propuestas: `mrs-curly-bandeja`.
- Servicios, variantes, profesionales y horarios: `mrs-curly-catalogo-equipo`.

Para comprobar conexión, descubre herramientas, llama a `polihair_capabilities` con `input: {}` y realiza una lectura útil. Una prueba de conexión no autoriza crear fixtures ni cambiar datos.

En flujos entre módulos, reutiliza IDs devueltos y separa dependencias. Para una petición como «cobra la cita y revisa el stock», consulta la cita y su documento vinculado, ejecuta el cobro autorizado y comprueba el resultado; el cobro ya aplica el consumo de la variante, por lo que no debes descontarlo otra vez.

Si la usuaria pregunta por una operación interrumpida, recupera su clave antes de ejecutar nada nuevo. Usa `operation_get` y la ficha correspondiente para distinguir un resultado confirmado de uno incierto.
