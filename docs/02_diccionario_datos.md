# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

## Resumen del modelo de datos

| Tabla | Archivo CSV | Granularidad | Clave principal | Dimensión táctica |
|---|---|---|---|---|
| Partidos | charting-m-matches.csv | 1 fila = 1 partido | match_id | Contexto general |
| Overview | charting-m-stats-Overview.csv | 1 fila = 1 jugador × partido | match_id + player | General |
| ServeBasics | charting-m-stats-ServeBasics.csv | 1 fila = 1 jugador × partido | match_id + player | **Saque** |
| ServeDirection | charting-m-stats-ServeDirection.csv | 1 fila = 1 jugador × partido | match_id + player | **Saque** |
| ReturnDepth | charting-m-stats-ReturnDepth.csv | 1 fila = 1 jugador × partido | match_id + player | **Resto** |
| ReturnOutcomes | charting-m-stats-ReturnOutcomes.csv | 1 fila = 1 jugador × partido | match_id + player | **Resto** |
| Rally | charting-m-stats-Rally.csv | 1 fila = 1 jugador × partido | match_id + player | **Rally** |
| NetPoints | charting-m-stats-NetPoints.csv | 1 fila = 1 jugador × partido | match_id + player | **Red** |
| KeyPointsServe | charting-m-stats-KeyPointsServe.csv | 1 fila = 1 jugador × partido | match_id + player | **Presión** |
| KeyPointsReturn | charting-m-stats-KeyPointsReturn.csv | 1 fila = 1 jugador × partido | match_id + player | **Presión** |

**Nota sobre granularidad:** Todas las tablas de estadísticas son datos **agregados por partido y jugador** (no punto a punto). Para análisis punto a punto se necesitaría `charting-m-points.csv`, que no está en el scope de este ejercicio.

---

## 1. charting-m-matches.csv

**Granularidad:** 1 fila = 1 partido  
**Clave de unión:** `match_id` — identificador único del partido, formato `AAAAMMDD-G-Torneo-Ronda-Jugador1-Jugador2`  
**Uso táctico:** Tabla de dimensión principal. Permite filtrar por superficie, año, torneo y rival.  
**Limitaciones:** Algunos partidos históricos pueden tener campos incompletos (ranking, resultado por sets).

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Identificador único del partido. Formato: `AAAAMMDD-G-Torneo-Ronda-P1-P2`. G=M (masculino). Clave de unión con todas las tablas de estadísticas. | Clave de join |
| Player 1 | string | Nombre del jugador que sacó primero en el partido. En todo el dataset, Player1 es **siempre** el primer saque. | Identificación |
| Player 2 | string | Nombre del jugador que restó primero. | Identificación |
| Ranking 1 | integer | Ranking ATP de Player1 en el momento del partido. Puede estar vacío en partidos históricos. | Contexto |
| Ranking 2 | integer | Ranking ATP de Player2 en el momento del partido. | Contexto |
| Score | string | Resultado final del partido en formato sets (ej: `6-3 7-6(4)`). | Resultado |
| Surface | string | Superficie: `Hard`, `Clay`, `Grass`, `Carpet`. Factor clave para segmentación táctica. | **Filtro crítico** |
| Tournament | string | Nombre del torneo (ej: `Wimbledon`, `Roland Garros`). | Filtro |
| Round | string | Ronda: `F` (final), `SF` (semifinal), `QF` (cuartos), `R16`, `R32`, `R64`, `R128`, `RR` (round-robin). | Filtro presión |
| Date | string/integer | Fecha del partido. Formato AAAAMMDD en el match_id; en este campo puede venir como string. Requiere parsing. | Temporal |
| Num Sets | integer | Número de sets jugados en el partido. | Contexto |
| Completed | boolean/integer | Indica si el partido se completó (`1`) o fue abandonado/walkover (`0`). Registros con 0 deben excluirse del análisis. | **Control calidad** |
| Comment | string | Notas libres sobre el partido (condiciones especiales, retiradas, etc.). | Contextual |

---

## 2. charting-m-stats-Overview.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido (2 filas por partido)  
**Clave de unión:** `match_id` + `player` (1 = Player1, 2 = Player2)  
**Uso táctico:** Visión global del partido. Permite calcular winners, errores y puntos totales.  
**Limitaciones:** Los valores son conteos absolutos (no porcentajes). Hay que calcular los ratios manualmente.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión con matches. | Join |
| player | integer | `1` = Player1 (primer saque), `2` = Player2. | Filtro jugador |
| row | string | Etiqueta de fila interna del sistema de charting (`serve`, `return`, `total`). | Segmentación |
| pts | integer | Total de puntos jugados por este jugador en este contexto (saque, resto o total). | Base cálculos |
| pts_won | integer | Puntos ganados. Junto con `pts`, permite calcular `% puntos ganados`. | **KPI principal** |
| winners | integer | Golpes ganadores directos (excluye aces). | Agresividad |
| unforced | integer | Errores no forzados cometidos. Indicador de consistencia. | Consistencia |
| forced | integer | Errores forzados (el rival forzó el error). | Presión recibida |
| aces | integer | Aces realizados. | Dominancia al saque |
| double_faults | integer | Dobles faltas. Vulnerabilidad al saque. | **Riesgo saque** |
| unret | integer | Saques no devueltos (ace + servicio directo sin contacto del restador). | Efectividad saque |

---

## 3. charting-m-stats-ServeBasics.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Análisis del rendimiento al saque: efectividad de primer y segundo saque.  
**Limitaciones:** Conteos absolutos; requiere calcular porcentajes con denominadores correctos.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Etiqueta de contexto. | Segmentación |
| s1_in | integer | Primeros saques que entraron. Numerador de `% 1er saque dentro`. | **% 1S dentro** |
| s1_total | integer | Total de primeros saques intentados. Denominador. | Base |
| s1_won | integer | Puntos ganados cuando el primer saque entró. | **% ganados con 1S** |
| s2_in | integer | Segundos saques que entraron (excluyendo dobles faltas). | % 2S dentro |
| s2_total | integer | Total de segundos saques intentados. | Base |
| s2_won | integer | Puntos ganados cuando el segundo saque entró. | **% ganados con 2S** |
| df | integer | Dobles faltas totales. Pérdida directa de punto. | Riesgo |
| aces | integer | Aces totales. | Dominancia |
| serve_pts | integer | Total de puntos al saque. Denominador global. | Base |
| serve_pts_won | integer | Total de puntos ganados al saque. | **KPI saque** |

---

## 4. charting-m-stats-ServeDirection.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Tendencias de dirección de saque. Permite construir el heatmap de saque por zona.  
**Limitaciones:** Datos agregados por partido, no por punto. No permite ver variación dentro del partido.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Contexto de saque: `wide_deuce`, `body_deuce`, `T_deuce`, `wide_ad`, `body_ad`, `T_ad`. Deuce = lado par; Ad = lado impar. | **Heatmap saque** |
| s1 | integer | Primeros saques dirigidos a esa zona. | Frecuencia 1S |
| s1_won | integer | Puntos ganados con primer saque en esa zona. | Efectividad 1S |
| s2 | integer | Segundos saques dirigidos a esa zona. | Frecuencia 2S |
| s2_won | integer | Puntos ganados con segundo saque en esa zona. | Efectividad 2S |

**Nota sobre zonas:**
- `wide`: saque hacia el exterior de la pista (al revés del rival diestro)
- `body`: saque directo al cuerpo del restador
- `T`: saque hacia el centro (línea T), el más recto
- `deuce`: lado derecho de la pista (punto par: 0-0, 30-0, 0-30, 40-40...)
- `ad`: lado izquierdo de la pista (punto impar: 15-0, 0-15, 40-30, 30-40...)

---

## 5. charting-m-stats-ReturnDepth.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Profundidad del resto devuelto. Mide la agresividad posicional del restador.  
**Limitaciones:** Clasificación subjetiva del chárter. La zona `deep` no siempre implica calidad táctica.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Tipo de saque recibido: `s1` (primer saque) o `s2` (segundo saque). | Segmentación |
| shallow | integer | Restos que aterrizaron en el tercio corto de la pista (zona de servicio del restador). Indica posición defensiva o saque muy agresivo. | Pasividad |
| medium | integer | Restos en la zona media. Resto neutro, ni agresivo ni defensivo. | Neutro |
| deep | integer | Restos que llegaron al fondo de la pista (último tercio). Indica presión sobre el servidor. | **Agresividad** |
| total | integer | Total de restos en juego. Base para calcular porcentajes de profundidad. | Denominador |
| pts_won | integer | Puntos ganados cuando el resto entró en juego. | Efectividad |

---

## 6. charting-m-stats-ReturnOutcomes.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Resultado inmediato del resto. Permite medir consistencia vs agresividad al devolver.  
**Limitaciones:** No distingue calidad del saque recibido. Un error contra un ace no es lo mismo que un error contra un segundo saque débil.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Tipo de saque: `s1` o `s2`. | Segmentación |
| winners | integer | Restos ganadores directos (winner de resto). Alta agresividad. | Agresividad |
| unforced | integer | Errores no forzados al restar. Pérdida directa de punto por fallo propio. | **Consistencia** |
| forced | integer | Errores forzados al restar (el saque forzó el fallo). | Calidad saque rival |
| in_play | integer | Restos que entraron en juego (rally continuó). Principal indicador de consistencia. | **% restos en juego** |
| total | integer | Total de restos intentados. Incluye errores y en juego, excluye los no devueltos. | Denominador |
| pts_won | integer | Puntos ganados en los rallies iniciados con resto en juego. | Efectividad resto |

---

## 7. charting-m-stats-Rally.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Rendimiento por duración de rally. Identifica si el jugador prefiere puntos cortos o largos.  
**Limitaciones:** Los buckets son fijos y pueden no capturar bien la transición táctica entre tramos. El conteo incluye el saque como primer golpe.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Bucket de duración: `1_3` (1-3 golpes), `4_6` (4-6), `7_9` (7-9), `10_` (10 o más). El conteo incluye el saque. | **Rally length chart** |
| pts | integer | Puntos jugados en ese bucket de rally. | Frecuencia |
| pts_won | integer | Puntos ganados en ese bucket. Dividir entre `pts` da el `% ganados por longitud`. | **% por rally** |

**Interpretación de buckets:**
- `1_3`: Puntos decididos en el saque (aces, dobles faltas, restos fallados, saques no devueltos, winners inmediatos)
- `4_6`: Rally corto — transición agresiva tras el saque
- `7_9`: Rally medio — construcción del punto
- `10_`: Rally largo — resistencia física y mental, especialidad de Djokovic en arcilla

---

## 8. charting-m-stats-NetPoints.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Frecuencia y efectividad en la red. Indica si el jugador es "de fondo" o "de transición".  
**Limitaciones:** No distingue entre subida voluntaria (táctica) y subida forzada (por bola corta del rival).

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Contexto de la subida a la red. | Segmentación |
| net_pts | integer | Total de puntos jugados en la red (el jugador llegó a la red). | Frecuencia de subida |
| pts_won | integer | Puntos ganados en la red. `pts_won / net_pts` = `% efectividad en red`. | **% red ganados** |
| net_winner | integer | Puntos acabados con winner desde la red (volea ganadora, smash, etc.). | Calidad en red |
| induced_forced | integer | Puntos ganados porque el rival cometió error forzado al intentar pasar. | Presión en red |
| net_unforced | integer | Errores no forzados propios en la red (volea fallada sin presión). | Vulnerabilidad red |
| passed_at_net | integer | Puntos perdidos porque el rival ejecutó un passing shot exitoso. | **Riesgo de subir** |
| passing_shot_induced_forced | integer | Intentos de passing del rival que resultaron en error forzado (el jugador en red forzó el error). | Presión defensiva |
| total_shots | integer | Total de golpes jugados en los puntos de red. Métrica auxiliar. | Contexto |

---

## 9. charting-m-stats-KeyPointsServe.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Rendimiento al saque en situaciones de máxima presión. Fundamental para evaluar mentalidad competitiva.  
**Limitaciones:** Volumen de puntos clave por partido es bajo (pocas observaciones), lo que introduce varianza alta. Hay que agregar muchos partidos para obtener conclusiones fiables.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Tipo de punto clave: `bp_faced` (break points en contra), `deuce`, `set_pts_served` (set points a favor), `match_pts_served` (match points a favor). | **Presión al saque** |
| pts | integer | Total de puntos clave al saque de ese tipo. | Volumen |
| pts_won | integer | Puntos clave ganados al saque. `pts_won / pts` = `% salvados` (en BP) o `% convertidos` (en set/match point). | **% BP salvados** |

**Puntos clave definidos:**
- **Break point (bp_faced):** El restador tiene oportunidad de romper el saque. Si Djokovic saca, es una situación de peligro.
- **Deuce:** Marcador igualado en el juego (40-40). Cada punto siguiente decide el juego.
- **Set point:** Un jugador tiene oportunidad de ganar el set.
- **Match point:** Un jugador tiene oportunidad de ganar el partido.

---

## 10. charting-m-stats-KeyPointsReturn.csv

**Granularidad:** 1 fila = 1 jugador × 1 partido  
**Clave de unión:** `match_id` + `player`  
**Uso táctico:** Rendimiento al resto en puntos clave. Complementa la tabla anterior desde el otro lado de la red.  
**Limitaciones:** Igual que KeyPointsServe — muestra pequeña por partido. Necesita agregación sobre muchos partidos.

| Columna | Tipo | Descripción | Uso táctico |
|---|---|---|---|
| match_id | string | Clave de unión. | Join |
| player | integer | `1` o `2`. | Filtro |
| row | string | Tipo de punto clave: `bp_opps` (break points a favor), `deuce`, `set_pts_returned` (set points en contra), `match_pts_returned` (match points en contra). | **Presión al resto** |
| pts | integer | Total de puntos clave al resto de ese tipo. | Volumen |
| pts_won | integer | Puntos clave ganados al resto. `pts_won / pts` = `% BP convertidos` (si restamos). | **% BP convertidos** |

**Puntos clave definidos (perspectiva del restador):**
- **Break point (bp_opps):** El restador (Djokovic) tiene oportunidad de romper. Indicador de efectividad ofensiva al resto.
- **Deuce:** Mismo marcador 40-40, ahora Djokovic restando.
- **Set point en contra:** El rival sirve para el set; Djokovic restando.
- **Match point en contra:** El rival sirve para el partido; Djokovic restando.

---

## Matriz táctica: qué tabla responde a cada pregunta

| Pregunta táctica | Tabla(s) necesaria(s) | Columnas clave |
|---|---|---|
| ¿Dónde saca Djokovic en deuce y advantage? | ServeDirection | `row` (zona), `s1`, `s2`, `s1_won`, `s2_won` |
| ¿Rinde menos con el segundo saque? | ServeBasics | `s1_won/s1_in` vs `s2_won/s2_in` |
| ¿Cómo neutraliza el saque rival? | ReturnDepth + ReturnOutcomes | `deep`, `in_play`, `unforced` |
| ¿Qué duración de rally le favorece? | Rally | `pts_won/pts` por bucket |
| ¿Rinde bajo presión? | KeyPointsServe + KeyPointsReturn | `pts_won/pts` en BP, deuce |
| ¿La superficie cambia su perfil? | Todas + matches (filtro Surface) | `Surface` como segmentador |
| ¿Qué diferencia victorias de derrotas? | Todas + matches (filtro Score) | Resultado + KPIs por tabla |
| ¿Sube a la red? ¿Con qué resultado? | NetPoints | `net_pts`, `pts_won`, `passed_at_net` |

---

## Preguntas que NO pueden responderse con datos agregados (requieren punto a punto)

- Variación táctica dentro de un partido (ajustes set a set)
- Dirección exacta de cada golpe de fondo (forehand/backhand cruzado vs paralelo)
- Secuencias de 3 golpes (serve + approach + volley)
- Posición en pista en cada momento del rally
- Velocidad de saque (no está en charting, está en datos ATP oficiales)

---

## Filtros mínimos recomendados en todo el informe

Para no mezclar contextos tácticos incompatibles, todo análisis debe aplicar como mínimo:

1. **Jugador = Djokovic** — filtrar `Player 1` o `Player 2` según en qué lado jugó en cada partido.
2. **Completed = 1** — excluir partidos abandonados o walkover.
3. **Surface** — nunca agregar hard + clay + grass sin segmentar; los perfiles tácticos cambian radicalmente.
4. **Ventana temporal** — se recomienda separar carrera completa vs últimas temporadas (sugerencia: desde 2018 como período reciente dado el volumen disponible).

---

*Fuentes: JeffSackmann/tennis_MatchChartingProject README, data_dictionary.txt del proyecto, Tennis Abstract match reports y documentación del charting notation.*  
*Licencia datos: CC BY-NC-SA 4.0 — uso académico no comercial, atribución requerida.*
