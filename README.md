# Bitácora de auditoría — AstroBitácora

**Estudiante:** _(bryan Ronaldo flores aviles)_
**URL publicada:** https://practiaca-05.vercel.app
**Fecha:** 1 de octubre de 2026

**Condiciones de medición (iguales antes y después):** PageSpeed Insights, Lighthouse 13.5.0, carga inicial de una sola página. Móvil: Moto G Power emulado con limitación 4G lenta. Escritorio: emulado con limitación personalizada. Medición inicial a las 15:10 (GMT-6) y medición final a las 15:21 (GMT-6).

> El modo **móvil** es la referencia principal. El de escritorio se incluye como comparación complementaria.

---

## 1. Medición inicial

### Móvil

| Indicador | Resultado inicial | Observación |
|---|---|---|
| Desempeño | 75 | Engañoso: FCP, TBT y CLS estaban bien, pero el LCP aportaba +0 puntos. |
| LCP | 126,9 s | Efecto de descargar ~30 MB con 4G lenta. Es el problema central. |
| CLS | 0 | En móvil no se vio desplazamiento, aunque las imágenes no tenían dimensiones. |
| INP | No disponible | No hay datos de usuarios reales (sitio nuevo). |
| Bytes transferidos / peso total | 31.105 KiB (~30,4 MB) | Casi todo corresponde a las 6 imágenes JPEG. |

### Escritorio

| Indicador | Resultado inicial | Observación |
|---|---|---|
| Desempeño | 76 | |
| LCP | 5,8 s | Muy por encima del objetivo de 2,5 s. |
| CLS | 0,057 | Lighthouse lo atribuyó a imágenes sin `width` y `height`. |
| INP | No disponible | Sin datos de usuarios reales. |
| Bytes transferidos / peso total | 31.105 KiB | Igual que en móvil. |

### Tres hallazgos principales

1. **Hallazgo:** Las imágenes se entregan en JPEG, sin formato moderno y sin ajustar al tamaño con que se muestran.
   **Evidencia:** "Mejorar la entrega de imágenes" indicaba un ahorro estimado de 30.510 KiB en móvil y 30.665 KiB en escritorio.
   **Recurso o archivo relacionado:** Las 6 imágenes de Cloudinary referenciadas en `index.html` (hero y 5 tarjetas).

2. **Hallazgo:** El LCP es enorme por el peso de la imagen principal, que se descarga completa antes de poder mostrarse.
   **Evidencia:** LCP de 126,9 s en móvil y 5,8 s en escritorio. Peso total de 31.105 KiB ("Evita cargas útiles de red de gran tamaño").
   **Recurso o archivo relacionado:** `hero-cosmos_11zon_uqgf6b.jpg` (`.hero__image` en `index.html`).

3. **Hallazgo:** Las imágenes no tienen `width` y `height` explícitos, por lo que el navegador no reserva su espacio.
   **Evidencia:** Diagnóstico "Los elementos de imagen no tienen width y height explícitos" y CLS de 0,057 en escritorio ("Causantes de los cambios de diseño").
   **Recurso o archivo relacionado:** Todas las etiquetas `<img>` de `index.html`.

**Hallazgo secundario:** "Solicitudes que bloquean el renderizado" con ahorro estimado de 430 ms en móvil y 150 ms en escritorio. Se relaciona con `<script src="script.js">` sin `defer` y con `styles.css`.

---

## 2. Hipótesis antes de modificar

**Preguntas de inspección**

- **¿Qué archivos pesan más?** Las imágenes: 31.105 KiB en total entre 6 archivos, mientras que HTML, CSS y JS pesan pocos KiB.
- **¿Todas las imágenes necesitan descargarse de inmediato?** No. Solo el hero está en el primer viewport; las 5 tarjetas están más abajo.
- **¿Hay imágenes sin dimensiones explícitas?** Sí, las 6.
- **¿El recurso principal usa un formato y tamaño razonables?** No. Es un JPEG sin formato moderno y a un tamaño muy superior al que se muestra.
- **¿Cómo se carga el JavaScript?** Con `<script src="script.js">` en el `<head>`, sin `defer` ni `async`.
- **¿Hay recursos que podrían entregarse en un formato más eficiente?** Sí, todas las imágenes (WebP/AVIF en vez de JPEG).

**Hipótesis**

1. Si cambio las URLs de Cloudinary para entregar formato moderno y un ancho acorde al diseño (`f_auto,q_auto,w_...`), espero mejorar el peso total y el LCP porque el navegador descargará muchos menos bytes por imagen.
2. Si agrego `width` y `height` a todas las `<img>`, espero mejorar el CLS porque el navegador reservará el espacio antes de que termine la descarga y el contenido no se moverá.
3. Si agrego `loading="lazy"` a las 5 tarjetas y dejo el hero con `fetchpriority="high"` y sin `lazy`, espero mejorar el LCP y el ancho de banda inicial porque las imágenes fuera del viewport dejarán de competir con el hero.
4. Si agrego `defer` al script principal, espero reducir el bloqueo del renderizado porque el HTML se analizará sin esperar la descarga y ejecución del JavaScript.

---

## 3. Cambios aplicados

| Cambio | Motivo | Archivo(s) | Resultado esperado |
|---|---|---|---|
| `f_auto,q_auto` en las URLs de Cloudinary | Entregar WebP/AVIF en lugar de JPEG pesado | `index.html` | Gran reducción del peso total |
| Imágenes redimensionadas con `w_800`, `w_1200` y `w_1600`, más `srcset` y `sizes` | Evitar descargar más píxeles de los que se muestran, sobre todo en móvil | `index.html` | Menor peso y menor LCP |
| `fetchpriority="high"` en el hero, sin `loading="lazy"` | El hero es el elemento LCP y debe descargarse con prioridad | `index.html` | Menor LCP |
| `loading="lazy"` y `decoding="async"` en las 5 tarjetas | Diferir imágenes que están fuera del primer viewport | `index.html` | Menos bytes en la carga inicial |
| `width` y `height` en las 6 imágenes | Reservar espacio y evitar saltos de diseño | `index.html` | CLS cercano a 0 |
| `defer` en `script.js` | Quitar el bloqueo del análisis del HTML | `index.html` | Menos bloqueo del renderizado |
| `<link rel="preconnect">` a `res.cloudinary.com` | Adelantar la conexión al servidor de imágenes | `index.html` | Descubrimiento y descarga de imágenes más rápidos |

**Nota:** los valores de `width` y `height` (1600×900 para el hero, 1200×675 para las tarjetas anchas y 800×600 para las demás) son proporciones estimadas. El CLS de 0 en ambas mediciones finales indica que reservan correctamente el espacio, pero conviene verificarlos contra las dimensiones reales.

---

## 4. Medición final

### Móvil (referencia principal)

| Indicador | Antes | Después | Diferencia |
|---|---|---|---|
| Desempeño | 75 | 98 | +23 puntos |
| LCP | 126,9 s | 1,7 s | −125,2 s (−98,7 %) |
| CLS | 0 | 0 | Sin cambio |
| INP | No disponible | No disponible | — |
| Bytes transferidos / peso total | 31.105 KiB | _(completar con Network)_ | _(completar)_ |
| FCP | 0,8 s | 0,8 s | Sin cambio |
| TBT | 0 ms | 0 ms | Sin cambio |
| Speed Index | 0,8 s | 3,6 s | +2,8 s (empeoró) |

### Escritorio

| Indicador | Antes | Después | Diferencia |
|---|---|---|---|
| Desempeño | 76 | 100 | +24 puntos |
| LCP | 5,8 s | 0,5 s | −5,3 s (−91 %) |
| CLS | 0,057 | 0 | −0,057 |
| INP | No disponible | No disponible | — |
| Bytes transferidos / peso total | 31.105 KiB | _(completar con Network)_ | _(completar)_ |
| FCP | 0,2 s | 0,2 s | Sin cambio |
| TBT | 0 ms | 0 ms | Sin cambio |
| Speed Index | 0,6 s | 0,5 s | −0,1 s |

**Indicio del peso final:** tras los cambios, "Mejorar la entrega de imágenes" solo estima 51 KiB de ahorro posible en móvil y 21 KiB en escritorio, frente a más de 30.000 KiB al inicio. Casi todo el peso excesivo desapareció. El valor exacto de "Tamaño total" debe tomarse de la pestaña Network o del diagnóstico "Evita cargas útiles de red de gran tamaño".

**Evidencias:** guardar las capturas en `evidencias/` con los nombres `medicion-inicial-movil.png`, `medicion-inicial-escritorio.png`, `medicion-final-movil.png` y `medicion-final-escritorio.png`.

---

## 5. Conclusión breve

**Cambio con mayor impacto:** la optimización de imágenes (formato moderno con `f_auto,q_auto` y redimensionado con `w_...` y `srcset`). Fue la que eliminó casi todos los ~30 MB de peso y llevó el LCP de 126,9 s a 1,7 s en móvil. No se puede separar con precisión cuánto aportó cada parte (formato, tamaño o `srcset`) porque se aplicaron juntas.

**Cambio con menor impacto:** `defer` en el script y `preconnect`. El JavaScript es mínimo y el TBT ya era 0 ms, por lo que no hubo mejora visible. "Solicitudes que bloquean el renderizado" sigue apareciendo (150 ms en móvil), probablemente por el CSS.

**Problema que todavía queda pendiente:** en móvil el Speed Index subió de 0,8 s a 3,6 s, y todavía quedan 51 KiB de ahorro posible en imágenes y 150 ms de recursos que bloquean el renderizado.

**¿Qué harías en una segunda iteración?**

1. Verificar las dimensiones reales de las imágenes y ajustar `width` y `height`.
2. Revisar si las imágenes del primer viewport pueden bajar más de peso (por ejemplo, reduciendo `q_auto` o el ancho del hero en móvil).
3. Reducir el bloqueo del CSS (CSS crítico en línea o minificación de `styles.css`).
4. Minificar `styles.css` y `script.js` como optimización secundaria.
5. Repetir la medición varias veces y usar la mediana, porque PageSpeed varía entre corridas.

---

## Preguntas finales

**1. ¿Qué significa LCP y cuál fue el elemento LCP antes y después?**
LCP (Largest Contentful Paint) mide cuánto tarda en mostrarse el elemento de contenido más grande visible en el primer viewport; el objetivo recomendado es 2,5 s o menos. Antes y después, el elemento LCP esperado es la imagen hero (`.hero__image`). _(Confirmar en el "Desglose de LCP" de PageSpeed y anotar aquí lo que indique.)_ Antes tardaba 126,9 s en móvil y 5,8 s en escritorio. Después tarda 1,7 s y 0,5 s.

**2. ¿Cómo puede una imagen sin dimensiones explícitas contribuir a CLS?**
Sin `width` y `height`, el navegador no sabe qué espacio ocupará la imagen hasta que descarga sus datos. Mientras tanto muestra el contenido sin ese espacio y, al llegar la imagen, empuja hacia abajo todo lo que sigue. Ese desplazamiento inesperado es lo que mide el CLS. Aquí se vio un CLS de 0,057 en escritorio antes de optimizar y de 0 después de agregar las dimensiones.

**3. ¿Por qué `loading="lazy"` es útil en una galería, pero puede ser mala idea en una imagen principal?**
En una galería, las imágenes están fuera del primer viewport, así que diferirlas ahorra ancho de banda y deja los recursos libres para lo importante. En la imagen principal ocurre lo contrario: es visible al cargar y suele ser el elemento LCP. Con `lazy`, el navegador espera a calcular el layout para decidir si la descarga, y eso retrasa el LCP. Por eso el hero usa `fetchpriority="high"` y no `lazy`.

**4. Diferencia entre `defer` y `async`. ¿Cuál elegiste y por qué?**
Ambos descargan el script sin bloquear el análisis del HTML. `async` lo ejecuta apenas termina de descargarse, en cualquier orden y posiblemente antes de que el DOM esté listo. `defer` lo ejecuta cuando el HTML ya fue analizado y respeta el orden de los scripts. Elegí **`defer`** porque `script.js` manipula elementos del DOM (menú, contador, año) y es un script propio y único; además ya usa `DOMContentLoaded`, así que no se rompe.

**5. ¿Qué formato de imagen elegí y qué comparación de peso obtuve?**
Usé `f_auto,q_auto` de Cloudinary, que entrega automáticamente el formato más eficiente que soporte el navegador (WebP o AVIF) en lugar de JPEG, junto con redimensionado a `w_800`, `w_1200` o `w_1600`. Antes el total era de 31.105 KiB (~30,4 MB). Después: _(completar con el peso final de Network)_. Referencia: el ahorro posible restante en imágenes bajó de más de 30.000 KiB a 51 KiB en móvil.

**6. Si el puntaje sube, pero el LCP sigue por encima del objetivo, ¿la optimización está terminada?**
No. El puntaje es un resumen ponderado de varias métricas y puede ocultar un LCP malo. Aquí se vio al inicio: 75 puntos con un LCP de 126,9 s. El LCP es una métrica Core Web Vitals que refleja cuánto espera el usuario para ver el contenido principal. Mientras supere los 2,5 s hay que seguir optimizando, aunque el puntaje parezca bueno. En este caso el LCP final (1,7 s en móvil y 0,5 s en escritorio) sí está dentro del objetivo.
