# Mrs Curly Work

Siete skills para agenda y clientas, bandeja, finanzas, inventario, catálogo/equipo, prioridades diarias y operaciones entre módulos. El paquete conecta directamente a `https://polihair.vercel.app/api/mcp` mediante Streamable HTTP y OAuth individual.

No requiere Node, procesos locales, variables de entorno ni credenciales incluidas en el paquete. Cada skill incluye su contrato de operación y funciona sin leer archivos de otras skills. La autorización se completa en el login de Mrs Curly, nunca enviando contraseñas al chat.

## Instalar en ChatGPT y usar desde móvil

1. En ChatGPT web, con la misma cuenta y espacio de trabajo que en el móvil, activa Ajustes → Seguridad e inicio de sesión → Modo desarrollador, si la política de tu cuenta lo permite.
2. En Plugins → +, registra el nombre Mrs Curly y la URL `https://polihair.vercel.app/api/mcp`. Selecciona OAuth y completa el login y consentimiento.
3. Instala/habilita la conexión creada. Abre una conversación nueva de Work y prueba «Consulta las capacidades de Mrs Curly».
4. Para asociar estas siete skills al registro de ChatGPT, copia el identificador `plugin_asdk_app...` de la URL de la conexión. Ese ID real se utiliza para generar `.app.json`; no es una contraseña. La guía del repo incluye el comando de asociación.
5. Distribuye el paquete asociado mediante la opción de publicación de tu workspace si eres administrador y está habilitada. La publicación privada permanece dentro del workspace; el directorio público tiene un proceso separado de revisión y publicación.
6. En el móvil abre el mismo workspace, entra en Work y selecciona el plugin disponible para tu cuenta en una conversación nueva. Si solo está instalado desde un marketplace local del ordenador, no des por hecha su disponibilidad en móvil.

El ZIP es el paquete de distribución; adjuntarlo a un chat no equivale a instalarlo. Una conexión MCP creada solo con la URL permite usar sus herramientas, pero no incorpora por sí sola estas skills.

## Autorización persistente

ChatGPT gestiona la renovación OAuth. No es necesario introducir un token en cada petición. Una autorización caducada o revocada puede requerir reconectar. Puedes retirar acceso en [Agentes](https://polihair.vercel.app/oauth/connections). El alcance `polihair:operate` permite operar todos los módulos del salón único; las escrituras se limitan a lo pedido por la usuaria.

## Estado de esta entrega

Paquete preparado y validado. El servidor OAuth ya está desplegado. La instalación en la cuenta de ChatGPT y su disponibilidad en la app móvil todavía requieren verificación después de registrar y autorizar la conexión. No se ha publicado en el directorio público.

## Fuentes

- [Empaquetado, registro del MCP y publicación en workspace](https://developers.openai.com/plugins/build/plugins).
- [Conectar y probar](https://developers.openai.com/plugins/deploy/connect-chatgpt).
- [Disponibilidad de plugins en móvil](https://learn.chatgpt.com/docs/plugins).

La versión local anterior `mrs-curly` es independiente; esta entrega `mrs-curly-work` permite conservar su conexión de compatibilidad mientras se termina la instalación remota.
