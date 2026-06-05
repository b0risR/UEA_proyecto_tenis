# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-Overview.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 set (incluyendo fila `Total`)  
**Filas:** 56.850 | **Columnas:** 20 | **Nulos:** 0 | **Duplicados:** 56  
**Clave de join:** `match_id` + `player` + `set`

---

## Nota estructural crítica

La columna `set` subdivide cada partido. Por cada partido y jugador hay múltiples filas: una fila `Total` (partido completo) y una fila por cada set jugado (`1`, `2`, `3`...). La granularidad real es **jugador × set**. Al filtrar en Power BI siempre habrá que decidir si se usa `Total` o el desglose por set.

---

## Columnas

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-Jesper_D...` | Clave de join con `matches`. | Igual que en todas las tablas de estadísticas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador al que pertenece la fila. | **Atención:** es nombre completo, no entero `1` o `2`. Para filtrar Djokovic se filtra directamente por nombre. Verificar consistencia ortográfica en toda la tabla. |
| `set` | object | `Total`, `1`, `2`, `3`... | Indica si la fila es el agregado del partido completo (`Total`) o un set específico. | **Filtro obligatorio** al usar esta tabla. Para el informe táctico general usar `set = "Total"`. Para análisis de evolución dentro del partido usar los sets individuales. |
| `serve_pts` | int64 | `80`, `61`, `52`... | Total de puntos jugados al saque por este jugador en este set/partido. Denominador base para `% puntos ganados al saque`. | `serve_pts` de Player 1 = `return_pts` de Player 2 en el mismo partido y set, y viceversa. |
| `aces` | int64 | `6`, `3`, `2`, `0`... | Número de aces realizados. Saques que el restador no tocó y que entraron en zona válida. | Contribuye a los puntos ganados al saque pero no aparece desglosado dentro de `first_won` o `second_won` en esta tabla — para eso está ServeBasics. |
| `dfs` | int64 | `4`, `2`, `1`, `0`... | Dobles faltas. Pérdida directa de punto al fallar dos saques consecutivos. | Nótese el nombre `dfs` (plural). La columna equivalente en ServeBasics también se llama `dfs`. |
| `first_in` | int64 | `48`, `34`, `32`... | Primeros saques que entraron en zona válida. Numerador para `% 1er saque dentro` = `first_in / serve_pts`. | Incluye tanto los puntos ganados como los perdidos con primer saque dentro. |
| `first_won` | int64 | `33`, `27`, `22`... | Puntos ganados cuando el primer saque entró. `first_won / first_in` = `% puntos ganados con 1S`. | KPI táctico directo. Un ratio alto indica que el primer saque genera ventaja inmediata. |
| `second_in` | int64 | `32`, `27`, `20`... | Segundos saques que entraron (sin contar dobles faltas). | `second_in + dfs` ≈ `serve_pts - first_in` (los puntos donde el primer saque fue falta). |
| `second_won` | int64 | `13`, `16`, `8`... | Puntos ganados cuando el segundo saque entró. `second_won / second_in` = `% puntos ganados con 2S`. | La caída entre `first_won/first_in` y `second_won/second_in` mide la **vulnerabilidad del segundo saque** — una de las preguntas clave del ejercicio. |
| `bk_pts` | int64 | `11`, `3`, `7`... | Break points afrontados por el servidor en ese set/partido. Número de veces que el restador tuvo oportunidad de romper el saque. | Nombre poco intuitivo (`bk_pts` = break points faced). No confundir con break points convertidos. |
| `bp_saved` | int64 | `8`, `3`, `6`... | Break points salvados por el servidor. `bp_saved / bk_pts` = `% BP salvados`. | KPI de presión al saque. Djokovic históricamente es uno de los mejores salvando break points. |
| `return_pts` | int64 | `61`, `80`, `34`... | Total de puntos jugados al resto. Es el espejo de `serve_pts` del rival en ese partido y set. | `return_pts` (Player 1) = `serve_pts` (Player 2) en el mismo match y set. |
| `return_pts_won` | int64 | `18`, `34`, `9`... | Puntos ganados al resto. `return_pts_won / return_pts` = `% puntos ganados al resto`. | KPI táctico fundamental. Junto con `serve_pts_won / serve_pts` forma los dos grandes indicadores de rendimiento por partido. |
| `winners` | int64 | `25`, `28`, `18`... | Golpes ganadores directos totales, **excluyendo aces y saques directos**. Solo golpes de rally que terminaron el punto sin que el rival tocara la bola. | La suma `winners + aces` da el total de golpes ganadores incluyendo saque. |
| `winners_fh` | int64 | `13`, `23`, `8`... | Winners de forehand (derecha). Subconjunto de `winners`. | Permite evaluar si el jugador es más ganador por derecha o por revés. |
| `winners_bh` | int64 | `5`, `2`, `3`... | Winners de backhand (revés). Subconjunto de `winners`. `winners_fh + winners_bh` = `winners`. | La proporción fh/bh es un indicador del perfil táctico del jugador. |
| `unforced` | int64 | `25`, `21`, `13`... | Errores no forzados totales. Golpes fallados sin presión aparente del rival. | **Indicador principal de consistencia.** El ratio `unforced / (winners + unforced)` alto indica jugador agresivo pero poco consistente. |
| `unforced_fh` | int64 | `15`, `11`, `7`... | Errores no forzados de forehand. Subconjunto de `unforced`. | Identifica el lado más propenso al error. Útil para decidir hacia dónde construir el punto contra este jugador. |
| `unforced_bh` | int64 | `6`, `8`, `4`... | Errores no forzados de backhand. Subconjunto de `unforced`. `unforced_fh + unforced_bh` = `unforced`. | Ídem pero por revés. |

---

## Medidas DAX derivables directamente de esta tabla

Estas son las medidas del ejercicio que se pueden calcular **solo con Overview + matches**:

```
% Puntos ganados al saque  = DIVIDE( SUM(first_won) + SUM(second_won), SUM(serve_pts) )
% Puntos ganados al resto  = DIVIDE( SUM(return_pts_won), SUM(return_pts) )
% 1er saque dentro         = DIVIDE( SUM(first_in),   SUM(serve_pts) )
% Ganados con 1S           = DIVIDE( SUM(first_won),  SUM(first_in) )
% Ganados con 2S           = DIVIDE( SUM(second_won), SUM(second_in) )
% BP salvados              = DIVIDE( SUM(bp_saved),   SUM(bk_pts) )
Ratio winners / UFE        = DIVIDE( SUM(winners),    SUM(unforced) )
```

> **Nota:** `serve_pts_won` no existe como columna explícita en Overview. Hay que calcularlo como `first_won + second_won`. En ServeBasics sí existe como columna directa.

---

## Hallazgos importantes

**1. `player` contiene el nombre completo, no un entero.**  
Se asumía que sería `1` o `2`, pero el campo contiene el nombre del jugador directamente. Esto simplifica el filtro de Djokovic pero obliga a verificar consistencia ortográfica en toda la tabla (sin variantes ni abreviaciones).

**2. No existe columna `pts_won` explícita.**  
Para calcular el total de puntos ganados al saque hay que sumar `first_won + second_won`. No está precalculado en esta tabla.

**3. Los 56 duplicados requieren investigación.**  
Con granularidad `match_id + player + set` no deberían existir duplicados. Pueden ser partidos re-charteados por distintos colaboradores o errores de carga. Hay que identificarlos y eliminarlos antes de modelar en Power BI.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador por nombre |
| `set` | Mantener — **filtrar por `Total`** para análisis de partido completo |
| `serve_pts` | Mantener — denominador saque |
| `aces` | Mantener — KPI saque |
| `dfs` | Mantener — KPI riesgo saque |
| `first_in` | Mantener — numerador % 1S dentro |
| `first_won` | Mantener — KPI rendimiento 1S |
| `second_in` | Mantener — denominador % 2S |
| `second_won` | Mantener — KPI rendimiento 2S |
| `bk_pts` | Mantener — denominador % BP salvados |
| `bp_saved` | Mantener — KPI presión al saque |
| `return_pts` | Mantener — denominador resto |
| `return_pts_won` | Mantener — KPI rendimiento al resto |
| `winners` | Mantener — agresividad |
| `winners_fh` | Mantener — perfil táctico fh/bh |
| `winners_bh` | Mantener — perfil táctico fh/bh |
| `unforced` | Mantener — consistencia |
| `unforced_fh` | Mantener — lado débil |
| `unforced_bh` | Mantener — lado débil |

---

*Fuente: output de `dfs["overview"].head(10)` y `dfs["overview"].dtypes` ejecutados en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*