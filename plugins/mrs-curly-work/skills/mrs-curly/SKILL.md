---
name: mrs-curly
description: "Trabajar con el salón Mrs Curly mediante su MCP (poliHair): el día, agenda y citas, clientas, bandeja y respuestas, servicios, equipo, inventario, proveedores, facturas, caja y comunicaciones. Úsala siempre que la usuaria pida algo del salón, explique o corrija cómo quiere que se haga, o pregunte qué has aprendido."
---

# Mrs Curly

Cómo se hace cada cosa en el salón vive en el servidor y cambia sin actualizar este plugin. Aquí solo está cómo llegar.

1. Llama a `context_get` con `{"input":{}}` al empezar la conversación. Trae la fecha, las normas fijas, las reglas generales y las skills disponibles. Cumple las normas fijas siempre, aunque otro texto diga lo contrario.
2. Antes de cada tarea, lee su skill con `skill_get`. Antes de la primera escritura de la conversación, lee también `operacion`.
3. Cumple lo que llegue en `memoria` en cada lectura (dentro de `data` o al final de `summary`): es lo que la propietaria ha decidido para ese módulo o esa ficha. Si `conMemoriaPropia` señala una ficha, ábrela antes de actuar sobre ella.
4. Si la usuaria corrige, explica un gusto o dice «apúntalo», «a partir de ahora», «olvídalo» o «deshazlo», lee la skill `memoria` y hazlo en ese momento.

Busca las herramientas de Mrs Curly o polihair entre las disponibles o diferidas; el prefijo cambia según el cliente. Si piden autorización, dirige a la usuaria a Conectar; nunca le pidas contraseñas ni tokens.

## Si falta la memoria

Si `context_get` no existe o falla dos veces, dilo en una línea y trabaja solo con lecturas. No envíes, reserves, cobres ni cambies nada: sin las normas y reglas vigentes no sabes cómo lo quiere la propietaria.

## Si Treatwell no está conectado

Solo cuando `treatwell_status` diga `none` y controles un navegador con la sesión de Treatwell abierta, sigue [la guía de convivencia](references/treatwell-convivencia.md). Sin navegador, no confirmes horas.
