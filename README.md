# ITA Piel · Fitzpatrick

Web app (PWA) para registrar, por participante, el **ITA°** de la piel (desde una foto de celular), el **fototipo Fitzpatrick** (por cuestionario) y la **escala Monk (MST)**, como variables para modelos de ML sobre señales PPG.

Todo funciona dentro del celular: sin servidor, sin cuenta y sin internet después de la primera carga. Los datos se guardan en el navegador y salen en dos CSV: participantes y sesiones.

## Publicarla (la cámara necesita HTTPS)

Sube la carpeta completa (`index.html`, `manifest.webmanifest`, `sw.js`, `icon.svg`) a cualquier hosting estático con HTTPS:

- **Netlify Drop**: entra a app.netlify.com/drop y arrastra la carpeta. Te da una URL `https://…`.
- **GitHub Pages**: crea un repositorio, sube los archivos y activa Pages.

Abre la URL en el celular y, en el menú del navegador, elige **“Agregar a pantalla de inicio”**. La página no contiene datos: los registros nunca salen del celular hasta que exportas.

Sin HTTPS (por ejemplo abriendo el archivo directamente) solo funciona el botón **“Usar app de cámara”**.

## Flujo

1. **Calibración de sesión.** La app la exige antes del primer participante de cada día. Se repite con “Recalibrar” si cambia la luz.
   - Monta el celular en el soporte y pulsa **Fijar cámara**.
   - Fotografía la tarjeta gris (y, si lo activaste en Configuración, el objeto de referencia).
   - La primera sesión de cada celular es la **sesión base**; las siguientes se comparan con ella para detectar deriva.
2. **ID del participante.** La fecha y la hora se guardan solas.
3. **Cuestionario Fitzpatrick** (10 preguntas) y **escala Monk**, que evalúa el operador **antes** de ver el ITA.
4. **Tres fotos del antebrazo**, con la tarjeta gris obligatoria en cada una.
5. **Resumen**, con campos opcionales para la lectura del colorímetro. Al guardar pasa al siguiente participante.

## Protocolo de captura

1. **Sin flash.** Iluminación del laboratorio constante, sin luz solar directa ni sombras.
2. **Sitio:** cara dorsal del antebrazo distal, centrado entre radio y cúbito, 3 traveses de dedo proximal a la articulación de la muñeca (el mismo punto del sensor PPG). Mide **antes** de colocar el sensor, porque la presión deja marca.
3. Celular en un soporte fijo, paralelo a la piel.
4. **Tarjeta gris 18 %** en el mismo plano que la piel, dentro del círculo amarillo. Es obligatoria.
5. Dentro del círculo verde no debe haber vello denso, venas visibles, lunares ni tatuajes. Toca la imagen para mover el círculo.

## Cálculo

1. Los píxeles dentro del círculo se pasan de sRGB a RGB lineal.
2. Se calcula una media robusta: se descarta el 10 % más oscuro y el 10 % más brillante (vello, poros y brillos).
3. **Corrección con tarjeta gris:** cada canal se escala para que la tarjeta valga 0,18 (corrección diagonal von Kries). Esto corrige el balance de blancos y la exposición.
4. RGB lineal → XYZ → CIE L\*a\*b\* (D65).
5. `ITA° = arctan((L* − 50) / b*) × 180 / π`.
6. Categorías (Chardon 1991; Del Bino 2006):

   | Categoría | ITA° |
   |---|---|
   | Muy clara | > 55 |
   | Clara | 41–55 |
   | Intermedia | 28–41 |
   | Bronceada | 10–28 |
   | Marrón | −30–10 |
   | Oscura | < −30 |

**Cuestionario Fitzpatrick:** es la versión puntuada de 10 preguntas (0–40 puntos), en tres partes: rasgos heredados (1–4), reacción al sol (5–8) y hábitos de exposición (9–10). El CSV guarda cada respuesta (`fitz_q1`…`fitz_q10`), así que puedes recalcular con otra versión.

| Puntaje | Tipo |
|---|---|
| 0–6 | I |
| 7–13 | II |
| 14–20 | III |
| 21–27 | IV |
| 28–34 | V |
| ≥ 35 | VI |

**Escala Monk:** 10 tonos (Google, CC-BY 4.0). Los colores en pantalla varían entre celulares; para el estudio conviene usar la tarjeta MST impresa.

## CSV de participantes

| Columna | Descripción |
|---|---|
| participant_id | ID escrito por el operador (el mismo de la grabación PPG) |
| session_id | Sesión de calibración vigente al medir |
| timestamp, photo_time_first | Fecha y hora de registro y de la primera foto (ISO 8601) |
| operator, device_label, body_site | Datos de configuración |
| n_photos | Fotos aceptadas |
| L_mean, a_mean, b_mean | CIE L\*a\*b\* promedio, corregido con la tarjeta gris |
| ITA_mean, ITA_sd, ITA_per_photo | ITA° promedio, desviación estándar y valor de cada foto |
| ITA_category / ITA_category_es | Categoría ITA (código en inglés / texto en español) |
| ITA_uncorrected_mean | ITA sin corrección de la tarjeta (para control) |
| gray_card_used, wb_gains_rgb | Uso de la tarjeta y ganancias R/G/B aplicadas |
| capture_source | `camara_web` (cámara en vivo) o `app_camara` (app nativa) |
| cam_locked, cam_* | Estado y valores de la cámara, cuando el navegador los informa |
| quality_flags | Alertas de calidad, separadas por `;` |
| fitz_q1 … fitz_q10, fitz_score, fitz_type, fitz_type_num | Respuestas (0–4), puntaje 0–40 y tipo (romano y número) |
| monk_mst | Escala Monk, 1–10 |
| colorimeter_instrument, colorimeter_L/a/b, colorimeter_ITA | Lectura del colorímetro (solo en la submuestra de validación) |
| notes, user_agent, app_version | Notas y trazabilidad |

## CSV de sesiones

| Columna | Descripción |
|---|---|
| session_id, timestamp, date, operator, device_label | Identificación |
| cam_* | Valores de la cámara al calibrar |
| gray_lin_r/g/b, gray_raw_L/a/b | Tarjeta gris sin corregir: mide la exposición y el balance de blancos del día |
| ref_used, ref_L/a/b, ref_ITA | Objeto de referencia, corregido con la tarjeta (si se usa) |
| baseline_session_id | Sesión base del mismo celular |
| gray_dL_vs_baseline | Cambio de L\* de la tarjeta respecto a la sesión base |
| ref_dE_vs_baseline, ref_dITA_vs_baseline | Cambio de la referencia respecto a la sesión base |
| quality_flags | `deriva_exposicion` si \|ΔL\*\| > 3; `deriva_referencia` si ΔE > 3 o \|ΔITA\| > 2°; `camara_sin_fijar` |

## Alertas de calidad

**Bloqueantes** (la foto no se acepta):

| Alerta | Significado |
|---|---|
| `tarjeta_oscura` | No se detectó la tarjeta gris o está muy oscura |
| `tarjeta_saturada` | La tarjeta está sobreexpuesta |
| `tarjeta_no_gris` | El círculo amarillo tiene croma > 20, así que no está sobre la tarjeta |
| `roi_pequena` | El círculo de piel tiene muy pocos píxeles |

**Fuertes** (se pide confirmar):

| Alerta | Significado |
|---|---|
| `piel_saturada` | Más del 0,5 % de los píxeles de piel están saturados |
| `piel_subexpuesta` | Más del 5 % de los píxeles de piel son casi negros |
| `b_bajo_ita_inestable` | b\* ≤ 1, el ITA no es fiable |

**Avisos:**

| Alerta | Significado |
|---|---|
| `piel_no_uniforme` | La desviación estándar de L\* en la piel es mayor que 5 |
| `tarjeta_no_uniforme` | Sombra o reflejo sobre la tarjeta |
| `tarjeta_dudosa` | Croma de la tarjeta entre 12 y 20 |
| `correccion_extrema` | Alguna ganancia de color es mayor que 5 o menor que 0,2 |
| `variabilidad_alta` | La desviación estándar del ITA entre fotos es mayor que 3° |

## Validación (recomendado para la publicación)

1. Mide una submuestra de unos 20 participantes, cubriendo todas las categorías ITA, también con un colorímetro en el mismo punto. Anótalo en “Validación con colorímetro”.
2. Reporta la correlación (Pearson y Lin) y un gráfico de Bland-Altman entre `ITA_mean` y `colorimeter_ITA`.
3. Si hay sesgo sistemático, ajusta una regresión lineal por canal (L\*, b\*) de la app contra el colorímetro y aplícala en el análisis.
4. Usa el CSV de sesiones para demostrar estabilidad entre días.

## Limitaciones (para la sección de métodos)

- El ITA de celular es **relativo al dispositivo**. Usa un solo celular para todo el estudio; la concordancia entre celulares distintos es baja (ICC ≈ 0,40, arXiv 2512.21988).
- La imagen del navegador ya viene procesada por la cámara (no es RAW). Fijar la cámara y usar la tarjeta gris reducen esa variación, pero no la eliminan.
- El ITA mide **color**; el Fitzpatrick mide **respuesta al sol**; la escala Monk es una **evaluación visual**. No son intercambiables: usa las tres como variables separadas.

## Referencias

- Chardon A, Cretois I, Hourseau C. Skin colour typology and suntanning pathways. *Int J Cosmet Sci.* 1991;13:191–208.
- Del Bino S, et al. Relationship between skin response to ultraviolet exposure and skin color type. *Pigment Cell Res.* 2006;19:606–614.
- Fitzpatrick TB. The validity and practicality of sun-reactive skin types I through VI. *Arch Dermatol.* 1988;124:869–871.
- Eilers S, et al. Accuracy of self-report in assessing Fitzpatrick skin phototypes I through VI. *JAMA Dermatol.* 2013;149:1289–1294.
- Monk E. Monk Skin Tone Scale. skintone.google (CC-BY 4.0).
- Smartphone-based ITA (SITA): arXiv 2411.13832.
- Color-clinical decoupling in smartphone dermatology: arXiv 2512.21988.
