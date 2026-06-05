# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

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