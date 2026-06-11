# Ejercicio 1 — Respuestas: Exploración de datos y calidad del dataset

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Repositorio:** https://github.com/JeffSackmann/tennis_MatchChartingProject  

---

## Pregunta 1 — ¿Qué tablas son imprescindibles para describir saque, resto, rally, red y puntos de presión?

El dataset cubre las cinco dimensiones tácticas con las siguientes tablas:

| Dimensión táctica | Tabla principal | Tabla complementaria | Motivo |
|---|---|---|---|
| **Saque** | `Overview` (`set = "Total"`) | `ServeBasics`, `ServeDirection` | `Overview` da el rendimiento global (% 1S, puntos ganados). `ServeBasics` añade dirección agregada y puntos resueltos en ≤3 golpes. `ServeDirection` desglosa por zona (wide / body / T) para el heatmap. |
| **Resto** | `ReturnDepth` (`row = "Total"`) | `ReturnOutcomes` | `ReturnDepth` mide la profundidad del resto (shallow / deep / very_deep). `ReturnOutcomes` añade el outcome del resto (winner, error, punto continuado) y el rally length resultante. |
| **Rally** | `ReturnOutcomes` (`row = "7"`, `"89"`, `"9"`) | `Overview` (`winners`, `unforced`) | `ReturnOutcomes` permite segmentar por duración del rally usando los valores numéricos de `row`. `Overview` complementa con winners y errores no forzados totales. `charting-m-stats-Rally.csv` fue **descartada** del modelo por no permitir identificar el rol de Djokovic de forma fiable. |
| **Red** | `NetPoints` (`row = "NetPoints"`) | — | Registra frecuencia y efectividad en red. Los otros valores de `row` (`Approach`, `NetPointsRallies`, `ApproachRallies`) son subconjuntos anidados — nunca agregar sin filtro. |
| **Puntos de presión** | `KeyPointsServe` (`row = "BP"`) | `KeyPointsReturn` (`row = "BPO"`) | `KeyPointsServe` cubre cómo Djokovic gestiona los break points **en su saque**. `KeyPointsReturn` cubre cómo los convierte **al resto**. Ambas tablas cubren también game points y deuce. |

**Tabla de dimensión central:** `Matches` — no es una tabla de hechos sino la dimensión que propaga filtros (superficie, año, rival, torneo) al resto del modelo mediante la clave `match_id`.

---

## Pregunta 2 — ¿Cuál es la granularidad de cada tabla?

| Tabla | Granularidad | Filas por partido (aprox.) | Clave natural | Nota crítica |
|---|---|---|---|---|
| `Matches` | 1 fila = 1 partido | 1 | `match_id` | Tabla dimensión. Sin filtro de Djokovic en Power Query — es la única tabla completa. |
| `Overview` | 1 fila = 1 jugador × 1 set | 6–12 (2 jugadores × sets jugados + Total) | `match_id + player + set` | Usar siempre `set = "Total"` para análisis de partido completo. Sin ese filtro las cifras se multiplican por el número de sets. |
| `ServeBasics` | 1 fila = 1 jugador × tipo de saque | 3 (row = "1", "2", "Total") | `match_id + player + row` | Granularidad jugador-partido-saque. |
| `ServeDirection` | 1 fila = 1 jugador × tipo de saque | 3 (row = "1", "2", "Total") | `match_id + player + row` | Mismo nivel que `ServeBasics`. Para el heatmap usar `row = "1"` y `"2"` por separado. |
| `ReturnDepth` | 1 fila = 1 jugador × saque recibido × tipo de golpe × contexto | **18** | `match_id + player + row` | La tabla más granular del modelo. Cruzar tres dimensiones simultáneas: tipo de saque, tipo de golpe y contexto. Para análisis global usar `row = "Total"`. |
| `ReturnOutcomes` | 1 fila = 1 jugador × tipo de saque × duración de rally | Variable (~21) | `match_id + player + row` | `row` combina tipo de saque (`v1st`, `v2nd`) y bucket de rally length (1–3, 4–6, 7–9, 10+). Filtro obligatorio o los totales se multiplican hasta 21×. |
| `NetPoints` | 1 fila = 1 jugador × tipo de punto de red | 4 (`NetPoints`, `Approach`, `NetPointsRallies`, `ApproachRallies`) | `match_id + player + row` | Los 4 valores son subconjuntos anidados, no categorías excluyentes. KPI principal: `row = "NetPoints"`. |
| `KeyPointsServe` | 1 fila = 1 jugador × situación de marcador | 4 (`BP`, `GP`, `Deuce`, `STotal`) | `match_id + player + row` | Cubre solo el ~40–50% de los puntos al saque (los jugados en situación de presión). |
| `KeyPointsReturn` | 1 fila = 1 jugador × situación de marcador | 4 (`BPO`, `GPO`, `DeuceR`, `RTotal`) | `match_id + player + row` | Simétrica a `KeyPointsServe` pero desde la perspectiva del restador. Tiene 4 columnas menos que su par al saque. |

---

## Pregunta 3 — ¿Cuántos partidos de Djokovic aparecen en la muestra y cómo se distribuyen?

**KPIs globales**

| Métrica | Valor |
|---|---|
| Partidos charteados | **553** |
| Rivales únicos | **161** |
| Torneos únicos | **51** |
| Superficies disponibles | Hard, Clay, Grass |

### Distribución por superficie

| Superficie | Partidos | % del total |
|---|---|---|
| Hard | 366 | 66,2% |
| Clay | 129 | 23,3% |
| Grass | 58 | 10,5% |

La muestra está claramente sesgada hacia pista dura, que concentra dos tercios de los partidos. Grass es la superficie con menor evidencia (58 partidos), lo que limita la fiabilidad estadística de los análisis tácticos en hierba. Cualquier conclusión sobre Wimbledon debe tratarse con cautela.

### Distribución por año

| Año | Partidos | | Año | Partidos |
|---|---|---|---|---|
| 2005 | 2 | | 2016 | 28 |
| 2006 | 5 | | 2017 | 14 |
| 2007 | 22 | | 2018 | 25 |
| 2008 | 21 | | 2019 | 28 |
| 2009 | 22 | | 2020 | 12 |
| 2010 | 18 | | 2021 | 29 |
| 2011 | 25 | | 2022 | 31 |
| 2012 | 41 | | 2023 | 45 |
| 2013 | 33 | | 2024 | 38 |
| 2014 | 20 | | 2025 | 37 |
| 2015 | 51 | | 2026 | 6\* |

\*2026 en curso (datos parciales a fecha de análisis).

Los años con mayor volumen de partidos charteados son 2015 (51), 2023 (45) y 2012 (41). Los años 2005–2006 tienen cobertura muy escasa (2 y 5 partidos respectivamente) y no son representativos del perfil táctico de Djokovic en su etapa de madurez. El período 2020 (12 partidos) refleja la reducción del calendario por la pandemia.

### Top 10 torneos por número de partidos

| Torneo | Partidos |
|---|---|
| Australian Open | 62 |
| US Open | 51 |
| Wimbledon | 48 |
| Tour Finals | 44 |
| Roland Garros | 40 |
| Rome Masters | 32 |
| Monte Carlo Masters | 28 |
| Paris Masters | 24 |
| Shanghai Masters | 23 |
| Miami Masters | 22 |

Los cuatro Grand Slams concentran 201 partidos (36,3% del total). El Australian Open lidera con diferencia, coherente con los múltiples títulos de Djokovic allí. El Tour Finals (44 partidos) tiene peso relevante pero sus condiciones indoor dificultan la extrapolación táctica a otros torneos.

### Top 10 rivales por número de partidos

| Rival | Partidos |
|---|---|
| Rafael Nadal | 52 |
| Roger Federer | 47 |
| Andy Murray | 30 |
| Tomas Berdych | 15 |
| Juan Martín Del Potro | 12 |
| Stefanos Tsitsipas | 11 |
| Gaël Monfils | 11 |
| Stan Wawrinka | 11 |
| Jo-Wilfried Tsonga | 10 |
| Jannik Sinner | 10 |

Nadal (52) y Federer (47) dominan el ranking de rivales, lo que introduce un sesgo potencial: Djokovic adapta su táctica de forma marcada contra ambos jugadores. Cualquier análisis de "estilo de juego medio" puede estar distorsionado por la alta frecuencia de estos matchups. Sinner y Alcaraz (7 partidos) tienen cobertura aún limitada para análisis de matchup detallado.

---

## Pregunta 4 — ¿La muestra permite comparar carrera completa contra rendimiento reciente? ¿Elegiríais una ventana temporal?

### Cobertura temporal

| Métrica | Valor |
|---|---|
| Primer año con datos | 2005 |
| Último año con datos | 2026 (parcial) |
| Temporadas cubiertas | **22** (sin huecos) |
| Rango total | 22 años consecutivos |

La cobertura es continua año a año, sin huecos. Esto permite construir series temporales sin interpolación.

### ¿Permite comparar carrera completa vs. rendimiento reciente?

Sí, la muestra lo permite con matices. Los 553 partidos cubren desde los inicios de Djokovic en el circuito (2005) hasta la actualidad (2026), con suficiente volumen en casi todos los años para trazar una evolución táctica. Sin embargo, los años 2005–2006 (7 partidos en total) tienen escasa representatividad estadística.

### Ventana temporal elegida: **2015–2025**

Se propone una ventana de **11 temporadas (2015–2025)** como marco principal de análisis, por las razones siguientes:

**Razones a favor de este corte:**

1. **Madurez táctica consolidada.** 2015 marca el inicio de la etapa de mayor dominio de Djokovic (cierre del Grand Slam en 2016). Su estilo de juego en pista dura, clay y hierba está completamente definido a partir de ese año.
2. **Volumen suficiente por año.** El período 2015–2025 acumula **363 partidos** con una media de 33 por temporada, suficiente para análisis por superficie y ronda.
3. **Relevancia táctica actual.** Los datos de 2005–2014 reflejan un Djokovic distinto — menor madurez física, estilo más conservador y diferentes condiciones de competición. Incluirlos sin segmentación distorsionaría los patrones actuales.
4. **Comparación interna posible.** Dentro de la ventana 2015–2025 se pueden definir dos subperíodos: **2015–2019** (cima de dominio) y **2020–2025** (etapa tardía con cambios físicos y de calendario), lo que permite la comparación carrera-reciente sin necesidad de recurrir a los años más antiguos.

**Excepción:** para análisis de Grand Slam por superficie conviene ampliar a **2012–2025** dado que 2012–2014 tiene buena cobertura (94 partidos en 3 años) y los Grand Slams de ese período son tácticamente comparables.

---

## Pregunta 5 — ¿Qué columnas presentan nulos, formatos inconsistentes o valores difíciles de interpretar?

### Tabla `Matches`

| Columna | Problema | Severidad | Acción |
|---|---|---|---|
| `Time` | Formato inconsistente (mezcla 12h, 24h y texto libre: `"5pm"`, `"14:30"`, `"NaN"`). Alta tasa de nulos. | Alta | **Descartar** — sin valor táctico |
| `Court` | Mezcla nombres propios, números de pista y nombres de patrocinador (`"Centre"`, `"7"`, `"Paribas"`). Alta tasa de nulos. | Alta | **Descartar** — inconsistencia irrecuperable |
| `Umpire` | Nulos frecuentes. Sin valor táctico. | Media | **Descartar** |
| `Date` | Viene como `object` (string AAAAMMDD), no como fecha. | Media | **Convertir** a tipo date en Power Query (`Date.FromText`) |
| `Best of` | Viene como `object` aunque solo contiene `"3"` o `"5"`. | Baja | **Convertir** a `Int64` en Power Query |
| `Final TB?` | Valores: `"1"`, `"A"`, `NaN`. El `NaN` no distingue entre "no hubo tie-break" y "dato no registrado". | Media | Tratar `NaN` como "sin tie-break decisivo" — documentar el supuesto |
| `Tournament` | Texto libre con variantes ortográficas del mismo torneo a lo largo de los años (posible). | Baja | Verificar consistencia antes de usar como filtro — normalizar con trim y mayúsculas |

### Tablas de hechos (transversal)

| Problema | Tablas afectadas | Acción |
|---|---|---|
| **Duplicados** por re-charting del mismo partido por distintos colaboradores | `Overview` (56), `ServeBasics` (47), `ServeDirection` (47), `ReturnDepth` (323), `ReturnOutcomes` (378), `NetPoints` (74), `KeyPointsServe` (84), `KeyPointsReturn` (84) | Eliminar con `Table.Distinct` sobre clave compuesta (`match_id + player + row/set`) en Power Query |
| **Columna `row` opaca** — valores numéricos o abreviados sin descripción en el CSV | `ReturnDepth`, `ReturnOutcomes`, `KeyPointsServe`, `KeyPointsReturn` | Usar los decodificadores de los diccionarios de datos. Sin filtro activo, los totales son incorrectos (hasta 21×). |
| **Columnas con frecuencia nula** (`err_foot`, `err_unknown` en ServeDirection; `err_wide_deep` en ReturnDepth; `passing_shot_induced_forced` en NetPoints) | Varias | Descartar en ingesta Power Query — sin información táctica útil |
| **Sesgo de charter** (`Charted by`) — distintos colaboradores pueden clasificar de forma diferente los errores forzados y no forzados | Todas las tablas de hechos | No usar como filtro analítico; mencionar en la sección de limitaciones del informe |
| **Encoding** — nombres de jugadores con caracteres especiales (Đoković, tildes, caracteres del este de Europa) | Todas | Aplicar `Encoding = 65001` (UTF-8) uniformemente en Power Query |

---

## Pregunta 7 — ¿Qué preguntas tácticas pueden responderse con datos agregados y cuáles requieren nivel punto a punto?

### Respondibles con datos agregados (nivel partido o jugador-partido)

Las tablas del modelo son todas **pre-agregadas** — registran totales y conteos por partido, no golpe a golpe. Con ellas se puede responder:

| Pregunta táctica | Tabla que la responde |
|---|---|
| ¿Qué % de primeros saques entra y cuántos puntos gana con ellos? | `Overview` / `ServeBasics` |
| ¿A qué zona saca más frecuentemente en deuce y advantage court? | `ServeDirection` |
| ¿Cuántos puntos resuelve en 3 golpes o menos? | `ServeBasics` (`pts_won_lte_3_shots`) |
| ¿Cómo rinde al resto en términos globales? | `Overview` (`return_pts_won / return_pts`) |
| ¿A qué profundidad devuelve? ¿Cuántos restos son superficiales? | `ReturnDepth` |
| ¿Con qué frecuencia sube a la red y qué % de puntos gana allí? | `NetPoints` |
| ¿Cómo gestiona los break points en su saque y en el del rival? | `KeyPointsServe` / `KeyPointsReturn` |
| ¿Los rallies largos le favorecen o le perjudican? | `ReturnOutcomes` (buckets de rally length) |
| ¿Hay diferencias tácticas entre superficies? | Cualquier tabla filtrada por `Surface` vía `Matches` |
| ¿Qué distingue sus victorias de sus derrotas? | Comparación de métricas de `Overview` por resultado (requiere columna de resultado — ver limitación) |

### Requieren nivel punto a punto (no disponibles en este modelo)

| Pregunta táctica | Por qué no se puede responder |
|---|---|
| ¿En qué combinación concreta de golpes termina cada punto? | Requeriría el CSV de secuencias punto a punto (`charting-m-points.csv`), no incluido en el modelo. |
| ¿Desde qué zona de la pista golpea Djokovic sus winners? | Necesita coordenadas de impacto — no disponibles en los CSV del modelo. |
| ¿Con qué efecto (topspin / slice / plano) ejecuta su segundo saque según superficie? | La dirección del saque está disponible pero no el efecto del golpe. |
| ¿Cómo varía su táctica punto a punto dentro de un mismo juego bajo presión? | Los datos de `KeyPointsServe/Return` son agregados por partido, no secuenciales. |
| ¿Anticipa el saque rival o reacciona? Tiempo de reacción al resto. | Dato biomecánico — fuera del alcance del dataset. |

**Conclusión:** el dataset es suficiente para un scouting táctico cuantitativo de primer nivel — patrones de saque, presión en break points, preferencia de rally length y efectividad en red — pero no permite análisis secuencial golpe a golpe ni análisis posicional dentro de la pista.

---

## Pregunta 8 — ¿Qué filtros mínimos deben estar presentes en todo el informe?

Tres filtros son **obligatorios** en todo el informe. Sin ellos, los datos mezclan contextos incomparables o los totales son aritméticamente incorrectos.

### Filtro 1 — Superficie (`Surface`)

**Obligatorio en todas las páginas.** Hard, Clay y Grass tienen distribuciones de juego radicalmente distintas en saque, rally length y juego en red. Agregarlas sin segmentar produce métricas que no describen ningún contexto real. El informe debe tener siempre activo un slicer de superficie, o páginas independientes por superficie.

### Filtro 2 — Jugador (`player = "Novak Djokovic"`)

**Aplicado en Power Query, no en DAX.** Las 8 tablas de hechos se filtran en el origen para contener únicamente registros de Djokovic. La tabla `Matches` se mantiene completa (sin filtro) porque es la dimensión que propaga los filtros contextuales. Este filtro ya está incorporado en la ingesta y no requiere slicer en el informe.

### Filtro 3 — Subgrupo de filas (`row` / `set`)

**Obligatorio en cada medida DAX.** Cada tabla de hechos contiene múltiples filas por partido que representan subconjuntos distintos (sets, tipos de saque, situaciones de marcador). Sin un filtro explícito sobre `row` o `set`, las medidas suman contextos incompatibles y los resultados son incorrectos por factores de hasta 21× (`ReturnOutcomes`). Cada medida debe especificar el valor de `row` que corresponde a su contexto:

| Tabla | Valor de `row` / `set` para el KPI principal |
|---|---|
| `Overview` | `set = "Total"` |
| `ServeBasics` | `row = "Total"` |
| `ServeDirection` | `row = "1"` (primer saque) o `"2"` (segundo) |
| `ReturnDepth` | `row = "Total"` |
| `ReturnOutcomes` | `row = "v1st"` o `"v2nd"` para outcomes; `"7"`, `"89"`, `"9"` para rally length |
| `NetPoints` | `row = "NetPoints"` |
| `KeyPointsServe` | `row = "BP"` (break points) o `"STotal"` (presión global) |
| `KeyPointsReturn` | `row = "BPO"` (break points) o `"RTotal"` (presión global) |

### Filtros recomendados adicionales (no obligatorios pero necesarios para análisis fiable)

| Filtro | Justificación |
|---|---|
| **Rango temporal** (recomendado: 2015–2025) | Los datos de 2005–2014 reflejan un perfil táctico distinto. Sin segmentación temporal se mezclan etapas incomparables. |
| **Best of** (`3` vs `5`) | Un partido de Grand Slam (5 sets) tiene una dinámica de gestión de energía y presión distinta a un Masters (3 sets). Mezclarlos puede distorsionar los KPIs de break points y rendimiento en sets decisivos. |
| **Round** (fases eliminatorias vs. clasificación) | Las rondas de clasificación (Q1, Q2, Q3) tienen un perfil de rival distinto. Para análisis de presión competitiva real, filtrar a rondas de cuadro principal (R128 en adelante). |

---

*Fuente de datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*  
*Cálculos ejecutados sobre `charting-m-matches.csv` mediante `02_cobertura_djokovic.ipynb`*  
*Última actualización: Ejercicio 1 — Exploración de datos*
