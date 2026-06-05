# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-ServeBasics.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 tipo de saque (Total, 1S, 2S)  
**Filas:** 45.341 | **Columnas:** 12 | **Nulos:** 0 | **Duplicados:** 47  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

La columna `row` tiene tres valores: `Total`, `1` (primer saque) y `2` (segundo saque). Esto significa que por cada jugador y partido hay **3 filas**, no 1. La granularidad real es **jugador × tipo de saque**, no jugador × partido como se asumía inicialmente. Para métricas globales de saque usar `row = "Total"`. Para comparar rendimiento de 1S vs 2S usar `row = "1"` y `row = "2"` por separado.

---

## Columnas

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-Jesper_D...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador. | Mismo patrón que Overview — nombre completo, no entero. Filtrar Djokovic por nombre. |
| `row` | object | `Total`, `1`, `2` | Tipo de saque al que corresponde la fila. `Total` = ambos saques combinados, `1` = solo primer saque, `2` = solo segundo saque. | **Filtro obligatorio.** Para la mayoría de análisis usar `Total`. Para comparar caída de rendimiento 1S→2S usar las filas `1` y `2` por separado. |
| `pts` | int64 | `80`, `48`, `32`... | Total de puntos al saque en ese contexto (`row`). Cuando `row = "Total"` equivale a `serve_pts` de Overview. Denominador global. | Verificar: `pts(row=1) + pts(row=2) + dfs ≈ pts(row=Total)`. Las dobles faltas son puntos al saque que no generan ni primer ni segundo saque "dentro". |
| `pts_won` | int64 | `46`, `33`, `13`... | Puntos ganados al saque en ese contexto. `pts_won / pts` = `% puntos ganados al saque` (global o por tipo). | **KPI principal del saque.** Cuando `row = "Total"` equivale a `first_won + second_won` de Overview. |
| `aces` | int64 | `6`, `6`, `0`... | Aces en ese contexto. Un ace siempre ocurre en el primer saque, por lo que cuando `row = "2"` el valor es siempre `0`. | Permite confirmar que los aces están correctamente asignados al primer saque. |
| `unret` | int64 | `1`, `1`, `0`... | Saques no devueltos que **no son aces** — el restador intentó devolverlo pero no logró contacto limpio o no llegó a tiempo. `aces + unret` = total de saques directos (puntos ganados antes del primer golpe de fondo). | Diferencia importante con `aces`: el restador hizo algún movimiento pero no consiguió contacto. En análisis de presión sobre el resto, `unret` es más relevante que `aces` porque indica que el saque fue difícil de leer. |
| `forced_err` | int64 | `9`, `7`, `2`... | Errores forzados del restador en el primer o segundo golpe del rally inmediatamente tras el saque. El saque fue tan bueno que forzó un error inmediato. | Distinto de `unret` — aquí el restador sí devolvió pero el saque forzó un error en el golpe siguiente. Junto con `aces + unret` forma el bloque de **influencia directa del saque** en el punto. |
| `pts_won_lte_3_shots` | int64 | `22`, `17`, `5`... | Puntos ganados en 3 golpes o menos (saque + hasta 2 golpes más). Indica cuántos puntos se resolvieron muy rápido gracias al saque. | Columna exclusiva de ServeBasics — no existe en Overview. Útil para el Rally length chart: estos puntos caen en el bucket `1_3` de la tabla Rally. Permite cruzar ambas tablas para validar consistencia. |
| `wide` | int64 | `27`, `18`, `9`... | Saques dirigidos hacia el exterior de la pista (zona wide) en ese contexto. Agrega deuce_wide + ad_wide sin distinción de lado. | **Atención:** para el heatmap por lado hay que ir a ServeDirection. Aquí solo sirve para conocer la preferencia global por zona. |
| `body` | int64 | `5`, `1`, `4`... | Saques dirigidos al cuerpo del restador. Generalmente menos frecuentes pero efectivos para bloquear el revés. Agrega deuce_body + ad_body. | Ídem — sin distinción de lado deuce/ad. |
| `t` | int64 | `48`, `29`, `19`... | Saques dirigidos hacia la línea T (centro). La zona más frecuente en muchos jugadores porque genera ángulo y velocidad. Agrega deuce_t + ad_t. | Verificar: `wide + body + t` debería aproximarse a los saques que entraron dentro en ese contexto. |

---

## Relaciones entre columnas — verificaciones útiles

Estas identidades permiten detectar errores en los datos antes de modelar en Power BI:

```
# Consistencia de dirección vs total de saques dentro
wide + body + t  ≈  pts(row="1")    [primer saque dentro]
wide + body + t  ≈  pts(row="2")    [segundo saque dentro]

# Consistencia con Overview (cuando row="Total")
pts      ↔  serve_pts              (Overview)
pts_won  ↔  first_won + second_won (Overview)
aces     ↔  aces                   (Overview)
```

---

## Medidas DAX derivables de esta tabla

```dax
-- Usando row = "Total"
% Puntos ganados al saque    = DIVIDE( SUM(pts_won), SUM(pts) )

-- Comparando row = "1" vs row = "2"
% Ganados con 1S             = DIVIDE( SUM(pts_won) [row=1], SUM(pts) [row=1] )
% Ganados con 2S             = DIVIDE( SUM(pts_won) [row=2], SUM(pts) [row=2] )
Caída 1S → 2S                = [% Ganados con 1S] - [% Ganados con 2S]

-- Influencia directa del saque
% Puntos resueltos en ≤3     = DIVIDE( SUM(pts_won_lte_3_shots), SUM(pts) )
% Saques directos            = DIVIDE( SUM(aces) + SUM(unret), SUM(pts) )

-- Dirección global (sin distinción deuce/ad)
% Saques al T                = DIVIDE( SUM(t),    SUM(wide) + SUM(body) + SUM(t) )
% Saques wide                = DIVIDE( SUM(wide), SUM(wide) + SUM(body) + SUM(t) )
% Saques al cuerpo           = DIVIDE( SUM(body), SUM(wide) + SUM(body) + SUM(t) )
```

---

## Hallazgos importantes

**1. La separación 1S/2S se hace mediante la columna `row`, no mediante columnas separadas.**

**2. Tres columnas no documentadas con alto valor táctico.**  
`forced_err`, `pts_won_lte_3_shots` y `unret` aportan información sobre la influencia directa del saque en el desarrollo del punto — una dimensión táctica que Overview no captura.

**3. Las columnas `wide`, `body`, `t` son agregadas.**  
No distinguen lado deuce de lado ad. Para el heatmap de dirección de saque por lado de la pista hay que usar ServeDirection, que sí tiene `deuce_wide`, `deuce_t`, `ad_wide`, `ad_t`, etc.

**4. Los 47 duplicados coinciden en número con los de ServeDirection** (también 47). Probablemente corresponden a los mismos partidos duplicados. Investigar con `match_id` para confirmar y eliminar antes de modelar en Power BI.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador por nombre |
| `row` | Mantener — **filtrar por `Total`** para análisis global; usar `1`/`2` para comparativa 1S vs 2S |
| `pts` | Mantener — denominador global saque |
| `pts_won` | Mantener — **KPI principal** |
| `aces` | Mantener — KPI dominancia al saque |
| `unret` | Mantener — influencia directa del saque |
| `forced_err` | Mantener — influencia directa del saque |
| `pts_won_lte_3_shots` | Mantener — conexión con análisis de rally length |
| `wide` | Mantener como referencia global — para heatmap usar ServeDirection |
| `body` | Mantener como referencia global — ídem |
| `t` | Mantener como referencia global — ídem |

---

*Fuente: output de `dfs["serve_basics"].head(10)` y `dfs["serve_basics"].dtypes` ejecutados en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*