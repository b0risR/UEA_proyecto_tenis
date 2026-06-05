# Diccionario de Datos — Tennis Abstract Match Charting Project

**Proyecto:** Scouting táctico Novak Djokovic  
**Fuente:** JeffSackmann/tennis_MatchChartingProject (CC BY-NC-SA 4.0)  
**Objetivo:** Documentar granularidad, campos, uso táctico y limitaciones de cada tabla.

---

# `charting-m-stats-Rally.csv` — Nota de descarte

**Decisión:** tabla **descartada del modelo Power BI**.

---

## Motivo

La tabla Rally mezcla en la misma fila los datos de ambos jugadores mediante las columnas `pl1_*` y `pl2_*`, donde `pl1` es siempre el primer servidor del partido y `pl2` el primer restador. El rol de Djokovic — si es `pl1` o `pl2` — cambia partido a partido sin ninguna columna explícita que lo indique.

Esto impide agregar datos entre partidos de forma directa: cada agregación requiere primero resolver el rol de Djokovic en cada `match_id`, luego seleccionar dinámicamente `pl1_won` o `pl2_won` según corresponda, y repetir esa lógica para cada columna de la tabla. El resultado es un modelo frágil y difícil de mantener.

## Alternativas disponibles en el modelo

Las preguntas tácticas que esta tabla respondería tienen cobertura equivalente o mejor en tablas ya incluidas:

| Pregunta táctica | Fuente alternativa |
|---|---|
| Rally length y puntos ganados por duración | `ReturnOutcomes` — columnas `total_shots / pts` y `pts_won` por `row = "7"`, `"89"`, `"9"` |
| Winners y errores no forzados por partido | `Overview` — columnas `winners`, `unforced`, `winners_fh`, `winners_bh`, `unforced_fh`, `unforced_bh` |
| Puntos resueltos en ≤3 golpes al saque | `ServeBasics` — columna `pts_won_lte_3_shots` |
| Perfil de finalización de punto (forced/unforced) | `Overview` — ratio `winners / unforced` por superficie y resultado |

---

*Decisión tomada durante la fase de exploración de datos (Ejercicio 1).*  
*Datos: JeffSackmann/tennis_MatchChartingProject — CC BY-NC-SA 4.0*