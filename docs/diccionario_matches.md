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