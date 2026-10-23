# Agente de Marketing n8n – Storedimo

Agente de marketing y ventas automatizado en n8n para vender **Storedimo** (sistema de inventario en la nube, Laravel).

## Arquitectura

```
                 ┌──────────────────────────────────────────┐
 Redes sociales  │ 02 Generador de contenido (cada lunes)   │──► Hoja "Contenido" (tú apruebas y publicas)
                 └──────────────────────────────────────────┘
        │ la gente escribe
        ▼
 ┌──────────────────────────────────────────┐
 │ 01 Agente de Ventas (chat con Claude)    │──► Hoja "Leads" (registrar_lead)
 │ web / WhatsApp / Telegram                │
 └──────────────────────────────────────────┘
        │ se registra en la prueba gratis
        ▼
 Laravel (Storedimo) ──webhook──► ┌──────────────────────────────────────┐
           ▲                      │ 03 Seguimiento Trial (día 0,3,10,14) │──► Gmail
           └──── ¿ya pagó? ───────└──────────────────────────────────────┘
```

| Workflow | Qué hace |
|---|---|
| `workflows/01_agente_ventas_chat.json` | Agente IA "Dimo": conversa, califica al prospecto, maneja objeciones, lo lleva a la prueba gratis y lo guarda en Google Sheets. |
| `workflows/02_generador_contenido_semanal.json` | Cada lunes genera 7 publicaciones (Facebook, Instagram, TikTok, LinkedIn) y las deja en una hoja para aprobarlas. |
| `workflows/03_seguimiento_trial.json` | Secuencia de correos durante los 15 días de prueba; deja de enviar si el cliente ya pagó. |
| `workflows/04_publicar_aprobados_fb_ig.json` | Todos los días crea la imagen (Bannerbear) y publica en Facebook e Instagram las filas `aprobado` del día. Guía: `GUIA_PUBLICACION_FB_IG.md`. |

## Instalación

1. **Google Sheet**: crea una hoja de cálculo con dos pestañas:
   - `Leads` con encabezados: `session_id, fecha, nombre, negocio, ciudad, correo, whatsapp, control_actual, interes, notas`
   - `Contenido` con encabezados: `dia, red, tipo, negocio_objetivo, gancho, texto, idea_visual, cta, hashtags, estado`
2. En n8n: **Workflows → Import from File** e importa los 3 JSON.
3. Crea las credenciales:
   - **Anthropic** (API key de https://console.anthropic.com) → nodos "Claude".
   - **Google Sheets OAuth2** → nodos `registrar_lead` y "Guardar en hoja Contenido".
   - **Gmail OAuth2** → nodos de correo del workflow 03.
   - **Header Auth** → ver `laravel/INTEGRACION_LARAVEL.md`.
4. Reemplaza `PEGA_AQUI_LA_URL_DE_TU_GOOGLE_SHEET` por la URL de tu hoja.
5. Aplica los cambios en Laravel de `laravel/INTEGRACION_LARAVEL.md`.
6. Prueba el workflow 01 con el botón **Chat** de n8n. Al activarlo, el Chat Trigger te da una URL pública que puedes enlazar o incrustar en softdimo.com.

## Siguientes pasos recomendados

- **WhatsApp**: en Colombia es el canal que más vende. Reemplaza el *Chat Trigger* por *WhatsApp Trigger* (WhatsApp Business Cloud API de Meta) y añade un nodo *WhatsApp → Send Message* con la respuesta del agente. Usa `{{ $json.messages[0].from }}` como `sessionKey` de la memoria.
- **Publicación automática**: cuando confíes en el contenido, agrega un workflow que lea las filas con `estado = aprobado` y publique con los nodos de Facebook Graph API / LinkedIn.
- **Prospección**: un workflow con Google Maps (Places API) buscando "ferreterías en Medellín", "minimercados en Cali", etc., para armar una lista de negocios y hacer contacto en frío (respetando la Ley 1581 de protección de datos y las políticas de WhatsApp: solo plantillas aprobadas y con opt-out).
- **Reporte semanal**: workflow que cuente leads por interés y trials convertidos y te lo envíe por Telegram o correo.

## Mejoras al producto que impactan las ventas

Lo encontré revisando la página pública de login/registro:

1. **Las descripciones de los planes son texto de relleno.** Ese es el primer lugar donde el cliente decide; define qué incluye cada plan (usuarios, productos, sucursales, soporte).
2. **Básico ($110.000) y Profesional ($121.000) casi cuestan lo mismo**, y Micro ($10.000) está muy lejos. Revisa la escalera de precios y aclara si son valores mensuales.
3. **No hay landing page de ventas**: el login es también la página de planes. Crea una landing con beneficios, capturas, video demo, testimonios y un botón de WhatsApp. El agente y los anuncios deben llevar ahí.
4. **El formulario de registro pide demasiados datos** (NIT, dígito de verificación, logo, fechas...). Para el Trial, pide solo nombre, negocio, correo y celular; el resto, después.
5. **No hay datos de contacto visibles** en la app.
