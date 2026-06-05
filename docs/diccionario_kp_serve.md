# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-KeyPointsServe.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 situación de marcador de presión (4 filas por jugador por partido)  
**Columnas:** 11 | **Nulos:** 0 | **Duplicados:** 84  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

Esta tabla cubre **únicamente los puntos de presión** jugados al saque — aquellos en los que el marcador tiene consecuencias directas sobre el juego (break point, game point o deuce). Los puntos con marcadores neutros (15-0, 30-15, 15-30, etc.) **no aparecen** en esta tabla.

La columna `row` tiene **4 valores mutuamente excluyentes y aditivos**:

| `row` | Situación de marcador | Descripción |
|---|---|---|
| `BP` | **Break Point** | Puntos en los que el **restador** tiene oportunidad de romper el saque. Marcadores: 30-40, 15-40, 0-40, y Ad-out (deuce en el que el restador tiene ventaja). El servidor está en desventaja máxima. |
| `GP` | **Game Point** | Puntos en los que el **servidor** tiene oportunidad de cerrar el juego. Marcadores: 40-0, 40-15, 40-30, y Ad-in (deuce en el que el servidor tiene ventaja). El servidor está en posición de dominio. |
| `Deuce` | **Deuce** | Puntos jugados con el marcador exactamente en 40-40. Situación de equilibrio máximo — el siguiente punto genera ventaja (Ad) para uno u otro. |
| `STotal` | **Serve Total** | Suma de `BP` + `GP` + `Deuce`. Total de puntos de presión jugados al saque. |

### Verificación de aditividad (verificada con datos reales):

```
pts(BP) + pts(GP) + pts(Deuce)               = pts(STotal)       ✓
pts_won(BP) + pts_won(GP) + pts_won(Deuce)   = pts_won(STotal)   ✓
[ídem para todas las columnas]
```

### Cobertura respecto al total de puntos al saque:

`STotal.pts` cubre aproximadamente el **40-50%** de los puntos al saque totales (verificado: 37 de ~80 puntos al saque de Jesper De Jong). El resto son puntos con marcadores neutros no incluidos en esta tabla.

---

## Tabla de columnas de datos

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador en rol de **servidor** en esos puntos de presión. | Para analizar cómo Djokovic gestiona la presión al saque, filtrar `player = "Novak Djokovic"`. |
| `row` | object | `BP`, `GP`, `Deuce`, `STotal` | Situación de marcador de los puntos incluidos en la fila. Ver tabla anterior. | **Filtro obligatorio.** Para el KPI de break points salvados usar `row = "BP"`. Para análisis de presión global usar `STotal`. |
| `pts` | int64 | `11`, `13`, `13`, `37`... | Total de puntos de presión al saque en esa situación de marcador. Denominador de todos los ratios de esta tabla. | No equivale a `serve_pts` de Overview — solo cubre los puntos de presión, no todos los puntos al saque. |
| `pts_won` | int64 | `8`, `8`, `8`, `24`... | Puntos ganados por el servidor en esa situación de presión. `pts_won / pts` = % puntos ganados al saque bajo presión. | **KPI principal de la tabla.** La comparativa `pts_won/pts` entre `BP`, `GP` y `Deuce` revela si el jugador rinde mejor en posición de dominio (GP) que bajo amenaza (BP). |
| `first_in` | int64 | `6`, `10`, `6`, `22`... | Primeros saques que entraron en zona válida en esa situación de marcador. `first_in / pts` = % de primer saque dentro bajo presión. | Indicador de solidez del saque bajo presión. Un `first_in` bajo en `BP` indica que el servidor arriesga más con el primer saque cuando está en desventaja — táctica de alto riesgo. |
| `aces` | int64 | `1`, `1`, `2`, `4`... | Aces realizados en esa situación de marcador. | Cuantifica cuántos puntos de presión se resuelven con el saque directo. Un `aces` alto en `BP` es el mejor indicador de saque bajo presión. |
| `svc_winners` | int64 | `1`, `1`, `2`, `4`... | Saques **no devueltos** por el restador que no son aces — el restador toca la bola pero no consigue devolverla (equivalente a `unret` de ServeBasics). | Junto con `aces`, forma el total de puntos resueltos sin rally. `aces + svc_winners` = saques directos en puntos de presión. |
| `rally_winners` | int64 | `2`, `2`, `2`, `6`... | Golpes **ganadores directos** del servidor durante el rally en esa situación de presión. | Distinto de `aces` y `svc_winners` — aquí el punto ya entró en rally y el servidor lo cierra con un winner propio. |
| `rally_forced` | int64 | `2`, `1`, `0`, `3`... | Errores **forzados del rival** en el rally provocados por la presión del servidor. El rival intentó devolver pero falló bajo presión del servidor. | Junto con `rally_winners`, forma los puntos ganados en rally por acción directa del servidor. El residual (`pts_won - aces - svc_winners - rally_winners - rally_forced`) corresponde a errores no forzados del rival — puntos ganados sin acción directa del servidor. |
| `unforced` | int64 | `1`, `2`, `2`, `5`... | Errores **no forzados cometidos por el propio servidor** durante el rally en puntos de presión. El servidor falla sin presión aparente del rival. | Principal fuente de puntos perdidos bajo presión junto con `dfs`. Un `unforced` alto en `BP` indica fragilidad mental bajo amenaza de break. |
| `dfs` | int64 | `1`, `1`, `0`, `2`... | **Dobles faltas** cometidas en esa situación de presión. Pérdida directa del punto por fallar el segundo saque. | Tácticamente crítico en `BP`: una doble falta en break point es el peor resultado posible para el servidor. El residual de puntos perdidos (`(pts - pts_won) - unforced - dfs`) corresponde a winners del rival y errores forzados del servidor no registrados explícitamente. |

---

## Estructura aritmética y verificaciones cruzadas

```
# Identidad garantizada
pts_won + (pts - pts_won)  =  pts                                      ✓ (trivial)
BP.pts + GP.pts + Deuce.pts  =  STotal.pts                             ✓ (verificado)

# Identidad aproximada de puntos ganados (con residual)
pts_won  ≈  aces + svc_winners + rally_winners + rally_forced  +  UFE_rival
# UFE_rival = errores no forzados del rival (no están en esta tabla)
# Residual observado: ~7 puntos en STotal sin categoría explícita de victoria

# Identidad aproximada de puntos perdidos (con residual)
pts - pts_won  ≈  unforced + dfs  +  winners_rival + forced_propio
# winners_rival y forced_propio no están en esta tabla
# Residual observado: ~6 puntos en STotal sin categoría explícita de derrota

# Correspondencia con Overview (mismo partido, mismo jugador)
STotal.pts        <  serve_pts (Overview)   [kp_serve cubre ~40-50% del total]
BP.pts_won / BP.pts  ≈  bp_saved / bk_pts (Overview)  [aproximación — Overview usa juegos, no puntos]

# Nota importante: Overview registra bp_saved/bk_pts a nivel de JUEGOS (cuántos juegos
# de break point se salvaron), mientras kp_serve registra PUNTOS individuales de break.
# No son equivalentes directos — un juego de break puede contener múltiples puntos de BP.
```

---

## Medidas DAX derivables de esta tabla

```dax
-- KPI principal: rendimiento al saque bajo presión
% Ganados en BP     = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "BP"]
% Ganados en GP     = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "GP"]
% Ganados en Deuce  = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "Deuce"]
% Ganados presión   = DIVIDE( SUM(pts_won), SUM(pts) )  -- [row = "STotal"]

-- Diferencial dominio vs amenaza
Δ GP vs BP          = [% Ganados en GP] - [% Ganados en BP]
-- Positivo (esperado): el servidor rinde mejor cuando domina
-- Un Δ bajo indica solidez mental bajo presión — característica de Djokovic

-- Solidez del saque en puntos clave
% 1S dentro en BP   = DIVIDE( SUM(first_in), SUM(pts) )  -- [row = "BP"]
% 1S dentro en GP   = DIVIDE( SUM(first_in), SUM(pts) )  -- [row = "GP"]
Δ 1S BP vs GP       = [% 1S dentro en BP] - [% 1S dentro en GP]
-- Negativo indica que el servidor arriesga más con el 1S cuando está en apuros

-- Saques directos en puntos de presión
% Saques directos BP = DIVIDE( SUM(aces) + SUM(svc_winners), SUM(pts) )  -- [row = "BP"]

-- Perfil de finalización en puntos de presión
% Winners rally BP   = DIVIDE( SUM(rally_winners), SUM(pts) )   -- [row = "BP"]
% Forced rival BP    = DIVIDE( SUM(rally_forced),  SUM(pts) )   -- [row = "BP"]
% UFE propios BP     = DIVIDE( SUM(unforced),       SUM(pts) )  -- [row = "BP"]
% DF en BP           = DIVIDE( SUM(dfs),            SUM(pts) )  -- [row = "BP"]

-- Comparativa presión vs partido completo (requiere cruzar con Overview)
Δ % ganados saque total vs presión =
    DIVIDE( SUM(first_won)+SUM(second_won), SUM(serve_pts) ) [Overview]
    - DIVIDE( SUM(pts_won), SUM(pts) ) [kp_serve, row="STotal"]
-- Negativo indica caída de rendimiento bajo presión — indicador de fragilidad
```

---

## Hallazgos importantes

**1. `BP`, `GP` y `Deuce` son mutuamente excluyentes y aditivos — `STotal` es su suma exacta.**
A diferencia de las tablas de resto (donde los grupos se solapan), aquí los tres grupos cubren conjuntos de puntos disjuntos. Esto permite sumar entre grupos sin riesgo de doble conteo.

**2. Esta tabla NO cubre todos los puntos al saque.**
`STotal.pts` representa aproximadamente el 40-50% de `serve_pts` de Overview. Los puntos con marcadores neutros (15-0, 0-15, 30-15, etc.) no están incluidos. Es una tabla de **situaciones de presión**, no de rendimiento global al saque.

**3. `bp_saved/bk_pts` de Overview y `pts_won/pts` de `row="BP"` no son equivalentes.**
Overview registra cuántos **juegos** de break point se salvaron (nivel de juego). `kp_serve` registra cuántos **puntos** individuales de break point se ganaron. Un juego puede contener varios puntos de BP consecutivos (en situación de deuce). Para análisis táctico fino, `kp_serve` es más preciso.

**4. El residual de puntos sin categoría (~6-7 por partido) es estructural.**
Las columnas de ganancia (`aces`, `svc_winners`, `rally_winners`, `rally_forced`) no cubren los errores no forzados del rival. Las columnas de pérdida (`unforced`, `dfs`) no cubren los winners del rival ni los errores forzados del propio servidor. Este residual es consistente y esperado — no indica problema de calidad de datos.

**5. Esta tabla es la fuente para responder la pregunta 5 del E2.**
"¿Qué cambia en puntos de break, tie-break, deuce o puntos de máxima presión?" se responde directamente comparando `% pts_won` entre `BP`, `GP` y `Deuce`. Djokovic históricamente destaca por mantener un `% ganados en BP` alto — esta tabla lo cuantifica con evidencia.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador |
| `row` | Mantener — `BP` para break points, `STotal` para análisis global de presión |
| `pts` | Mantener — denominador de puntos de presión |
| `pts_won` | Mantener — **KPI principal** |
| `first_in` | Mantener — solidez del saque bajo presión |
| `aces` | Mantener — resolución directa con el saque |
| `svc_winners` | Mantener — saques no devueltos bajo presión |
| `rally_winners` | Mantener — agresividad en rally bajo presión |
| `rally_forced` | Mantener — presión ejercida sobre el rival |
| `unforced` | Mantener — fragilidad bajo presión |
| `dfs` | Mantener — **KPI crítico en BP** — dobles faltas en el peor momento |

---

*Fuente: output de `dfs["kp_serve"].head(10)` ejecutado en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*