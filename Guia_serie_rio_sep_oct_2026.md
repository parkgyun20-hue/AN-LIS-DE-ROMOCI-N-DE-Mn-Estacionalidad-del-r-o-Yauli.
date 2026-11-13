# Secuencia temporal del cauce del río · septiembre–octubre 2026

**Herramienta:** Google Earth Engine (Code Editor, JavaScript)
**Script:** `serie_rio_sep_oct_2026.js` (misma carpeta)
**Zona:** Junín, río Yauli (La Oroya), afluente del Mantaro. Se puede cambiar a cualquier río dibujando una línea.

---

## 1. Qué produce el script

| # | Salida | Dónde aparece | Para qué sirve en la tesis |
|---|---|---|---|
| 1 | Área de agua del cauce (ha) en cada fecha despejada de sep–oct 2026 | Consola → gráfico de líneas | Muestra cómo cambia el cauce día a día |
| 2 | Índice de agua medio (NDWI o MNDWI) del corredor | Consola → gráfico | Es más sensible que el área en ríos angostos |
| 3 | % del tramo sin nubes por fecha | Consola → barras | Control de calidad: justifica qué fechas se usaron |
| 4 | Compuestos quincenales (1–15 sep, 16–30 sep, 1–15 oct, 16–31 oct) | Mapa (capas) | Figura comparativa del cauce |
| 5 | GIF / timelapse de las escenas despejadas | Consola (animación y enlace) | Secuencia visual del cauce |
| 6 | Septiembre vs. octubre, 2019–2026 | Consola → barras agrupadas | Dice si 2026 fue un año normal, seco o húmedo |
| 7 | Lluvia mensual 2026 vs. promedio 2016–2025 | Consola → barras | **Explica la estacionalidad** (transición de estiaje a lluvias) |
| 8 | Lluvia diaria sep–oct 2026 | Consola → barras | Relaciona los cambios del cauce con eventos de lluvia |
| 9 | Tres CSV (serie diaria, comparación anual, lluvia) | Pestaña *Tasks* → Google Drive, carpeta `GEE_tesis2` | Tablas para Excel y para el informe |

---

## 2. Cómo pegarlo y ejecutarlo en GEE

1. Entra a <https://code.earthengine.google.com> con tu cuenta (la misma del script del lago).
2. Panel izquierdo, pestaña **Scripts** → **NEW → File**. Nómbralo `serie_rio_sep_oct_2026`.
3. Abre el archivo `serie_rio_sep_oct_2026.js`, copia **todo** y pégalo en el editor central (reemplaza lo que haya).
4. Pulsa **Run**. Los gráficos tardan entre 30 s y 2 min en aparecer en la pestaña **Console** (derecha). Si sale *"Computation timed out"*, vuelve a pulsar Run: es normal la primera vez.
5. Para descargar las tablas: pestaña **Tasks** → botón **RUN** en cada tarea → **Run**. Los CSV llegan a tu Google Drive.
6. Para guardar un gráfico como imagen: en cada gráfico, icono ↗ (esquina superior derecha) → **Download PNG/CSV**.
7. Para el GIF: clic en el enlace que imprime la consola → se abre la animación → clic derecho → *Guardar imagen como…*.

> **Tu script del lago no se toca.** Este es un script nuevo. Si quieres tener los dos juntos, pega este debajo del tuyo y borra la sección 1 de este (caja y centroLago ya existen en el tuyo).

---

## 3. Paso clave: verificar (o cambiar) el tramo de río

El trazado por defecto del río Yauli es **aproximado** (lo tracé con coordenadas de Yauli → La Oroya). Antes de usar los resultados:

1. Tras el primer Run, mira el mapa: la **línea blanca** debe ir sobre el río y el **contorno amarillo** (corredor) debe cubrir el cauce.
2. Si no coincide, o si quieres otro río (por ejemplo el **Mantaro a la salida del Lago Chinchaycocha** o el **San Juan**):
   - En el mapa, clic en el ícono de **línea** (herramienta de dibujo, arriba a la izquierda).
   - Haz clic siguiendo el centro del río, punto a punto, y doble clic para terminar.
   - Arriba del código aparecerá **Imports** con una variable `geometry`. Haz clic en el nombre y cámbialo a **`rio`**.
   - Pulsa **Run**. El script usa tu línea automáticamente.
3. Si el río es ancho (> 30 m) o analizas el lago, cambia `INDICE` a `'MNDWI'` (como en tu script original).

---

## 4. Parámetros (sección 0 del script)

| Parámetro | Valor | Cuándo cambiarlo |
|---|---|---|
| `FECHA_INI` / `FECHA_FIN` | `2026-09-01` / `2026-11-01` | La fecha final es exclusiva: así entra el 31 de octubre |
| `INDICE` | `'NDWI'` | NDWI usa B3 y B8 a **10 m**; MNDWI usa B11 a **20 m**, que en un río de 10–15 m de ancho casi no detecta agua |
| `UMBRAL_AGUA` | `0` | Súbelo a `0.1` si aparecen sombras de cerros o carretera como "agua" |
| `BUFFER_M` | `60` | Ancho del corredor a cada lado de la línea. Si tu línea queda desviada, súbelo a 100 |
| `UMBRAL_CS` | `0.60` | Cloud Score+. Bájalo a `0.50` si quedan muy pocas fechas útiles |
| `MIN_CLARO` | `70` | % mínimo del tramo sin nubes para aceptar una fecha |
| `FUENTE_LLUVIA` | `'CHIRPS'` | Si octubre 2026 sale vacío (CHIRPS publica con 3–4 semanas de retraso), cambia a `'ERA5'` |

---

## 5. Método (texto base para la tesis)

Puedes adaptar este párrafo al marco metodológico o a un anexo:

> Para caracterizar la variabilidad estacional del cauce del río Yauli durante septiembre y octubre de 2026 se procesaron en Google Earth Engine las imágenes Sentinel-2 de nivel 2A (reflectancia de superficie, colección armonizada). Las nubes y sombras se enmascararon con Cloud Score+ (umbral *cs_cdf* ≥ 0,60), y las teselas de una misma fecha se unieron en un mosaico diario. Sobre un corredor de 60 m a cada lado del eje del río se calculó el índice de agua de diferencia normalizada (NDWI = (B3 − B8)/(B3 + B8)), y se clasificaron como agua los píxeles con NDWI > 0. Para cada fecha con al menos 70 % del corredor libre de nubes se obtuvieron el área de agua (ha) y el NDWI medio. La estacionalidad se evaluó comparando los compuestos medianos de septiembre y octubre de 2019 a 2026 y contrastando la precipitación mensual de 2026 (CHIRPS) con el promedio 2016–2025 en un radio de 15 km alrededor del tramo.

**Fuentes de datos (para referencias):** Copernicus Sentinel-2 L2A (ESA); Cloud Score+ (Pasquarella et al., 2023, CVPR Workshops); CHIRPS (Funk et al., 2015, *Scientific Data*, 2, 150066); JRC Global Surface Water (Pekel et al., 2016, *Nature*, 540, 418–422).
Ojo: ya estás en el máximo de 30 referencias. Si incluyes esto, cita como mínimo Sentinel-2 y CHIRPS, o ponlo en un anexo.

---

## 6. Cómo interpretar la estacionalidad

**Qué se espera en la sierra central (≈ 3 700–4 100 m s. n. m.):**

- **Junio–agosto:** estiaje fuerte. Casi no llueve y el río va con su caudal mínimo.
- **Septiembre:** sigue el estiaje, con las primeras lluvias aisladas.
- **Octubre:** transición. Aumentan las lluvias y el caudal empieza a subir, aunque la temporada de avenida plena es de diciembre a marzo.

Por eso **septiembre–octubre es una ventana de transición**: deberías ver poco cambio en septiembre y un aumento del área de agua o del NDWI medio hacia la segunda quincena de octubre, si las lluvias llegaron a tiempo.

**Cómo leer los gráficos:**

1. **Gráfico de lluvia mensual.** Si las barras de sep y oct 2026 están por debajo del promedio, el inicio de lluvias se retrasó (año seco). Si están por encima, se adelantó.
2. **Barras sep vs. oct (2019–2026).** En años normales, octubre ≥ septiembre. Compara la diferencia de 2026 con la de otros años.
3. **Serie diaria.** Busca saltos del área de agua 1–3 días después de los días de lluvia del gráfico diario.

**Relación con el manganeso (el puente con tu tesis):** en estiaje hay menos dilución, así que la concentración de Mn²⁺ del río tiende a ser **mayor**; con las lluvias aumenta el caudal y suele bajar la concentración, aunque las primeras lluvias pueden arrastrar sedimentos y relaves y producir picos puntuales. Si el periodo sep–oct 2026 resulta de bajo caudal, ese es un argumento para que el protocolo se diseñe con la condición más desfavorable (estiaje). Es una **hipótesis a contrastar** con los monitoreos de la ANA (RYaul6), no un resultado de este script.

---

## 7. Limitaciones (decirlas evita observaciones del jurado)

- **El río Yauli es angosto (≈ 10–20 m).** Con píxeles de 10 m, el área de agua tiene un error de ± 1 píxel por borde. Los cambios pequeños de caudal **no** se notan en el ancho. El satélite muestra la tendencia, **no** mide caudal; para caudal habría que usar una estación (SENAMHI/ANA).
- **Nubes.** Octubre ya tiene nubosidad; puede haber pocas fechas útiles. El gráfico de % despejado lo documenta.
- **Sombras de cerros y asfalto** pueden aparecer como agua con NDWI. Por eso el análisis se restringe a un corredor estrecho y se puede subir el umbral.
- **Turbidez.** Agua muy cargada de sedimentos o de relaves puede bajar el NDWI y "perder" píxeles de agua.
- **CHIRPS** tiene ~5 km de resolución y subestima la lluvia en algunas zonas altoandinas. Sirve para comparar entre años, no como valor exacto.

---

## 8. Si algo falla

| Mensaje / síntoma | Solución |
|---|---|
| `Fechas con ≥ 70 % … : 0` | Baja `MIN_CLARO` a 50 o `UMBRAL_CS` a 0.50 |
| Área de agua siempre ≈ 0 | La línea no está sobre el río: dibújala (sección 3) o sube `BUFFER_M` a 100 |
| Área de agua muy alta en fechas sueltas | Sombras o nieve: sube `UMBRAL_AGUA` a 0.1 |
| Barras de oct 2026 vacías en lluvia | CHIRPS aún no publica octubre: usa `FUENTE_LLUVIA = 'ERA5'` |
| `User memory limit exceeded` / `timed out` | Pulsa Run de nuevo; si persiste, reduce `ANIOS_COMP` a 4 años |
| `rio is not defined` | La variable dibujada no se llama exactamente `rio` (minúsculas) |
