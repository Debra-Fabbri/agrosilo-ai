# 🌾 AgroSilo AI

Agente de operaciones agroindustriales para monitoreo y análisis operativo de sistemas de almacenamiento mediante silo-bolsas.

### Checkpoint 4: Integraciones Avanzadas e Interconexión de Sistemas
Sobre este archivo

Este workflow es la evolución directa del proyecto integrador que viene creciendo módulo a módulo (M1 → M2 → M3 → M4). En este mismo lienzo conviven dos entradas independientes al sistema, no por error de armado sino por diseño:

1. Brazo de Chat (heredado de M2/M3): *Trigger - Petición del cliente → Sheets Leer Memoria → AI Agent - Agente Central → ...*
2. Brazo de Email (nuevo en M4): *Gmail Trigger → IF anti auto-reply → AI Agent - Clasifica y Redacta → ...*

Ambos brazos comparten las mismas herramientas **(Tool - Worker 1 (Silo Registry Query) y (Tool - Worker 2 — Silo Analysis), ambas como sub-workflows independientes)** y el mismo criterio de negocio: si un cliente pregunta por un silo puntual —sea por chat o por mail—, el agente correspondiente puede consultar el registro real y, si hace falta, encadenar un análisis operativo, en lugar de responder con datos genéricos o inventados.

Esta reutilización no es una casualidad; refleja que es un mismo sistema con múltiples puntos de entrada, no dos proyectos separados pegoteados en un archivo.

---------

### Qué resuelve este checkpoint (M4)

El objetivo de este hito es conectar el agente con herramientas reales de negocio (CRM, casilla de correo, canal de equipo) vía OAuth2, y blindar esa interconexión contra vulnerabilidades operativas clásicas. Los 4 nodos que la rúbrica evalúa, con su rol específico dentro de la arquitectura de AgroSilo AI:

① IF anti auto-reply (① IF · ¿Es Auto-Reply / Out of Office?) — corta el bucle infinito de respuestas automáticas. Escanea el asunto y el remitente del correo entrante contra patrones de auto-respuesta (auto-reply, out of office, undeliverable, no-reply@) antes de que el agente procese nada. Si detecta un patrón automático, corta la ejecución en un nodo Stop, sin generar respuesta ni notificación.

② Look up antes del Create (② Look up · ¿Contacto existe en Airtable?) — evita el Error 409 (duplicados). Antes de crear un contacto nuevo en el CRM (Airtable), busca si ya existe un registro con ese email. Si existe, actualiza (Update Record); si no, recién ahí crea (Create Record). Este es el mismo patrón de "buscar antes de escribir" que ya se usa en el Registro de Silos desde el Módulo 2.

③ Create Draft (③ Create Draft · Borrador para Aprobación) — guardrail Human-in-the-loop. El agente redacta la respuesta al cliente, pero nunca la envía de forma autónoma: la deja como borrador en Gmail, con destinatario, asunto y cuerpo ya completos, esperando revisión y envío manual de un humano. Este guardrail es coherente con el principio de diseño de AgroSilo AI: el sistema analiza y recomienda, pero no ejecuta acciones críticas ni irreversibles sin supervisión.

④ Set de limpieza de payload — deja campos limpios y validados, evitando el Error 400 (payload mal formado). Se aplica en dos puntos del flujo: antes de escribir en el CRM (dejando solo from, subject, body_text, nombre_contacto, con fallback si el remitente viniera vacío) y antes de notificar a Slack (un resumen de una línea, sin volcar el cuerpo completo del correo ni metadatos técnicos pesados al canal del equipo).

---------

### Conectores utilizados (OAuth2)
Conector - Rol - Alcance configurado

- Gmail	Casilla de soporte / alertas operativas	Lectura de inbox + creación de borradores (sin permiso de envío autónomo)

- Airtable	CRM — registro de contactos/clientes	Lectura y escritura acotada a la base contactos_clientes

- Slack	Canal del equipo de operaciones	Envío de mensajes al canal designado

---------

### Cómo importar y probar
1. Abrí n8n → Workflows → Import from File → seleccioná checkpoint4_nombre_apellido.json.
1. Configurá tus propias credenciales OAuth2 para Gmail, Airtable y Slack (las del archivo original no viajan con el JSON por seguridad).
1. Actualizá las referencias de Base/Table de Airtable y el Channel de Slack a tus propios recursos.
1. Los sub-workflows **Worker 1 · Silo Registry Query y Worker 2 · Silo Analysis** deben existir e importarse (o ya estar activos) por separado — son los mismos del Módulo 2/3, ***NO vienen embebidos en este archivo.***
1. Publicá y activá cada sub-workflow (Worker 1 y 2) antes de probar — en instancias de n8n con control de versiones, además de guardar hace falta un Publish explícito para que el workflow quede activo en producción.
1. Para probar el brazo de email: mandate un correo real a la casilla conectada mencionando un código de silo (ej: "¿Cómo está el SILO-04?") y confirmá que se genere un borrador de respuesta en Gmail y una notificación en el canal de Slack.
1. Para probar el brazo de chat: usá el panel de chat de n8n con el **Trigger - Petición del cliente**, igual que en M2/M3.

---------

### Notas de diseño
- El circuito de memoria de largo plazo (Memoria_Sesiones, Google Sheets) y el registro de silos (Silo Registry) vienen de módulos anteriores y no se modificaron en este checkpoint.
- El principio de mínimo privilegio se aplicó acotando los scopes de OAuth2 al mínimo necesario para cada función (lectura de inbox + borradores en Gmail, sin acceso de envío o eliminación; base específica en Airtable, no acceso a todo el workspace; canal puntual en Slack, no administración).

----------
