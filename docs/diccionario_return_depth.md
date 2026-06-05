# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-ReturnDepth.csv` — Significado de columnas

**Granularidad:** 1 fila = 1 jugador × 1 combinación (saque × tipo de golpe × lado de pista × dirección de saque)  
**Columnas:** 11 | **Nulos:** 0 | **Duplicados:** 323  
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