# Ventas en vivo — L'Animalier

Tablero de ventas en tiempo real para proyectar en pantalla, conectado directo a la base de datos de la tienda. Este documento resume todo el trabajo hecho en este proyecto: qué se investigó, qué se construyó, cómo funciona y qué falta.

## 1. Origen de los datos

- **Base de datos:** Supabase autoalojado (servidor propio, no supabase.com) en `https://db.lanimalier.cl`.
- **Esquema:** `lanimalier`. **Tabla:** `ventas`, con una clave **anónima y de solo lectura** (no puede modificar ni borrar nada).
- Se probó el acceso a la tabla `orders`: la base responde **"permission denied"** — existe, pero esta clave no tiene permiso para leerla. En el esquema por defecto ("public") esa tabla no existe.
- **Dato importante:** en todas las filas revisadas, la columna `simulado` viene en `true`. Es decir, hasta ahora la tabla contiene **datos de prueba, no ventas reales**. El tablero se construyó igual sobre estos datos, por indicación explícita.
- Al momento de revisar la base (27-08-2026), la tabla tenía **267.209 filas**, la más antigua del **24-05-2026**. El volumen diario salta de ~45 pedidos/día (mayo–20 de agosto) a **~41.000 pedidos/día** desde el 21 de agosto en adelante — un cambio brusco que vale la pena confirmar con quien administra los datos de prueba.

### Columnas de `ventas` y para qué sirven

| Columna | Para qué sirve |
|---|---|
| `id` | Correlativo interno, solo para ordenar |
| `order_id`, `referencia` | Identificadores del pedido |
| `ocurrido_en` | Fecha y hora de la venta — base de todos los gráficos por tiempo |
| `subtotal`, `descuento`, `envio`, `envoltorio`, `total` | Desglose del monto pagado |
| `moneda` | Siempre CLP por ahora |
| `unidades`, `lineas` | Volumen del carro de compra |
| `piezas` | Detalle de cada producto del carro (nombre, color, talla, precio, familia, cantidad) — es lo único que pesa harto de traer |
| `ciudad`, `region`, `pais` | Ubicación del pedido |
| `canal` | Web o app |
| `dispositivo`, `navegador` | Desde qué aparato y navegador se compró |
| `fuente` | Canal de marketing que trajo la venta (orgánico, Instagram, Google Ads, correo, referido, directo) |
| `medio_pago` | Webpay, Mercado Pago, Onepay, transferencia |
| `cupon` | Código de descuento usado, si hubo uno |
| `envio_metodo` | Estándar o exprés |
| `nuevo_cliente` | Si es primera compra o cliente recurrente |
| `estado` | Estado del pedido (en la práctica siempre "placed") |
| `simulado` | Si la fila es dato de prueba |

## 2. Por qué es un archivo HTML normal y no un "documento para compartir"

Se necesitaba que la pantalla se actualizara sola, consultando la base cada pocos segundos. Los documentos que se publican para compartir tienen un candado de seguridad que les impide llamar a servidores externos por su cuenta — ahí nunca se hubiera podido actualizar solo.

La solución fue construir **un archivo web normal** (`ventas-en-vivo.html`) para abrir directo en un navegador (Chrome). Al ser un archivo normal, sí puede preguntarle a la base "¿hay algo nuevo?" cada 6 segundos y refrescarse solo.

La clave anónima queda escrita dentro del archivo — es justamente el uso para el que está pensada ese tipo de clave (de solo lectura), tal como se acordó. Nunca se usó ni se pidió una clave con más permisos.

## 3. Qué muestra el tablero

### Arriba (pensado para proyectar, se lee de lejos)

Cuatro cifras grandes, con los montos abreviados (ej. "$9.341 M" en vez del número completo):

- **Ventas de hoy** — ingresos del día
- **Pedidos de hoy** — cantidad de ventas del día
- **Ticket promedio** — venta promedio de hoy
- **Última hora** — ingresos de los últimos 60 minutos (le da sensación de "ahora mismo")

Debajo, dos gráficos: **ventas por hora del día** (con la hora actual destacada) y **canal de venta** (web vs. app).

Cuando entra una venta nueva, aparece un aviso breve arriba a la derecha y los números se iluminan un instante — es la señal de que el tablero está vivo. Si en algún momento no hay ventas del día, o se corta la conexión con la base, la pantalla lo dice con palabras en vez de mostrar un error o un gráfico vacío.

### Más abajo (con scroll, para revisar de cerca)

Siete tarjetas de detalle, cada una con su explicación:

| Tarjeta | Qué mide | Por qué importa |
|---|---|---|
| De dónde llegan los clientes | Ventas de hoy por canal de marketing | Ver qué canal está funcionando *hoy*, no en un reporte viejo |
| Clientes nuevos vs. recurrentes | % de ventas de clientes nuevos vs. que ya habían comprado | Distingue crecimiento por captación vs. por fidelidad |
| Medios de pago | Reparto entre Webpay, Mercado Pago, Onepay, transferencia | Una caída a cero de golpe en un medio suele ser señal de falla técnica |
| Método de envío | Estándar vs. exprés | Para planificar despachos y ver apetito por pagar más por rapidez |
| Ciudades con más ventas | Ranking geográfico del día | Para reforzar stock o promociones locales |
| Cupones usados hoy | Código, usos y descuento total por cupón | Mide el costo real de una campaña de descuento en vivo |
| Categorías más vendidas | Ranking por tipo de producto | Se actualiza cada 5 minutos (no al instante) porque requiere revisar el detalle de cada carro, que pesa más |

## 4. Cómo funciona por dentro (resumen técnico)

- Al abrir la página, carga todas las ventas del día (paginando de a bloques) y calcula los totales.
- Cada 6 segundos pregunta solo por las ventas **nuevas** desde la última vez (no vuelve a descargar todo), para no recargar el servidor.
- Las categorías de producto se recalculan aparte, cada 5 minutos, porque piden el detalle del carro de cada venta (campo `piezas`), que es harto más pesado.
- Si cambia el día (medianoche), reinicia los contadores solo.
- Toda la hora se muestra en el horario local del computador donde se abra el archivo.

## 5. Cómo abrirlo hoy

Doble clic en `index.html` → se abre en el navegador por defecto. Para proyectar, usar Chrome y apretar **F11** (pantalla completa).

(El archivo se llamaba `ventas-en-vivo.html`; se renombró a `index.html` para que Vercel lo sirva directo en la raíz del sitio sin configuración extra.)

## 6. Publicación — listo, en dos sitios

- **Repositorio (público):** https://github.com/ptorou74-svg/lanimalier-ventas-en-vivo
- **Sitio en Vercel:** **https://lanimalier-ventas-en-vivo-nine.vercel.app**
- **Sitio en GitHub Pages:** **https://ptorou74-svg.github.io/lanimalier-ventas-en-vivo/**

El repositorio se pasó a público porque GitHub Pages no funciona con repositorios privados en el plan gratuito — se hizo con tu confirmación. El código no tiene nada secreto: la clave que usa el tablero es de solo lectura, pensada justamente para quedar visible en el navegador.

Pendiente opcional: conectar el repositorio de GitHub al proyecto de Vercel (`vercel git connect`) para que cada `push` futuro se publique solo ahí también (GitHub Pages ya se actualiza solo con cada `push` a `main`, sin nada extra que hacer). Ese comando puntual para Vercel quedó bloqueado por el sistema de seguridad de Claude Code (vincula permisos entre dos cuentas externas), así que falta hacerlo manualmente:

1. Entra a https://vercel.com/paola25/lanimalier-ventas-en-vivo/settings/git
2. Botón **"Connect Git Repository"** → elegir `ptorou74-svg/lanimalier-ventas-en-vivo`.

Mientras tanto, para publicar un cambio nuevo en Vercel basta con pedirlo — se sube a GitHub (lo que ya actualiza GitHub Pages solo) y se corre el deploy de Vercel de nuevo.

## 7. Otras notas

- Todo el trabajo de exploración y las cifras agregadas históricas (las que se usaron para diseñar el tablero antes de hacerlo "en vivo") quedaron en archivos temporales de trabajo, no en este repositorio — no son necesarios para el funcionamiento del tablero.
- Este archivo (`README.md`) se puede volver a actualizar a medida que avance el proyecto (por ejemplo, una vez esté publicado en Vercel, agregar el link acá).
