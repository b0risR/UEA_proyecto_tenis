# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-matches.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 partido  
**Filas:** 7.566  
**Columnas:** 15  
**Nulos:** 7,56% (concentrados en `Time`, `Court`, `Umpire`)  
**Duplicados:** 0

---

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-Jesper_D...` | Identificador único del partido. Formato: `AAAAMMDD-M-Torneo-Ronda-Jugador1-Jugador2`. Es la **clave de join** con todas las tablas de estadísticas. | Truncado en pantalla pero completo en el CSV. El campo `M` indica masculino. |
| `Player 1` | object | `Jesper De Jong`, `Casper Ruud`... | Nombre del jugador que **sacó primero** en el partido. En todo el dataset, Player 1 es siempre el primer servidor. | Para filtrar Djokovic hay que buscarlo en **ambas** columnas Player 1 y Player 2. |
| `Player 2` | object | `Michael Zheng`, `Jannik Sinner`... | Nombre del jugador que restó primero. | Ídem. |
| `Pl 1 hand` | object | `R`, `L` | Mano dominante de Player 1. `R` = diestro, `L` = zurdo. | Relevante tácticamente: la dirección del saque wide/T se invierte para zurdos. |
| `Pl 2 hand` | object | `R`, `L` | Mano dominante de Player 2. | Ídem. |
| `Date` | object | `20260521`, `20260517`... | Fecha del partido en formato `AAAAMMDD`. Viene como `object` (string), **no como fecha**. Hay que convertirlo a datetime en Power Query. | Primera columna a transformar al cargar en Power BI. |
| `Tournament` | object | `Roland Garros`, `Rome Masters`, `Madrid Masters`... | Nombre del torneo. Texto libre, puede haber variantes ortográficas del mismo torneo a lo largo de los años. | Revisar consistencia antes de usar como filtro (ej. `US Open` vs `US_Open`). |
| `Round` | object | `Q3`, `F`, `R16`, `R128`, `SF`, `QF`... | Ronda del torneo. `F`=Final, `SF`=Semifinal, `QF`=Cuartos, `R16/R32/R64/R128`=rondas previas, `Q1/Q2/Q3`=clasificación. | Útil para filtrar partidos de alta presión (F, SF, QF). Las rondas de clasificación (Q) tienen menor relevancia táctica. |
| `Time` | object | `5pm`, `4:35 PM`, `9pm`, `14:30`, `NaN`... | Hora local de inicio del partido. Formato inconsistente — mezcla 12h, 24h y texto libre. | **Alta tasa de nulos.** No es útil para el análisis táctico. Columna descartable. |
| `Court` | object | `Centre`, `Santana`, `Paribas`, `7`, `Arantxa Sanchez`, `NaN`... | Nombre o número de la pista donde se jugó. Muy inconsistente: mezcla nombres propios, números y pistas con nombre de patrocinador. | **Alta tasa de nulos.** Valor informativo bajo. Descartable salvo para contextualizar. |
| `Surface` | object | `Clay`, `Hard`, `Grass` | Superficie de la pista. **Variable más importante** del dataset para segmentación táctica. Valores limpios y consistentes. | **Filtro obligatorio** en todo el informe. Nunca mezclar superficies sin segmentar. |
| `Umpire` | object | `Renaud Lichtenstein`, `Mohamed Lahyani`, `NaN`... | Nombre del árbitro de silla. | Sin valor táctico. Presencia de nulos frecuente. Columna descartable. |
| `Best of` | object | `3`, `5` | Número máximo de sets del partido. `3` = mejor de 3 (Masters, ATP 500/250), `5` = mejor de 5 (Grand Slams). | Importante para contextualizar: los patrones de presión en un Grand Slam (5 sets) son distintos a un Masters (3 sets). Viene como `object` aunque es numérico — convertir en Power Query. |
| `Final TB?` | object | `1`, `A`, `NaN`... | Indica el tipo de desempate en el set decisivo. `1` = tie-break normal a 7 puntos, `A` = super tie-break a 10 puntos (formato ATP moderno en 3er set). | Relevante para análisis de puntos de presión. El valor `A` indica formato diferente al tie-break tradicional. |
| `Charted by` | object | `stard54`, `Edo`, `BG`, `Ludo`, `Julie DL`... | Alias o nombre del colaborador que registró el partido punto a punto. | **Fuente de sesgo potencial**: distintos chárters pueden tener distintos criterios para clasificar errores forzados/no forzados. Importante mencionarlo en la sección de metodología. |

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `Player 1` | Mantener — filtro de jugador |
| `Player 2` | Mantener — filtro de jugador |
| `Pl 1 hand` | Mantener — contexto táctico |
| `Pl 2 hand` | Mantener — contexto táctico |
| `Date` | Mantener y **convertir a fecha** en Power Query |
| `Tournament` | Mantener — normalizar texto (trim, mayúsculas) |
| `Round` | Mantener — crear columna derivada de peso de ronda |
| `Surface` | Mantener — **filtro obligatorio** |
| `Best of` | Mantener y convertir a entero |
| `Final TB?` | Mantener — contexto de presión |
| `Charted by` | Mantener como referencia de sesgo — no usar como filtro analítico |
| `Time` | **Descartar** — nulos altos, formato inconsistente, sin valor táctico |
| `Court` | **Descartar** — nulos altos, muy inconsistente |
| `Umpire` | **Descartar** — sin valor táctico |

---

*Fuente: output de `dfs["matches"].head(10)` y `dfs["matches"].dtypes` ejecutados en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*

---
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

---

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

---
---

# `charting-m-stats-ServeDirection.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 tipo de saque (Total, 1S, 2S)  
**Filas:** 45.341 | **Columnas:** 15 | **Nulos:** 0 | **Duplicados:** 47  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural

Igual que ServeBasics, la columna `row` toma los valores `Total`, `1` y `2`, generando 3 filas por jugador y partido. La tabla tiene **dos bloques de columnas con lógica completamente distinta:**

- **Bloque de saques dentro** (`deuce_wide`, `deuce_middle`, `deuce_t`, `ad_wide`, `ad_middle`, `ad_t`): cuentan saques que **entraron** en zona válida, desglosados por lado de pista y dirección.
- **Bloque de errores de saque** (`err_net`, `err_wide`, `err_deep`, `err_wide_deep`, `err_foot`, `err_unknown`): cuentan saques que **fallaron**, desglosados por tipo de error.

Cuando `row = "1"` los errores son faltas de primer saque (sin consecuencia inmediata). Cuando `row = "2"` los errores son **dobles faltas** — pérdida directa de punto. Esta distinción es crítica para el análisis de riesgo al saque.

Una segunda diferencia importante respecto al diccionario previo: la zona central se llama `middle` aquí, no `body`. Son conceptos distintos — `body` en ServeBasics es dirección al cuerpo del rival, `middle` en ServeDirection es la zona geométrica central del cuadro de servicio.

---

## Columnas

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-Jesper_D...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador. | Filtrar Djokovic por nombre. |
| `row` | object | `Total`, `1`, `2` | Tipo de saque. `Total` = ambos saques combinados, `1` = primer saque, `2` = segundo saque. | **Filtro obligatorio.** Para el heatmap de dirección usar `row = "1"` y `row = "2"` por separado — la táctica de dirección cambia entre primer y segundo saque. |
| `deuce_wide` | int64 | `8`, `6`, `2`... | Saques dirigidos al exterior (wide) desde el lado deuce que **entraron** en zona válida. El lado deuce es el lado derecho de la pista — puntos pares (0-0, 30-0, 0-30, 40-40). Para un diestro, wide en deuce va al backhand del rival diestro. | **Columna central del heatmap.** Para Djokovic diestro: wide en deuce ataca el revés del rival. |
| `deuce_middle` | int64 | `1`, `1`, `0`... | Saques dirigidos a la zona central del cuadro de servicio desde el lado deuce que entraron. Va aproximadamente al cuerpo del rival. | No confundir con `body` de ServeBasics. Aquí `middle` es una zona geométrica del cuadro, no necesariamente al cuerpo. Frecuencia baja — es la zona menos usada. |
| `deuce_t` | int64 | `33`, `18`, `15`... | Saques dirigidos a la línea T (centro de la red) desde el lado deuce que entraron. Para un diestro en deuce, el T va al forehand del rival diestro. | Suele ser la zona más frecuente en deuce para jugadores diestros porque genera ángulo y velocidad. |
| `ad_wide` | int64 | `19`, `12`, `7`... | Saques dirigidos al exterior desde el lado ad que entraron. El lado ad es el lado izquierdo — puntos impares (15-0, 0-15, 40-30, 30-40). Para un diestro, wide en ad va al forehand del rival diestro. | En puntos de break point desde el lado ad, muchos jugadores optan por wide al forehand del rival para evitar el revés de Djokovic. |
| `ad_middle` | int64 | `4`, `0`, `4`... | Saques a la zona central desde el lado ad que entraron. | Frecuencia muy baja. Zona de uso táctico puntual. |
| `ad_t` | int64 | `15`, `11`, `4`... | Saques a la línea T desde el lado ad que entraron. Para un diestro en ad, el T va al backhand del rival diestro. | En ad, sacar al T es atacar el revés del rival — táctica habitual en puntos clave. |
| `err_net` | int64 | `16`, `0`, `16`... | Saques que golpearon la red. Tipo de error más común en primer saque al intentar mayor potencia. | Cuando `row = "1"` este campo es siempre `0` — las faltas de primer saque no se registran aquí. Solo aparecen en `row = "2"` (doble falta) y `row = "Total"`. |
| `err_wide` | int64 | `6`, `0`, `6`... | Saques que salieron por el lateral (fuera del cuadro de servicio lateralmente). | Ídem — solo relevante en `row = "2"` y `Total`. |
| `err_deep` | int64 | `11`, `0`, `11`... | Saques que salieron largos (fuera del cuadro de servicio por el fondo). | Ídem. |
| `err_wide_deep` | int64 | `3`, `0`, `3`... | Saques que salieron tanto laterales como largos simultáneamente. Error de máxima imprecisión. | Ídem. Frecuencia baja. |
| `err_foot` | int64 | `0`, `0`, `0`... | Faltas de pie (foot fault) — el servidor pisó la línea de fondo antes de golpear. | Frecuencia casi nula en el dataset — raramente registrado por los chárters. Prácticamente descartable. |
| `err_unknown` | int64 | `0`, `0`, `0`... | Errores de saque cuyo tipo no pudo determinarse. | Frecuencia nula en los datos observados. Descartable. |

---

## Lógica de los errores por tipo de `row`

Este es el hallazgo más importante de la tabla y no estaba documentado previamente:

```
row = "1"  →  deuce_wide + deuce_middle + deuce_t
              + ad_wide + ad_middle + ad_t  =  primeros saques dentro
              err_*  =  todos 0  (las faltas de 1S no se registran aquí)

row = "2"  →  deuce_wide + deuce_middle + deuce_t
              + ad_wide + ad_middle + ad_t  =  segundos saques dentro
              err_*  =  dobles faltas desglosadas por tipo de error

row = "Total"  →  suma de ambos + todos los errores
```

Para calcular el **% de primer saque dentro** no se puede usar esta tabla directamente — hay que ir a ServeBasics (`pts` con `row = "1"` como denominador). ServeDirection solo cuenta los saques que **entraron**, no el total de intentos.

---

## Diagrama de zonas del cuadro de servicio

```
                    RED
    ┌─────────────────────────────────────┐
    │  LADO DEUCE (derecho)               │
    │                                     │
    │  [wide]   [middle]   [T]            │
    │  ←ext      centro    →centro red    │
    └─────────────────────────────────────┘
    ┌─────────────────────────────────────┐
    │  LADO AD (izquierdo)                │
    │                                     │
    │  [T]      [middle]   [wide]         │
    │  →centro  centro     ←ext           │
    └─────────────────────────────────────┘
                 SERVIDOR
```

Para un jugador **diestro** (caso de Djokovic):
- Deuce wide → backhand del rival diestro
- Deuce T → forehand del rival diestro
- Ad wide → forehand del rival diestro
- Ad T → backhand del rival diestro

---

## Medidas DAX derivables de esta tabla

```dax
-- Frecuencia de dirección en primer saque (heatmap) [row = "1"]
-- Denominador = total saques dentro en 1S

Total saques dentro 1S =
    SUM(deuce_wide) + SUM(deuce_middle) + SUM(deuce_t)
    + SUM(ad_wide)  + SUM(ad_middle)   + SUM(ad_t)

% Deuce Wide  1S = DIVIDE( SUM(deuce_wide),   [Total saques dentro 1S] )
% Deuce T     1S = DIVIDE( SUM(deuce_t),      [Total saques dentro 1S] )
% Ad Wide     1S = DIVIDE( SUM(ad_wide),      [Total saques dentro 1S] )
% Ad T        1S = DIVIDE( SUM(ad_t),         [Total saques dentro 1S] )
% Deuce Mid   1S = DIVIDE( SUM(deuce_middle), [Total saques dentro 1S] )
% Ad Mid      1S = DIVIDE( SUM(ad_middle),    [Total saques dentro 1S] )

-- Idem para segundo saque [row = "2"]

-- Análisis de dobles faltas por tipo de error [row = "2"]
Total DF =
    SUM(err_net) + SUM(err_wide) + SUM(err_deep) + SUM(err_wide_deep)

% DF por red      = DIVIDE( SUM(err_net),       [Total DF] )
% DF largas       = DIVIDE( SUM(err_deep),       [Total DF] )
% DF laterales    = DIVIDE( SUM(err_wide),       [Total DF] )
% DF wide y larga = DIVIDE( SUM(err_wide_deep),  [Total DF] )
```

---

## Hallazgos importantes

**1. La zona central se llama `middle`, no `body`.**  
El diccionario previo documentaba `body` como zona central. En ServeDirection la zona central geométrica del cuadro se llama `middle`. Son conceptos relacionados pero distintos.

**2. Los errores de saque solo aparecen en `row = "2"` y `Total`.**  
En `row = "1"` todos los campos `err_*` son 0. Las faltas de primer saque no se registran aquí — solo los saques que entraron. Las dobles faltas solo ocurren en segundo saque.

**3. `err_foot` y `err_unknown` son prácticamente nulos.**  
Pueden descartarse del modelo sin pérdida de información relevante.

**4. Los 47 duplicados coinciden exactamente con los de ServeBasics.**  
Son los mismos partidos — el problema de duplicados es a nivel de `match_id`, no de tabla individual. Hay que identificarlos y eliminarlos antes de modelar.

**5. Esta tabla no permite calcular % de primer saque dentro.**  
Solo registra saques que entraron, no el total de intentos. Para el denominador hay que cruzar con ServeBasics.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador por nombre |
| `row` | Mantener — usar `1` y `2` por separado para el heatmap |
| `deuce_wide` | Mantener — **heatmap saque** |
| `deuce_middle` | Mantener — heatmap saque |
| `deuce_t` | Mantener — **heatmap saque** |
| `ad_wide` | Mantener — **heatmap saque** |
| `ad_middle` | Mantener — heatmap saque |
| `ad_t` | Mantener — **heatmap saque** |
| `err_net` | Mantener — análisis de dobles faltas |
| `err_wide` | Mantener — análisis de dobles faltas |
| `err_deep` | Mantener — análisis de dobles faltas |
| `err_wide_deep` | Mantener — análisis de dobles faltas |
| `err_foot` | **Descartar** — frecuencia nula |
| `err_unknown` | **Descartar** — frecuencia nula |

---

*Fuente: output de `dfs["serve_direction"].head(10)` y `dfs["serve_direction"].dtypes` ejecutados en `01_exploracion_inicial.ipynb`.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*

---
---

# `charting-m-stats-ReturnDepth.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 combinación (saque × tipo de golpe × lado de pista × dirección de saque)  
**Columnas:** 11 | **Nulos:** 0 | **Duplicados:** por confirmar  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

Esta es la tabla **más granular** de todo el dataset. A diferencia de Overview (1 fila por jugador×set) o ServeBasics (3 filas por jugador), aquí hay **18 filas por jugador por partido**. Cada fila representa una intersección de tres dimensiones simultáneas:

1. **Tipo de saque recibido** — primer o segundo saque (`v1st` / `v2nd`)
2. **Tipo de golpe de resto** — topspin forehand/backhand, o slice forehand/backhand (`fh`, `bh`, `gs`, `sl`)
3. **Contexto del saque** — lado de pista (Deuce/Advantage) y dirección (wide=4, body=5, T=6)

Las columnas de datos se repiten con idéntico significado en todas las filas — lo que cambia es el subconjunto de restos que cada fila describe.

### Estructura aritmética fundamental

La relación entre columnas es la siguiente (verificada con datos reales):

```
Total restos intentados  =  shallow + deep + very_deep
                         =  returnable + unforced + err_net + err_deep + err_wide + err_wide_deep
```

**Implicación clave:** `shallow`, `deep` y `very_deep` registran la profundidad de **todos** los restos intentados, incluyendo los que terminaron en error. No son un subconjunto de `returnable` — son el total. Esto tiene sentido tácticamente: la profundidad de un resto es observable aunque el golpe acabe en la red.

```
Ejemplo verificado — Jesper De Jong, fila Total (row="Total"):
  shallow(13) + deep(37) + very_deep(10) = 60  ← total restos intentados
  returnable(52) + unforced(4) + err_net(0) + err_deep(1) + err_wide(3) + err_wide_deep(0) = 60  ✓
```

---

## Tabla 1 — Decodificador completo de la columna `row`

Esta tabla es **imprescindible** para usar ReturnDepth. Sin ella, los valores de `row` son opacos.

| Valor de `row` | Dimensión 1: Saque | Dimensión 2: Tipo de golpe | Dimensión 3: Contexto | Descripción en lenguaje natural |
|---|---|---|---|---|
| `Total` | Ambos | **Todos** | **Todos** | Todos los restos del partido sin ningún filtro. Fila de totales globales. |
| `v1st` | Primer saque | **Todos** | **Todos** | Todos los restos contra primer saque, sin filtro de golpe ni dirección. |
| `v2nd` | Segundo saque | **Todos** | **Todos** | Todos los restos contra segundo saque, sin filtro de golpe ni dirección. |
| `fh` | Ambos | Forehand topspin | **Todos** | Restos de derecha (topspin), primer y segundo saque combinados. |
| `bh` | Ambos | Backhand topspin | **Todos** | Restos de revés (topspin), primer y segundo saque combinados. |
| `gs` | Ambos | Groundstroke (fh+bh topspin) | **Todos** | Total de restos de golpe de fondo (derecha + revés topspin). Equivale a `fh` + `bh`. |
| `sl` | Ambos | Slice (fh o bh) | **Todos** | Total de restos con efecto cortado (chip/slice). Incluye devoluciones defensivas. |
| `D` | Ambos | **Todos** | Lado Deuce | Todos los restos jugados desde el lado Deuce (puntos pares: 0-0, 30-0, 0-30, 40-40). |
| `A` | Ambos | **Todos** | Lado Advantage | Todos los restos jugados desde el lado Advantage (puntos impares: 15-0, 0-15, 40-30, 30-40). |
| `4` | Ambos | **Todos** | Saque wide | Restos contra saques dirigidos al exterior (wide), ambos lados combinados. |
| `5` | Ambos | **Todos** | Saque al cuerpo | Restos contra saques al cuerpo del restador, ambos lados combinados. |
| `6` | Ambos | **Todos** | Saque a la T | Restos contra saques a la línea central (T), ambos lados combinados. |
| `4D` | Ambos | **Todos** | Wide + Deuce | Restos contra saque wide **desde el lado Deuce**. Para rival diestro: ataca el backhand del restador diestro. |
| `4A` | Ambos | **Todos** | Wide + Advantage | Restos contra saque wide **desde el lado Advantage**. Para rival diestro: ataca el forehand del restador diestro. |
| `5D` | Ambos | **Todos** | Cuerpo + Deuce | Restos contra saque al cuerpo **desde el lado Deuce**. |
| `5A` | Ambos | **Todos** | Cuerpo + Advantage | Restos contra saque al cuerpo **desde el lado Advantage**. |
| `6D` | Ambos | **Todos** | T + Deuce | Restos contra saque a la T **desde el lado Deuce**. Para rival diestro: ataca el forehand del restador diestro. |
| `6A` | Ambos | **Todos** | T + Advantage | Restos contra saque a la T **desde el lado Advantage**. Para rival diestro: ataca el backhand del restador diestro. |

> **Nota sobre jerarquía:** Los grupos de filas NO son mutuamente excluyentes entre sí. `v1st`, `gs`, `D` y `4D` pueden estar describiendo parcialmente los mismos restos desde ángulos distintos. Nunca sumar filas de grupos distintos. Para análisis general usar `v1st` y `v2nd`. Para el heatmap de dirección usar `4D`, `4A`, `5D`, `5A`, `6D`, `6A`.

> **Verificación de aditividad interna:** Dentro del grupo saque: `shallow(v1st) + shallow(v2nd) = shallow(Total)`. Ídem para `deep`, `very_deep`, `returnable` y cada columna de error. Esta identidad permite detectar errores de carga.

---

## Tabla 2 — Columnas de datos (comunes a todas las filas)

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador **restador** — quien devuelve el saque, no quien sirve. | Para analizar cómo Djokovic devuelve, filtrar `player = "Novak Djokovic"`. |
| `row` | object | `Total`, `v1st`, `v2nd`, `fh`, `bh`, `gs`, `sl`, `D`, `A`, `4`, `5`, `6`, `4D`, `4A`, `5D`, `5A`, `6D`, `6A` | Identificador del subconjunto de restos que describe la fila. Ver Tabla 1. | **Filtro obligatorio siempre.** Sin él, las sumas se multiplican por el número de grupos activos y todos los totales son incorrectos. |
| `returnable` | int64 | `52`, `27`, `25`... | Restos que **entraron en juego** — el servidor no ganó el punto directamente con el saque y el resto no fue error. `returnable / (shallow + deep + very_deep)` = % restos en juego (RiP%). | No es el total de restos intentados. El total real es `shallow + deep + very_deep`. La diferencia son los errores de resto (`unforced` + `err_*`). |
| `shallow` | int64 | `13`, `8`, `5`... | Restos que aterrizaron **dentro del cuadro de servicio** (zona corta, código 7 en el sistema MCP). Devolución corta — baja presión sobre el servidor. | Registrado sobre el **total de restos intentados**, incluyendo los que acabaron en error. Un `shallow` alto indica que el restador no está atacando la devolución. |
| `deep` | int64 | `37`, `20`, `17`... | Restos que aterrizaron **entre la línea de servicio y la mitad de la pista** (zona media, código 8). Devolución neutra. | Zona intermedia — ni corta ni profunda. Permite al servidor organizar el punto sin ventaja clara para el restador. |
| `very_deep` | int64 | `10`, `5`, `5`... | Restos que aterrizaron **en el cuarto posterior de la pista**, cerca de la línea de fondo (zona profunda, código 9). Devolución profunda — máxima presión sobre el servidor. | **Columna táctica más importante de la tabla.** Un `very_deep` alto indica que el restador empuja al servidor fuera de la pista y toma control del rally desde el primer golpe. Djokovic históricamente lidera este indicador. |
| `unforced` | int64 | `4`, `0`, `4`... | Errores **no forzados** del restador — fallos cometidos sin presión aparente del servidor. Resto fallado que el jugador debería haber devuelto. | Complementario a `err_*`. La suma `unforced + err_net + err_deep + err_wide + err_wide_deep` da el total de errores de resto. Un `unforced` alto en `row = "v2nd"` indica que el restador está sobreagresivo contra el segundo saque. |
| `err_net` | int64 | `0`, `1`, `0`... | Restos que golpearon la red. Tipo de error más común al intentar restos agresivos con poca trayectoria. | Generalmente el tipo de error más frecuente. Comparte lógica con `err_net` de ServeDirection. |
| `err_deep` | int64 | `1`, `0`, `1`... | Restos que salieron largos (fuera de la pista por el fondo). | Indica falta de control en profundidad — habitual en restos agresivos con mucho topspin. |
| `err_wide` | int64 | `3`, `0`, `3`... | Restos que salieron por el lateral. | Indica falta de control en dirección — habitual en restos intentando cambiar el ángulo del saque. |
| `err_wide_deep` | int64 | `0`, `0`, `0`... | Restos que salieron a la vez laterales y largos. Error de máxima imprecisión. | Frecuencia casi nula — prácticamente descartable en el análisis. |

---

## Relaciones y verificaciones cruzadas

```
# Identidad fundamental (verificada con datos reales)
shallow + deep + very_deep  =  returnable + unforced + err_net + err_deep + err_wide + err_wide_deep

# Aditividad interna entre grupos v1st / v2nd
shallow(v1st) + shallow(v2nd)       = shallow(Total)       ✓
deep(v1st) + deep(v2nd)             = deep(Total)          ✓
very_deep(v1st) + very_deep(v2nd)   = very_deep(Total)     ✓
returnable(v1st) + returnable(v2nd) = returnable(Total)    ✓

# gs ≈ fh + bh  (pueden diferir levemente si hay restos sin código de tipo registrado)
returnable(gs)  ≈  returnable(fh) + returnable(bh)

# Verificación con Overview (mismo partido, mismo jugador, set="Total")
shallow(Total) + deep(Total) + very_deep(Total)  ≈  return_pts (Overview)
```

---

## Medidas DAX derivables de esta tabla

```dax
-- Total de restos intentados (no existe como columna explícita)
Total Restos =
    SUM(shallow) + SUM(deep) + SUM(very_deep)
    -- Usar siempre con filtro de row activo

-- % Restos en juego (RiP%)
% Restos en juego = DIVIDE( SUM(returnable), SUM(shallow) + SUM(deep) + SUM(very_deep) )

-- Profundidad del resto (Matriz de profundidad — visualización obligatoria del E2)
% Shallow   = DIVIDE( SUM(shallow),   SUM(shallow) + SUM(deep) + SUM(very_deep) )
% Deep      = DIVIDE( SUM(deep),      SUM(shallow) + SUM(deep) + SUM(very_deep) )
% Very Deep = DIVIDE( SUM(very_deep), SUM(shallow) + SUM(deep) + SUM(very_deep) )

-- Return Depth Index (RDI) — fórmula oficial de Tennis Abstract para ATP
-- Pesos oficiales ATP: shallow=1, deep=2, very_deep=3.5
RDI =
    DIVIDE(
        SUM(shallow)*1 + SUM(deep)*2 + SUM(very_deep)*3.5,
        SUM(shallow) + SUM(deep) + SUM(very_deep)
    )
-- Rango orientativo: ~2.5 (poco profundo) a ~3.3 (muy profundo)

-- Caída de agresividad 1S → 2S
% Very Deep 1S = DIVIDE( SUM(very_deep), SUM(shallow)+SUM(deep)+SUM(very_deep) )  -- [row = "v1st"]
% Very Deep 2S = DIVIDE( SUM(very_deep), SUM(shallow)+SUM(deep)+SUM(very_deep) )  -- [row = "v2nd"]
Δ Agresividad  = [% Very Deep 2S] - [% Very Deep 1S]
-- Positivo → Djokovic ataca más el segundo saque (esperado)

-- Tipos de error de resto
% Error red   = DIVIDE( SUM(err_net),       SUM(shallow)+SUM(deep)+SUM(very_deep) )
% Error largo  = DIVIDE( SUM(err_deep),     SUM(shallow)+SUM(deep)+SUM(very_deep) )
% Error lateral= DIVIDE( SUM(err_wide),     SUM(shallow)+SUM(deep)+SUM(very_deep) )
% Error no forzado = DIVIDE( SUM(unforced), SUM(shallow)+SUM(deep)+SUM(very_deep) )

-- Efectividad por dirección de saque recibida (heatmap — usar filas 4D, 4A, 5D, 5A, 6D, 6A)
% En juego vs Wide Deuce      = DIVIDE( SUM(returnable), SUM(shallow)+SUM(deep)+SUM(very_deep) ) -- [row="4D"]
% En juego vs T Advantage     = DIVIDE( SUM(returnable), SUM(shallow)+SUM(deep)+SUM(very_deep) ) -- [row="6A"]
-- Repetir para cada celda del heatmap
```

---

## Hallazgos importantes

**1. `shallow + deep + very_deep` es el total real de restos intentados, no `returnable`.**
Este es el hallazgo más importante de la tabla y no es evidente sin verificación aritmética. El total de restos es 60 para Jesper De Jong en la fila `Total`, no 52. El denominador correcto para todos los ratios de profundidad y de error es la suma de las tres zonas.

**2. La profundidad se registra incluso en restos que acaban en error.**
A diferencia de lo que podría esperarse, `shallow`, `deep` y `very_deep` incluyen restos fallados. Un resto profundo que va a la red sigue contando en `very_deep`. Esto tiene sentido táctico: la intención del golpe es observable aunque el resultado sea un error.

**3. `unforced` distingue errores con y sin presión.**
La tabla separa explícitamente errores no forzados (`unforced`) de errores forzados por el saque (`err_net`, `err_deep`, `err_wide`). Esta distinción es tácticamente valiosa: un `unforced` alto en `v1st` indica que el restador está asumiendo demasiado riesgo contra el primer saque.

**4. `err_wide_deep` es prácticamente nulo.**
Consistente con ServeDirection — puede descartarse del modelo sin pérdida de información relevante.

**5. La granularidad de 18 filas exige control estricto del filtro `row` en Power BI.**
Sin filtro activo sobre `row`, cualquier medida suma las 18 filas y los totales son incorrectos por un factor de hasta 18x. Usar siempre una tabla desconectada de selección de `row` o parámetros de campo en el modelo.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador |
| `row` | Mantener — **filtro obligatorio siempre activo** |
| `returnable` | Mantener — % restos en juego (RiP%) |
| `shallow` | Mantener — profundidad corta |
| `deep` | Mantener — profundidad media |
| `very_deep` | Mantener — **profundidad alta, KPI táctico principal** |
| `unforced` | Mantener — errores no forzados al resto |
| `err_net` | Mantener — tipo de error más frecuente |
| `err_deep` | Mantener — error por largo |
| `err_wide` | Mantener — error por lateral |
| `err_wide_deep` | **Descartar** — frecuencia prácticamente nula |

---

*Fuente: output de `dfs["return_depth"].head(37)` y `dfs["return_depth"].dtypes` ejecutados en `01_exploracion_inicial.ipynb`, glosario oficial de Tennis Abstract (agosto 2019), y sistema de codificación del MatchChart.md.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*

---
---

# `charting-m-stats-ReturnOutcomes.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 subconjunto de restos (por tipo de saque, tipo de golpe, lado/dirección del saque, o profundidad del resto)  
**Columnas:** 10 | **Nulos:** 0 | **Duplicados:** por confirmar  
**Clave de join:** `match_id` + `player` + `row`

---

## Nota estructural crítica

ReturnOutcomes tiene **21 filas por jugador por partido** — tres más que ReturnDepth (18). La columna `row` incluye todos los valores de ReturnDepth más tres nuevos: `7`, `89` y `9`. Estos tres valores representan la misma segmentación de profundidad que las columnas `shallow`, `deep` y `very_deep` de ReturnDepth, pero aquí la profundidad se usa como **dimensión de fila**, permitiendo cruzar profundidad con resultado del punto.

La tabla tiene más columnas de resultado que ReturnDepth: además de `returnable`, añade `returnable_won`, `in_play`, `in_play_won`, `winners` y `total_shots`. La distinción entre `returnable` e `in_play` es uno de los hallazgos más importantes de esta tabla.

### Estructura aritmética fundamental

```
pts  =  returnable  +  errores de resto
     ≈  suma de (row=7) + (row=89) + (row=9)  [con diferencia ≤1 por restos sin profundidad registrada]

returnable  ≥  in_play
returnable - in_play  =  restos que entraron pero el punto terminó en el primer golpe
                         del rally (winner o forced error inmediato del servidor)
```

Ejemplo verificado — Jesper De Jong, fila Total (row="Total"):
```
pts(61) = returnable(52) + errores(9)
7(13) + 89(37) + 9(10) = 60  ≈ pts(61)  [1 resto sin profundidad registrada]
returnable(52) - in_play(48) = 4  →  4 restos que entraron pero no generaron rally
```

---

## Tabla 1 — Decodificador completo de la columna `row`

| Valor de `row` | Dimensión segmentada | Descripción en lenguaje natural |
|---|---|---|
| `Total` | Ninguna (totales globales) | Todos los restos del partido sin filtro. Fila de referencia para verificar consistencia. |
| `v1st` | Tipo de saque | Restos contra **primer saque**. Denominador para análisis de efectividad al resto de 1S. |
| `v2nd` | Tipo de saque | Restos contra **segundo saque**. Denominador para análisis de efectividad al resto de 2S y agresividad. |
| `7` | Profundidad del resto | Restos que aterrizaron **dentro del cuadro de servicio** (zona corta, código 7 MCP = `shallow` en ReturnDepth). |
| `89` | Profundidad del resto | Restos que aterrizaron **entre la línea de servicio y el área media de la pista** (zona media, código 8 MCP = `deep` en ReturnDepth). El nombre "89" indica la transición entre la zona media (8) y la profunda (9) — corresponde numéricamente a `deep`. |
| `9` | Profundidad del resto | Restos que aterrizaron **muy cerca de la línea de fondo** (zona profunda, código 9 MCP = `very_deep` en ReturnDepth). Máxima presión sobre el servidor. |
| `fh` | Tipo de golpe | Restos de **forehand topspin** (derecha), primer y segundo saque combinados. |
| `bh` | Tipo de golpe | Restos de **backhand topspin** (revés), primer y segundo saque combinados. |
| `gs` | Tipo de golpe | Restos de **groundstroke** (derecha + revés topspin). Equivale a `fh` + `bh`. |
| `sl` | Tipo de golpe | Restos de **slice** (chip/slice forehand o backhand). Devoluciones defensivas. |
| `D` | Lado de pista | Restos desde el **lado Deuce** (puntos pares: 0-0, 30-0, 0-30, 40-40). |
| `A` | Lado de pista | Restos desde el **lado Advantage** (puntos impares: 15-0, 0-15, 40-30, 30-40). |
| `4` | Dirección del saque recibido | Restos contra saque **wide**, ambos lados combinados. |
| `5` | Dirección del saque recibido | Restos contra saque **al cuerpo**, ambos lados combinados. |
| `6` | Dirección del saque recibido | Restos contra saque **a la T**, ambos lados combinados. |
| `4D` | Dirección + lado | Restos contra saque wide **desde el lado Deuce**. Para rival diestro: ataca el backhand del restador diestro. |
| `4A` | Dirección + lado | Restos contra saque wide **desde el lado Advantage**. Para rival diestro: ataca el forehand del restador diestro. |
| `5D` | Dirección + lado | Restos contra saque al cuerpo **desde el lado Deuce**. |
| `5A` | Dirección + lado | Restos contra saque al cuerpo **desde el lado Advantage**. |
| `6D` | Dirección + lado | Restos contra saque a la T **desde el lado Deuce**. Para rival diestro: ataca el forehand del restador diestro. |
| `6A` | Dirección + lado | Restos contra saque a la T **desde el lado Advantage**. Para rival diestro: ataca el backhand del restador diestro. |

> **Correspondencia con ReturnDepth:** Los grupos de `row` `Total`, `v1st/v2nd`, `fh/bh/gs/sl`, `D/A`, `4/5/6` y `4D/4A/5D/5A/6D/6A` son idénticos a los de ReturnDepth. Los tres valores nuevos `7`, `89`, `9` corresponden exactamente a las columnas `shallow`, `deep`, `very_deep` de ReturnDepth, pero aquí permiten ver los **outcomes** (puntos ganados, winners, golpes en rally) **dentro de cada zona de profundidad**.

> **Los grupos no son mutuamente excluyentes entre sí.** Nunca sumar filas de grupos distintos. Para análisis general usar `v1st` y `v2nd`. Para cruzar profundidad con resultado usar `7`, `89`, `9`.

---

## Tabla 2 — Columnas de datos

| Columna | Tipo real | Valores observados | Significado | Notas críticas |
|---|---|---|---|---|
| `match_id` | object | `20260521-M-Roland_Garros-Q3-...` | Clave de join con `matches`. | Igual que en todas las tablas. |
| `player` | object | `Jesper De Jong`, `Michael Zheng`... | Nombre completo del jugador **restador**. | Para analizar cómo Djokovic devuelve, filtrar `player = "Novak Djokovic"`. |
| `row` | object | 21 valores — ver Tabla 1 | Subconjunto de restos que describe la fila. | **Filtro obligatorio siempre.** Sin él los totales son incorrectos por hasta 21x. |
| `pts` | int64 | `61`, `34`, `25`... | Total de puntos jugados al resto en ese subconjunto. Incluye tanto restos en juego como errores directos de resto. Denominador global de esta tabla. | Equivale a `serve_pts` del rival para ese subconjunto. Para `row = "Total"` coincide con `return_pts` de Overview. |
| `pts_won` | int64 | `18`, `7`, `9`... | Puntos ganados por el restador en ese subconjunto. `pts_won / pts` = % puntos ganados al resto en ese contexto. | **KPI principal de esta tabla.** Permite calcular la efectividad del resto por tipo de saque, profundidad, dirección y lado. |
| `returnable` | int64 | `52`, `27`, `25`... | Restos que **cruzaron la red y entraron en juego** — el servidor no ganó el punto directamente con el saque. `returnable / pts` = % restos en juego (RiP%). | No es el total de puntos. La diferencia `pts - returnable` son los errores directos de resto (aces, saques no devueltos, errores del restador). Igual a `returnable` de ReturnDepth. |
| `returnable_won` | int64 | `16`, `7`, `9`... | Puntos ganados por el restador **entre los restos que entraron en juego**. `returnable_won / returnable` = % puntos ganados en restos en juego. | Difiere de `pts_won` al excluir los errores directos. Permite aislar la efectividad del restador **una vez que el punto comienza**. |
| `in_play` | int64 | `48`, `27`, `21`... | Puntos en los que se desarrolló un **rally real** (el resto entró y el punto no se resolvió en el primer golpe post-saque). `in_play ≤ returnable` siempre. | La diferencia `returnable - in_play` son los restos que entraron pero el servidor ganó el punto con un winner o forced error inmediato en su primer golpe de rally. Esta diferencia suele concentrarse en `v2nd`, donde el restador devuelve con más agresividad y el servidor tiene menos margen. |
| `in_play_won` | int64 | `16`, `7`, `9`... | Puntos ganados por el restador **dentro del rally** (subconjunto de `in_play`). `in_play_won / in_play` = % puntos ganados en rallies. | En este partido, `in_play_won = returnable_won` en todos los grupos — indica que los puntos que el restador ganó fueron siempre en rallies, nunca con un winner de resto directo (`winners = 0`). |
| `winners` | int64 | `0`, `1`... | Winners de **resto directo** — restos ganadores que el servidor no pudo devolver. | Frecuencia muy baja: los winners de resto son raros incluso en los mejores restadores. Cuando `winners > 0` indica agresividad excepcional contra el segundo saque. |
| `total_shots` | int64 | `325`, `177`, `146`... | Total de **golpes** (shots) acumulados en todos los puntos del subconjunto, incluyendo el saque y el resto. `total_shots / pts` = promedio de golpes por punto en ese contexto. | Columna exclusiva de esta tabla — no existe en ReturnDepth. Permite calcular la **longitud media del rally desde la perspectiva del restador**. Un `total_shots` alto en `row = "9"` indicaría que los restos profundos generan rallies más largos. |

---

## Relaciones y verificaciones cruzadas

```
# Jerarquía de subconjuntos
pts  ≥  returnable  ≥  in_play
pts_won  ≥  returnable_won  ≥  in_play_won

# Consistencia entre grupos de profundidad (7 + 89 + 9 ≈ Total)
pts(7) + pts(89) + pts(9)  ≈  pts(Total)    [diferencia ≤ pts sin profundidad registrada]

# Aditividad interna v1st / v2nd
pts(v1st) + pts(v2nd)              =  pts(Total)
pts_won(v1st) + pts_won(v2nd)      =  pts_won(Total)
returnable(v1st) + returnable(v2nd)= returnable(Total)
total_shots(v1st) + total_shots(v2nd) ≈ total_shots(Total)  [puede diferir ±2 por redondeo]

# Correspondencia con ReturnDepth (mismo partido, mismo jugador, row="Total")
returnable(ReturnOutcomes)  =  returnable(ReturnDepth)
pts(ReturnOutcomes, row="Total")  ≈  shallow+deep+very_deep(ReturnDepth)

# Correspondencia con Overview (mismo partido, mismo jugador, set="Total")
pts(row="Total")  =  return_pts(Overview)
pts_won(row="Total")  =  return_pts_won(Overview)

# Verificación de consistencia de winners
winners  ≤  returnable_won   [un winner es siempre un punto ganado]
in_play_won + winners  ≤  returnable_won  [aproximadamente, pueden superponerse]
```

---

## Medidas DAX derivables de esta tabla

```dax
-- KPIs principales de resto [usar row = "v1st" y "v2nd" por separado]
% Puntos ganados al resto 1S  = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "v1st"]
% Puntos ganados al resto 2S  = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "v2nd"]

% Restos en juego 1S           = DIVIDE( SUM(returnable), SUM(pts) ) -- [row = "v1st"]
% Restos en juego 2S           = DIVIDE( SUM(returnable), SUM(pts) ) -- [row = "v2nd"]

-- Efectividad una vez en juego
% Puntos ganados en rally       = DIVIDE( SUM(in_play_won), SUM(in_play) )

-- Agresividad del resto (winners directos)
% Winners de resto              = DIVIDE( SUM(winners), SUM(returnable) )

-- Rally length medio desde perspectiva del restador
Shots por punto                 = DIVIDE( SUM(total_shots), SUM(pts) )

-- Efectividad por profundidad del resto (cruzar con 7, 89, 9)
% Ganados con resto corto       = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "7"]
% Ganados con resto medio       = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "89"]
% Ganados con resto profundo    = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "9"]
-- Diferencial profundo - corto indica el valor táctico de atacar la profundidad del resto

-- Efectividad por dirección de saque recibida (heatmap)
% Ganados vs Wide Deuce         = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "4D"]
% Ganados vs T Advantage        = DIVIDE( SUM(pts_won), SUM(pts) )   -- [row = "6A"]
-- Repetir para cada celda: 4A, 5D, 5A, 6D

-- Caída 1S → 2S
Δ % Ganados al resto            = [% Puntos ganados 2S] - [% Puntos ganados 1S]
-- Positivo indica que el restador gana más puntos contra segundo saque (esperable)
-- La magnitud indica cuánto se puede explotar el segundo saque

-- Rally generado por profundidad
Shots por punto (restos profundos) = DIVIDE( SUM(total_shots), SUM(pts) ) -- [row = "9"]
Shots por punto (restos cortos)    = DIVIDE( SUM(total_shots), SUM(pts) ) -- [row = "7"]
```

---

## Hallazgos importantes

**1. `7`, `89`, `9` son las filas equivalentes a `shallow`, `deep`, `very_deep` de ReturnDepth.**
La verificación numérica confirma la correspondencia exacta: `pts(row="7") = shallow`, `pts(row="89") = deep`, `pts(row="9") = very_deep`. ReturnOutcomes añade el eje de resultados (pts_won, in_play, total_shots) a esa misma segmentación de profundidad, permitiendo responder la pregunta táctica completa: ¿los restos más profundos de Djokovic realmente se traducen en puntos ganados?

**2. `returnable` ≠ `in_play` — distinción táctica importante.**
`returnable` cuenta todos los restos que cruzaron la red. `in_play` excluye los puntos donde el servidor respondió con un winner o forced error inmediato (el rally no llegó a desarrollarse). La diferencia se concentra en `v2nd` (4 puntos en el ejemplo), indicando que el segundo saque genera restos más expuestos a una respuesta agresiva inmediata del servidor.

**3. `total_shots` permite calcular rally length por contexto de resto.**
Combinando `total_shots / pts` por `row`, se puede ver si los restos profundos (`row = "9"`) generan rallies más largos que los restos cortos (`row = "7"`). Esta métrica conecta directamente con la tabla Rally y permite validar la consistencia entre ambas.

**4. `winners` de resto es casi nulo en la mayoría de partidos.**
Un `winners = 0` en la fila Total no indica problema de datos — refleja que los winners de resto son eventos raros. Su presencia ocasional (`row = "v2nd"` o `row = "9"`) tiene alto valor táctico: indica los contextos donde el restador puede atacar directamente.

**5. Esta tabla es la fuente más completa para la "Matriz de profundidad del resto" del E2.**
Combina en una sola tabla la información de profundidad de ReturnDepth con los outcomes de punto que ReturnDepth no tiene (`pts_won`, `in_play`, `total_shots`). Para el informe Power BI, es preferible usar ReturnOutcomes sobre ReturnDepth cuando se necesita cruzar profundidad con resultado.

---

## Columnas recomendadas para el modelo Power BI

| Columna | Acción recomendada |
|---|---|
| `match_id` | Mantener — clave de join |
| `player` | Mantener — filtro de jugador |
| `row` | Mantener — **filtro obligatorio siempre activo** |
| `pts` | Mantener — denominador global (total puntos al resto) |
| `pts_won` | Mantener — **KPI principal de efectividad al resto** |
| `returnable` | Mantener — % restos en juego (RiP%) |
| `returnable_won` | Mantener — efectividad condicionada al resto en juego |
| `in_play` | Mantener — denominador de efectividad en rally |
| `in_play_won` | Mantener — efectividad en rally |
| `winners` | Mantener — agresividad excepcional del resto |
| `total_shots` | Mantener — **rally length por contexto de resto** |

---

*Fuente: output de `dfs["return_outcomes"].head(24)` ejecutado en `01_exploracion_inicial.ipynb`, diccionario de ReturnDepth (sesión previa), y sistema de codificación del MatchChart.md.*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*

---
---

