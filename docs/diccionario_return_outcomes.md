# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-ReturnOutcomes.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 subconjunto de restos (por tipo de saque, tipo de golpe, lado/dirección del saque, o profundidad del resto)  
**Columnas:** 10 | **Nulos:** 0 | **Duplicados:** 378  
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