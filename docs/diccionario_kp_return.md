# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-KeyPointsReturn.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 situación de marcador de presión (4 filas por jugador por partido)  
**Columnas:** 7 | **Nulos:** 0 | **Duplicados:** 84  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

`kp_return` es el **espejo exacto de `kp_serve`** desde la perspectiva del restador. Describe los mismos puntos de presión que `kp_serve` pero contados desde el punto de vista del jugador que está devolviendo el saque. Tienen menos columnas que `kp_serve` porque las métricas de saque (aces, svc_winners, first_in, dfs) no aplican al restador.

La columna `row` tiene **4 valores mutuamente excluyentes y aditivos**, simétricos a los de `kp_serve`:

| `row` | Situación de marcador | Equivalente en `kp_serve` | Descripción |
|---|---|---|---|
| `BPO` | **Break Point Opportunity** | `BP` | Puntos en los que el **restador tiene oportunidad de break**. El servidor está bajo amenaza. Marcadores: 30-40, 15-40, 0-40, Ad-out. |
| `GPF` | **Game Point Faced** | `GP` | Puntos en los que el **restador enfrenta un game point del servidor**. El servidor está a punto de cerrar el juego. Marcadores: 40-0, 40-15, 40-30, Ad-in. |
| `DeuceR` | **Deuce (perspectiva Restador)** | `Deuce` | Puntos jugados con el marcador en 40-40. Mismos puntos que `Deuce` de `kp_serve`, perspectiva opuesta. |
| `RTotal` | **Return Total** | `STotal` | Suma de `BPO` + `GPF` + `DeuceR`. Total de puntos de presión al resto. |

### Simetría verificada (partido Roland Garros Q3):

```
kp_serve  Jesper (servidor):  BP.pts=11,  GP.pts=13,  Deuce.pts=13,  STotal.pts=37
kp_return Zheng  (restador):  BPO.pts=11, GPF.pts=13, DeuceR.pts=13, RTotal.pts=37  ✓

kp_serve  Zheng  (servidor):  STotal.pts=24
kp_return Jesper (restador):  RTotal.pts=24  ✓

pts_won(kp_serve BP, Jesper) + pts_won(kp_return BPO, Zheng) = pts
  8 + 3 = 11  ✓  (los puntos de un partido son cero-suma)
```

### Verificación de aditividad:

```
pts(BPO) + pts(GPF) + pts(DeuceR)               = pts(RTotal)       ✓
pts_won(BPO) + pts_won(GPF) + pts_won(DeuceR)   = pts_won(RTotal)   ✓
[ídem para todas las columnas]
```

---

## Tabla de columnas de datos

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador en rol de **restador** en esos puntos de presión. | Para analizar cómo Djokovic convierte oportunidades de break, filtrar `player = "Novak Djokovic"`. |
| `row` | object | `BPO`, `GPF`, `DeuceR`, `RTotal` | Situación de marcador de los puntos incluidos en la fila. Ver tabla anterior. | **Filtro obligatorio.** Para el KPI de break points convertidos usar `row = "BPO"`. Para análisis global de presión al resto usar `RTotal`. |
| `pts` | int64 | `3`, `14`, `7`, `24`... | Total de puntos de presión al resto en esa situación de marcador. Denominador de todos los ratios. | Coincide exactamente con `pts` del grupo simétrico en `kp_serve` del jugador rival (mismo partido). |
| `pts_won` | int64 | `0`, `4`, `0`, `4`... | Puntos ganados por el restador en esa situación de presión. `pts_won / pts` = % puntos ganados al resto bajo presión. | **KPI principal de la tabla.** `pts_won(BPO) / pts(BPO)` = tasa de conversión de break points — la métrica más importante para evaluar al restador bajo presión. Para Djokovic, históricamente el mejor restador del circuito, este valor en `BPO` es el indicador táctico más revelador de toda la tabla. |
| `rally_winners` | int64 | `0`, `1`, `2`, `2`... | Golpes **ganadores directos del restador** durante el rally en esa situación de presión. El servidor no pudo devolver el golpe del restador. | En el contexto del resto, los rally winners son los golpes más decisivos del restador. Excluye winners de saque (que no existen para el restador) y excluye el resto ganador directo — este tipo de golpe es tan infrecuente que no tiene columna propia. |
| `rally_forced` | int64 | `0`, `1`, `1`, `1`... | Errores **forzados del servidor** en el rally provocados por la presión del restador. El servidor intentó devolver bajo presión del restador y falló. | Junto con `rally_winners`, forma los puntos ganados por acción directa del restador en el rally. El residual `pts_won - rally_winners - rally_forced` corresponde principalmente a errores no forzados del servidor (UFE del rival) y dobles faltas del servidor — puntos ganados sin acción directa del restador. |
| `unforced` | int64 | `1`, `1`, `1`, `3`... | Errores **no forzados cometidos por el propio restador** durante el rally en puntos de presión. El restador falla sin presión aparente del servidor. | Principal fuente de puntos perdidos al resto bajo presión, junto con el residual de saques directos del servidor (aces, svc_winners) que no tienen columna explícita en esta tabla. Un `unforced` alto en `BPO` indica que el restador se pone nervioso precisamente cuando tiene la oportunidad de romper. |

---

## Estructura aritmética y verificaciones cruzadas

```
# Identidad garantizada
pts_won + (pts - pts_won)  =  pts                                        ✓

# Aditividad entre grupos
BPO.pts + GPF.pts + DeuceR.pts  =  RTotal.pts                           ✓ (verificado)

# Identidad aproximada de puntos ganados (con residual)
pts_won  ≈  rally_winners + rally_forced  +  residual_ganado
# residual_ganado = UFE del servidor + DF del servidor  (no tienen columna en kp_return)
# Verificado: residual_ganado(RTotal) ≈ 7 puntos por partido

# Identidad aproximada de puntos perdidos (con residual)
pts - pts_won  ≈  unforced  +  residual_perdido
# residual_perdido = aces del servidor + svc_winners + rally_winners del servidor
#                   + errores forzados del restador
# Estos NO tienen columna en kp_return — es la asimetría estructural con kp_serve
# Verificado: residual_perdido(RTotal) ≈ 17 puntos — mayor que en kp_serve
# porque el saque (aces, unret) genera más puntos sin categoría en la perspectiva del restador

# Simetría con kp_serve (mismo partido, jugadores opuestos)
pts(kp_return BPO, jugador A)      =  pts(kp_serve BP, jugador B)         ✓
pts_won(kp_return BPO, A) +
pts_won(kp_serve BP, B)            =  pts(BPO)                            ✓ (zero-suma)

# Cruce verificado: unforced(kp_return BPO, Zheng=2)
#                  ≈ unforced(kp_serve BP, Jesper=1) + dfs(kp_serve BP, Jesper=1) = 2  ✓
# Los errores no forzados del restador coinciden con UFE+DF del servidor en el grupo simétrico

# Correspondencia con Overview (mismo partido, mismo jugador)
RTotal.pts          <  return_pts (Overview)   [kp_return cubre ~40-50% del total al resto]
BPO.pts_won / BPO.pts  ≈  % puntos de break convertidos a nivel de PUNTO
                           [distinto de bp_converted/bp_opp a nivel de JUEGO en Overview]
```

---

## Asimetría estructural con `kp_serve`

`kp_return` tiene **4 columnas menos** que `kp_serve`. Las ausentes corresponden a métricas de saque que no aplican al restador:

| Columna presente en `kp_serve` | Ausente en `kp_return` | Motivo |
|---|---|---|
| `first_in` | ✗ | El restador no saca — no hay primer saque que registrar |
| `aces` | ✗ | Los aces los comete el servidor, no el restador |
| `svc_winners` | ✗ | Ídem — saques no devueltos son del servidor |
| `dfs` | ✗ | Las dobles faltas las comete el servidor |

Como consecuencia, el **residual de puntos perdidos** es estructuralmente mayor en `kp_return` (~17) que en `kp_serve` (~6): en `kp_serve` el servidor tiene columnas para casi todas sus formas de ganar (aces, svc_winners, rally_winners, rally_forced); en `kp_return` el restador carece de columnas para los puntos perdidos por el saque del rival (aces, svc_winners del servidor son puntos perdidos para el restador sin categoría explícita).

---

## Medidas DAX derivables de esta tabla

```dax
-- KPI principal: conversión de break points (pregunta clave del E2)
% Conversión BP         = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "BPO"]

-- Resistencia bajo presión adversa
% Ganados GPF           = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "GPF"]
-- Un % alto en GPF indica que el restador salva puntos de game del rival

-- Rendimiento en Deuce al resto
% Ganados DeuceR        = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "DeuceR"]

-- Presión global al resto en momentos clave
% Ganados presión resto = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "RTotal"]

-- Diferencial BPO vs GPF (oportunidad vs adversidad)
Δ BPO vs GPF            = [% Conversión BP] - [% Ganados GPF]
-- Positivo indica que el restador rinde MEJOR cuando tiene oportunidad de break
-- que cuando el servidor está a punto de cerrar — perfil de restador mental fuerte

-- Perfil de cómo gana puntos el restador bajo presión
% Rally winners al resto BPO  = DIVIDE( SUM(rally_winners), SUM(pts) )  -- [row = "BPO"]
% Rally forced al resto BPO   = DIVIDE( SUM(rally_forced),  SUM(pts) )  -- [row = "BPO"]
% UFE propios al resto BPO    = DIVIDE( SUM(unforced),      SUM(pts) )  -- [row = "BPO"]

-- Comparativa presión vs partido completo (requiere cruzar con Overview)
Δ % ganados resto total vs presión =
    DIVIDE( SUM(return_pts_won), SUM(return_pts) ) [Overview]
    - DIVIDE( SUM(pts_won), SUM(pts) )             [kp_return, row = "RTotal"]
-- Negativo indica caída de rendimiento al resto bajo presión
-- Positivo (raro) indica que el restador se crece bajo presión — característica de Djokovic

-- Cruce con kp_serve para análisis completo de presión
-- (requiere join por match_id y rol de jugador)
Presión combinada BP+BPO =
    -- % ganados en BP (kp_serve) + % conversión BPO (kp_return) para el mismo jugador
    -- mide la solidez total en los puntos más decisivos del partido
```

---

## Hallazgos importantes

**1. `BPO.pts_won / BPO.pts` es el KPI táctico más importante de toda la tabla para Djokovic.**
La conversión de break points es la métrica que más diferencia a los mejores restadores del circuito. Djokovic históricamente convierte una proporción alta de sus oportunidades de break — esta tabla permite cuantificarlo con evidencia y compararlo por superficie, rival y periodo temporal.

**2. `kp_return` y `kp_serve` describen los mismos puntos desde perspectivas opuestas.**
Son tablas complementarias, no redundantes. `kp_serve` revela si Djokovic salva sus break points; `kp_return` revela si Djokovic convierte los del rival. Juntas construyen el perfil completo de presión: el mejor servidor bajo presión minimiza su `BP` en `kp_serve`; el mejor restador bajo presión maximiza su `BPO` en `kp_return`.

**3. El residual de puntos perdidos es estructuralmente mayor que en `kp_serve`.**
~17 puntos perdidos sin categoría en `RTotal` (vs ~6 en `kp_serve`). No es un problema de calidad de datos — es consecuencia de que los aces y saques no devueltos del servidor no tienen columna en `kp_return`. Este residual es esperado y predecible.

**4. `unforced` en `BPO` es el indicador de fragilidad mental más directo.**
Un restador que comete errores no forzados precisamente cuando tiene break point indica que la presión le afecta psicológicamente. Para Djokovic, históricamente un jugador de gran solidez mental, se espera un `unforced` muy bajo en `BPO`.

**5. `GPF` mide la resistencia defensiva del restador.**
Cuando el servidor está a punto de cerrar el juego (40-0, 40-15, 40-30, Ad-in), el restador está en su momento de mayor adversidad. Un `pts_won/pts` alto en `GPF` indica capacidad de revertir situaciones aparentemente perdidas — otra fortaleza histórica de Djokovic.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador |
| `row` | Mantener — `BPO` para conversión de break points; `RTotal` para análisis global de presión al resto |
| `pts` | Mantener — denominador de puntos de presión al resto |
| `pts_won` | Mantener — **KPI principal: conversión de break points** |
| `rally_winners` | Mantener — agresividad del restador bajo presión |
| `rally_forced` | Mantener — presión ejercida sobre el servidor |
| `unforced` | Mantener — **indicador de fragilidad mental bajo presión** |

---

## Nota de uso conjunto con `kp_serve`

Para el modelo Power BI se recomienda crear una tabla unificada de presión que combine `kp_serve` y `kp_return` para Djokovic:

```
Djokovic al saque bajo presión   →  kp_serve  filtrado por player = "Novak Djokovic"
Djokovic al resto bajo presión   →  kp_return filtrado por player = "Novak Djokovic"
```

Esta separación es suficiente para responder las preguntas del E2 y E3 sin necesidad de cruzar las tablas entre sí. El cruce (verificar que `pts_won` de ambas suma `pts`) es útil solo para validación de calidad de datos.

---

*Fuente: output de `dfs["kp_return"].head(16)` ejecutado en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*