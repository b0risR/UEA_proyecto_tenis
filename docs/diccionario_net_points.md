# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-NetPoints.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 subconjunto de puntos en red (4 filas por jugador por partido)  
**Columnas:** 10 | **Nulos:** 0 | **Duplicados:** 74  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

La columna `row` tiene **4 valores** que representan subconjuntos anidados de los mismos puntos, no categorías mutuamente excluyentes. Cada fila filtra los puntos en red desde una perspectiva distinta:

| `row` | Descripción | Subconjunto de |
|---|---|---|
| `NetPoints` | **Todos** los puntos en los que el jugador subió a la red, independientemente de cómo llegó o si llegó a golpear. | — (fila más amplia) |
| `Approach` | Puntos en los que el jugador subió a la red mediante un **golpe de aproximación** explícito. | `NetPoints` |
| `NetPointsRallies` | Puntos en red en los que el jugador **realizó al menos un golpe** desde la posición de red (volea, smash, etc.). | `NetPoints` |
| `ApproachRallies` | Puntos en los que hubo golpe de aproximación **y** el jugador golpeó en red. | `Approach` y `NetPointsRallies` |

"Subió a la red" significa que el jugador abandonó la línea de fondo y avanzó hacia la red durante el transcurso del punto, adoptando una posición en la zona delantera de la pista (aproximadamente entre la línea de servicio y la red). No implica que haya llegado a golpear — solo que se desplazó hacia esa zona.

"Golpeó en red" significa que el jugador, ya en posición delantera, realizó efectivamente un golpe desde esa posición.

### Jerarquía verificada con datos reales (Jesper De Jong, Roland Garros Q3):

```
NetPoints(16) >= Approach(13) >= ApproachRallies(10)
NetPoints(16) >= NetPointsRallies(13) >= ApproachRallies(10)
```

La diferencia entre filas revela información táctica adicional:

```
NetPoints - Approach       = 3  →  puntos en red sin approach explícito
                                   (net rush al saque, subidas espontáneas)
NetPoints - NetPointsRallies = 3  →  puntos en red donde el jugador NO llegó a golpear
                                   (el punto se resolvió antes: DF del rival, etc.)
Approach - ApproachRallies  = 3  →  approaches donde el punto terminó sin golpe en red
                                   (el rival pasó antes del primer contacto del net player)
```

> **Advertencia:** En algunos partidos `Approach.net_pts` puede coincidir numéricamente con `NetPointsRallies.net_pts`, pero **no son el mismo conjunto de puntos**. `Approach` incluye approaches sin golpe en red; `NetPointsRallies` incluye subidas sin approach donde el jugador sí golpeó.

---

## Tabla de columnas de datos

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador que **subió a la red** — la perspectiva es siempre la del jugador en red, no la del pasador. | Para analizar el juego de red de Djokovic, filtrar `player = "Novak Djokovic"`. |
| `row` | object | `NetPoints`, `Approach`, `NetPointsRallies`, `ApproachRallies` | Subconjunto de puntos en red que describe la fila. Ver tabla y jerarquía anteriores. | **Filtro obligatorio.** Para el KPI principal de red usar `NetPoints`. Para análisis de efectividad del approach usar `Approach` o `ApproachRallies`. |
| `net_pts` | int64 | `16`, `13`, `11`... | Total de puntos en red en ese subconjunto. Denominador de todos los ratios de esta tabla. | No confundir con `pts` de otras tablas — aquí solo cuenta puntos donde el jugador estuvo en posición de red. |
| `pts_won` | int64 | `13`, `11`, `9`... | Puntos ganados por el jugador en red. `pts_won / net_pts` = % puntos ganados en red. | **KPI principal de la tabla.** Medida obligatoria del E2. Djokovic históricamente tiene un % de red bajo para su nivel, ya que su juego es de fondo. |
| `net_winner` | int64 | `8`, `6`, `4`... | Golpes **ganadores directos** realizados desde la posición de red (voleas y smashes que el rival no pudo devolver). | Principal fuente de puntos ganados en red junto con `induced_forced`. Excluye aces — los aces no son golpes de red. |
| `induced_forced` | int64 | `4`, `4`, `2`... | Errores **forzados del rival** al intentar el passing shot, provocados por la presión del jugador en red. El rival intentó pasar pero falló bajo presión. | Distinto de `net_winner`: aquí el punto lo gana el net player por error del rival, no por winner propio. Tácticamente equivale a un winner en términos de efectividad. |
| `net_unforced` | int64 | `1`, `0`, `1`... | Errores **no forzados cometidos por el propio jugador** en red — voleas o smashes fallados sin presión del rival. | Contribuye a los puntos **perdidos** en red. Un `net_unforced` alto indica falta de técnica o concentración en red. |
| `passed_at_net` | int64 | `1`, `1`, `1`... | Veces que el **rival pasó con éxito** al jugador en red — passing shots exitosos del rival. | Principal fuente de puntos perdidos en red. La diferencia `net_pts - pts_won - net_unforced - passed_at_net` da el residual de puntos sin categoría explícita (ver hallazgo 3). |
| `passing_shot_induced_forced` | int64 | `0`, `0`, `0`... | Errores **forzados del rival al intentar el passing shot** por una causa distinta de `induced_forced` — categoría complementaria que captura presión indirecta del net player. | Frecuencia muy baja o nula en la mayoría de partidos. Explica parte del residual en `pts_won`. Puede descartarse del modelo si es consistentemente 0. |
| `total_shots` | int64 | `80`, `62`, `70`... | Total de golpes acumulados en todos los puntos del subconjunto. `total_shots / net_pts` = promedio de golpes por punto en red. | Permite estimar si los puntos en red de Djokovic son cortos (red agresiva) o largos (red defensiva). No coincide con `total_shots` de ReturnOutcomes — aquí la perspectiva es el jugador en red, no el restador. |

---

## Estructura aritmética y verificaciones cruzadas

```
# Identidad de puntos ganados (aproximada — residual de ~1 punto sin categoría)
pts_won  ≈  net_winner + induced_forced + passing_shot_induced_forced  +  residual
pts_perdidos  =  net_pts - pts_won
pts_perdidos  ≈  passed_at_net + net_unforced  +  residual

# El residual (~1 punto por partido) corresponde a puntos ganados/perdidos
# sin categoría de finalización registrada (DF del rival en red, puntos de penalización, etc.)
# Verificado con datos reales: residual = 1 en todas las filas de Jesper De Jong

# Jerarquía de subconjuntos
net_pts(NetPoints)  >=  net_pts(Approach)        >=  net_pts(ApproachRallies)
net_pts(NetPoints)  >=  net_pts(NetPointsRallies) >=  net_pts(ApproachRallies)

# Correspondencia con Overview (mismo partido, mismo jugador)
# No hay columna directa en Overview equivalente a net_pts
# pts_won(NetPoints) es un subconjunto de (first_won + second_won) de Overview
```

---

## Medidas DAX derivables de esta tabla

```dax
-- KPI principal (visualización obligatoria del E2)
% Puntos ganados en red  = DIVIDE( SUM(pts_won), SUM(net_pts) )  -- [row = "NetPoints"]

-- Efectividad del approach
% Puntos ganados con approach = DIVIDE( SUM(pts_won), SUM(net_pts) ) -- [row = "Approach"]
% Approach con rally en red   = DIVIDE( SUM(net_pts) [row="ApproachRallies"],
                                         SUM(net_pts) [row="Approach"] )

-- Perfil de cómo se ganan/pierden los puntos en red
% Winners en red        = DIVIDE( SUM(net_winner),      SUM(net_pts) )  -- [row = "NetPoints"]
% Errores forzados ind. = DIVIDE( SUM(induced_forced),  SUM(net_pts) )
% Pasado en red         = DIVIDE( SUM(passed_at_net),   SUM(net_pts) )
% UFE en red            = DIVIDE( SUM(net_unforced),    SUM(net_pts) )

-- Frecuencia de subida a red (requiere cruzar con Overview)
% Puntos jugados en red = DIVIDE( SUM(net_pts) [row="NetPoints"],
                                   SUM(serve_pts) + SUM(return_pts) )  -- denominador de Overview

-- Rally length en red
Shots por punto en red  = DIVIDE( SUM(total_shots), SUM(net_pts) )  -- [row = "NetPointsRallies"]

-- Comparativa approach vs subida espontánea
net_pts sin approach    = SUM(net_pts) [row="NetPoints"] - SUM(net_pts) [row="Approach"]
```

---

## Hallazgos importantes

**1. Las 4 filas son subconjuntos anidados, no categorías excluyentes.**
`ApproachRallies` es el subconjunto más restrictivo (approach + rally en red). `NetPoints` es el más amplio. Nunca sumar filas entre sí — representan distintas vistas del mismo conjunto de puntos.

**2. La diferencia `NetPoints - Approach` mide las subidas a red sin approach.**
En el ejemplo, 3 puntos en red no fueron precedidos por un golpe de aproximación explícito. Estos corresponden a net rush al saque (server-and-volley), subidas espontáneas tras saque corto del rival, o situaciones donde el jugador ya estaba en red desde el punto anterior.

**3. Hay un residual de ~1 punto por partido sin categoría de finalización.**
La suma `net_winner + induced_forced + passed_at_net + net_unforced` no alcanza `net_pts`. El residual (~1 punto) corresponde a puntos en red terminados por causas no capturadas en estas columnas: doble falta del rival mientras el jugador estaba en red, puntos de penalización, o `passing_shot_induced_forced` (que en muchos partidos es 0 pero puede cubrir parte del residual en otros).

**4. `passing_shot_induced_forced` es casi siempre 0 y puede descartarse.**
En todos los partidos del `head(10)` esta columna vale 0. Es una categoría muy específica (error forzado del rival al intentar el passing, pero causado por presión indirecta, no por el golpe directo del net player) que raramente se registra.

**5. Esta tabla es la fuente del `% puntos ganados en red` exigido en el E2.**
Djokovic es históricamente un jugador de fondo con poca presencia en red — esta tabla permitirá cuantificar exactamente cuánto sube y con qué efectividad, y detectar si en ciertas superficies (hierba) o contra ciertos perfiles de rival aumenta su presencia en red.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador |
| `row` | Mantener — usar `NetPoints` para KPI principal; `Approach` para análisis de approach |
| `net_pts` | Mantener — denominador global |
| `pts_won` | Mantener — **KPI principal** |
| `net_winner` | Mantener — agresividad en red |
| `induced_forced` | Mantener — presión ejercida desde red |
| `net_unforced` | Mantener — consistencia en red |
| `passed_at_net` | Mantener — vulnerabilidad al passing |
| `passing_shot_induced_forced` | **Descartar** — frecuencia nula, información cubierta por `induced_forced` |
| `total_shots` | Mantener — rally length en red |

---

*Fuente: output de `dfs["net_points"].head(10)` ejecutado en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*