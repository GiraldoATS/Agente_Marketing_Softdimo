# Guía – Workflow 04: Publicar aprobados en Facebook e Instagram

## Cómo funciona

```
Todos los días 9am (hora Colombia)
  → Lee la hoja "Contenido" (solo filas con estado = aprobado)
  → Se queda con las de HOY cuya red sea facebook o instagram
  → Bannerbear crea la imagen con tu plantilla (pone el "gancho" como texto)
  → Publica en Facebook o en Instagram (con texto + cta + hashtags)
  → Cambia la fila a estado = publicado y guarda la URL de la imagen
```

Las filas de **tiktok** y **linkedin** no se tocan; esas las publicas tú.

Tu rutina semanal queda así: el lunes revisas la hoja, corriges lo que quieras y cambias a `aprobado` las publicaciones que te gusten. El resto lo hace n8n.

---

## Paso 1 – Ajustar la hoja "Contenido"

1. Agrega una columna nueva al final llamada **`imagen_url`**.
2. Selecciona la columna **`dia`** → menú **Formato → Número → Texto sin formato**, para que Google no cambie las fechas de formato.
3. Para aprobar una publicación, escribe exactamente `aprobado` (en minúscula) en la columna `estado`.

## Paso 2 – Bannerbear (crea las imágenes)

En la carpeta `plantillas/` ya están listos 3 fondos con tu logo y tus colores. El flujo elige uno según la columna `tipo`:

| Fondo | Se usa cuando `tipo` es | Color del texto |
|---|---|---|
| `fondo_dolor.png` | dolor | Blanco `#FFFFFF` |
| `fondo_tip.png` | educativa, demostración (y cualquier otro) | Azul oscuro `#0B2545` |
| `fondo_oferta.png` | oferta, prueba social | Blanco `#FFFFFF` |

Los archivos `vista_previa_*.png` muestran cómo se ve cada una ya con texto.

1. Crea una cuenta en https://www.bannerbear.com (la prueba gratis incluye 30 imágenes).
2. Repite esto **3 veces**, una por cada fondo:
   1. **Create Template → Blank**, tamaño **1080 x 1080**.
   2. Sube el fondo (`fondo_dolor.png`, etc.) como **Image** y estíralo para que ocupe todo el lienzo (x: 0, y: 0, 1080 x 1080). Bloquea la capa.
   3. Agrega una capa de **Text** y ponle de nombre exactamente **`gancho`**.
   4. Ubícala en **x: 90, y: 300, ancho: 900, alto: 420** (es el espacio vacío del fondo).
   5. Fuente **Montserrat**, peso **ExtraBold (800)**, tamaño **68**, alineación izquierda y vertical al centro, con el color de la tabla.
   6. Activa **Auto-fit / Shrink to fit**, para que los textos largos se achiquen y no se salgan.
   7. Guarda y copia el **ID del template** (Template → API Console → `uid`).
3. Copia tu **API Key** (Settings → API Key).

> Para cambiar los diseños (colores, frases fijas, logo en alta resolución), edita los HTML de `plantillas/fuente/` y vuelve a exportarlos. También puedes rehacerlos en Canva con las mismas medidas.

## Paso 3 – Meta (Facebook + Instagram)

Requisitos:
- Una **Página de Facebook** de Storedimo.
- Una **cuenta de Instagram Profesional (Empresa)**, vinculada a esa página (Instagram → Configuración → Cuenta → Vincular a página de Facebook).

Obtener el token:
1. Entra a https://developers.facebook.com → **Mis apps → Crear app** → tipo **Empresa**.
2. Abre el **Explorador de la API Graph** (Herramientas → Graph API Explorer).
3. Selecciona tu app y en **Permisos** agrega:
   `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `instagram_basic`, `instagram_content_publish`, `business_management`
4. Clic en **Generate Access Token** y acepta.
5. Consulta `me/accounts`. En la respuesta busca tu página y copia:
   - `id` → es tu **ID de página de Facebook**
   - `access_token` → es el **token de la página**
6. Consulta `TU_ID_DE_PAGINA?fields=instagram_business_account` y copia el `id` que aparece: es tu **ID de cuenta de Instagram**.
7. **Haz que el token no caduque:** pega el token de la página en el **Access Token Debugger** (https://developers.facebook.com/tools/debug/accesstoken) → **Extend Access Token**. El token de página que se obtiene a partir de un token de usuario extendido no vence.

## Paso 4 – Configurar el workflow en n8n

1. Importa `workflows/04_publicar_aprobados_fb_ig.json`.
2. Abre el nodo **Configuración** y pega:
   - `facebook_page_id`
   - `instagram_user_id`
   - `bannerbear_template_dolor`, `bannerbear_template_tip` y `bannerbear_template_oferta` (los 3 IDs del paso 2)
3. En los nodos **Leer aprobados** y **Marcar como publicado**, pega la URL de tu Google Sheet y asigna la credencial de Google Sheets.
4. Crea estas credenciales:

| Credencial (tipo en n8n) | Name | Value | Nodos |
|---|---|---|---|
| **Header Auth** – "Bannerbear" | `Authorization` | `Bearer TU_API_KEY_DE_BANNERBEAR` (con la palabra Bearer y un espacio) | Crear imagen (Bannerbear) |
| **Query Auth** – "Meta Page Token" | `access_token` | el token de la página del paso 3 | Publicar en Facebook, Instagram - Subir imagen, Instagram - Publicar |

## Paso 5 – Probar

1. En la hoja, pon una fila con `dia` = la fecha de **hoy**, `red` = `facebook` y `estado` = `aprobado`.
2. En n8n, haz clic en **Execute workflow**.
3. Revisa que la publicación salga en tu página y que la fila cambie a `publicado`.
4. Repite la prueba con `red` = `instagram`.
5. Cuando todo funcione, **activa** el workflow (interruptor *Active*).

## Errores comunes

| Error | Causa / solución |
|---|---|
| No pasa nada después de "Filtrar publicaciones de hoy" | No hay filas `aprobado` con la fecha de hoy y red facebook/instagram. Revisa la fecha y que `estado` esté en minúscula. |
| `(#200) permissions error` | Faltan permisos en el token (paso 3.3) o la cuenta de Instagram no es profesional o no está vinculada a la página. |
| `Invalid OAuth access token` / `Session has expired` | El token venció. Repite el paso 3.7 para generar uno que no caduque. |
| `Media ID is not available` en Instagram | Instagram aún no termina de procesar la imagen. Sube la espera del nodo "Esperar a que Instagram procese" a 30-60 segundos. |
| Bannerbear `401` | La credencial debe ser `Bearer ` + la API key, con un espacio entre ambas. |
| La imagen sale sin texto | La capa de texto de la plantilla no se llama exactamente `gancho`. |
