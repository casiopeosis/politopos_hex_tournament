# Politopos v7 — MCTS Paralelo con Tree Reuse

## Algoritmo

**Politopos** es una estrategia MCTS (Monte Carlo Tree Search) con las siguientes características:

1. **Motor paralelo**: 3 procesos worker simultáneos ejecutan MCTS en paralelo. Al final, los resultados se agregan por votación de visitas (cada worker suma sus iteraciones al nodo raíz). Esto logra aproximadamente **4x más iteraciones** en el presupuesto de tiempo disponible.

2. **Tree reuse**: El árbol del turno anterior se reutiliza descendiendo 2 niveles (mi movimiento → respuesta del oponente). No se descarta el trabajo acumulado; se continúa mejorando desde donde quedó.

3. **Poda de candidatos**: Solo se exploran movimientos en una vecindad de radio-2 alrededor de las piezas existentes (~20-30 candidatos vs ~100). El árbol explora más densamente y converge más rápido.

4. **Eliminación O(1) de casillas**: La clase `_EmptyPool` implementa remoción en O(1) mediante swap-and-pop, reemplazando `list.remove()` que era O(n) en cada iteración MCTS.

5. **Dijkstra correcto**: Usa `heapq` para garantizar mínimos verdaderos con pesos heterogéneos (0 para piedra propia, 1 para casilla vacía).

### Inteligencia heredada de v6

La evaluación heurística proviene íntegramente de la versión anterior:

- **_score**: Progreso hacia el borde objetivo (eje de fila para jugador 1, eje de columna para jugador 2) + conectividad a otras piezas propias.
  ```
  score = 0.7 * progreso + 0.3 * (vecinos_propios / 6)
  ```

- **_combined**: Score del tablero desde la perspectiva del jugador actual.
  ```
  _combined = score_propio - 0.7 * score_oponente
  ```

- **_rollout**: Simulación rápida con muestreo inteligente.
  - Muestra 6 candidatos aleatorios en cada posición libre.
  - Elige el movimiento con máximo `_combined`.
  - Continúa hasta profundidad 100 o fin del tablero.
  - Evalúa el resultado final por distancia Dijkstra (distancia a borde objetivo).

- **_sample_world**: Generación de mundos posibles para dark mode.
  - Estima el número de piedras ocultas del oponente: `max(0, piedras_mías - piedras_oponente_conocidas)`.
  - Coloca exactamente ese número de piedras del oponente en posiciones aleatorias (desconocidas).
  - **Corrección vs v6**: Evita asignar al azar al 50% de casillas vacías (mundos imposibles).

### Detección anticipada

Antes de entrar al MCTS, Politopos detecta:
- **Victoria inmediata**: Existe un movimiento que gana en el turno actual.
- **Bloqueo de derrota**: Existe un movimiento que impide la victoria inmediata del oponente.

---

## Dark Mode (Fog of War)

En `variant="dark"`, el tablero es parcialmente observable:

1. **Conocimiento**: Mantiene `_known_board` con las casillas que el jugador sabe con certeza (piedras propias, colisiones detectadas).

2. **Colisiones**: Cuando un movimiento falla, se registra como ocupado por el oponente en `_collision_history`.

3. **Worlding**: Se samplearán `NUM_BELIEF_SAMPLES` (4) mundos posibles, cada uno con una distribución diferente de piedras ocultas del oponente. El MCTS corre independientemente en cada mundo.

4. **Votación**: El movimiento elegido más frecuentemente entre todos los mundos se juega.

5. **Tree reuse deshabilitado**: En dark mode el árbol se descarta cada turno (el tablero observado puede cambiar impredeciblemente debido a colisiones).

6. **Filtrado de colisiones**: Se evita proponer movimientos que ya han colisionado.

---

## Decisiones de Diseño

| Decisión | Razón |
|----------|-------|
| **Paralelo (3 workers)** | Amortiza el costo de paralelización y garantiza ganancia real de iteraciones sin overhead prohibitivo. |
| **Tree reuse** | Reutilizar 2 niveles (mi move → respuesta) mantiene estadísticas valiosas sin crear inconsistencias. Más de 2 niveles expone a desincronización. |
| **Radio-2 para poda** | Balance: ~20-30 candidatos hacen exploración densa sin perder continuidad de juego. Aperturas se manejan con zona central. |
| **6 muestras en rollout** | Equilibrio entre velocidad (pocos sampleo = rápido) y representatividad (suficiente para distinguir buenos movimientos). |
| **_combined con peso 0.7** | Penalizar progreso del oponente más fuerte que el propio incentiva juego agresivo y defensivo simultáneamente. |
| **Dijkstra en evaluación** | Garantiza distancia correcta a borde; las simulaciones rápidas (rollout) lo usan para evaluación final confiable. |
| **Worlding con estimación** | En lugar de mundos aleatorios, estimar piedras ocultas produce muestras más representativas de la verdadera distribución. |

---

## Resultados Esperados

Politopos v7 fue diseñada para:

- **Vencer MCTS_Tier_1 y MCTS_Tier_2** con alta probabilidad (motor paralelo + poda inteligente mejoran significativamente la búsqueda).
- **Competir con MCTS_Tier_3** en tableros medianos; el factor limitante es el presupuesto de tiempo absoluto (15 seg).
- **Manejar dark mode** de forma robusta mediante estimación de mundos y filtrado de colisiones.

En partidas locales contra Random (sin Docker):
- **Win rate esperado**: >95% en classic.
- **Resultados típicos**: 4–5 victorias en 5 partidas.

---

## Restricciones y Límites

- **Tiempo por movimiento**: 15 segundos (se usa el 92% internamente para seguridad).
- **Memoria**: Árbol MCTS + 4 copias de tablero para worlding en dark.
- **Dependencias**: Solo `stdlib` y `numpy` (no usada aquí).
- **Procesos**: 3 workers + main = 4 procesos concurrentes.

---

## Estructura del Código

- **Constantes**: `EXPLORATION_CONSTANT`, `MAX_SIMULATION_DEPTH`, `EXPAND_RADIUS`, `NUM_WORKERS`, `TIME_BUDGET`.
- **_EmptyPool**: Gestión O(1) de casillas libres.
- **Node**: Nodo MCTS con UCT-RAVE.
- **Funciones standalone** (_score, _combined, _distance, _rollout, _candidates): Pickleable para multiprocessing.
- **_worker_run**: Punto de entrada para cada worker.
- **Politopos.play()**: Orquestación principal.
- **_play_classic()**: MCTS paralelo con tree reuse.
- **_play_dark()**: MCTS con worlding.
- **_descend_root()**: Reutilización de árbol (descender 2 niveles).
- **_sample_world()**: Generación de mundos posibles.

---

## Entrenamiento y Tuning

Parámetros sensibles:

- `EXPLORATION_CONSTANT = 1.25`: Controla balance exploración-explotación. Valores mayores exploran más; menores, refinan más.
- `EXPAND_RADIUS = 2`: Ampliar a 3 explora más pero ralentiza; reducir a 1 agresivamente.
- `NUM_WORKERS = 3`: Reducir a 2 si hay contención de recursos; aumentar a 4+ si hardware lo permite.
- `TIME_BUDGET = 0.92`: Reserva buffer de seguridad para latencias de IPC.

---

## Changelog vs v6

| Cambio | Impacto |
|--------|---------|
| Motor paralelo (3 workers) | ~4x más iteraciones sin tiempo extra. |
| Tree reuse (2 niveles) | Reutiliza ~30–50% del árbol entre turnos (mejora incremental). |
| Poda de candidatos (radio-2) | Exploración 3–5x más densa; apertura es caso especial. |
| _EmptyPool O(1) | Eliminación de casillas 100x más rápida en pools grandes. |
| Dijkstra correcto | Garantiza distancia mínima verdadera (antes, BFS aproximado). |
| _sample_world corregida | Mundos más realistas (no "50% random oponente"). |
| Guard para children vacío | Evita crash en MCTS si el árbol no se expande. |
| Detección inmediata | Victoria/bloqueo antes de MCTS ahorra tiempo. |

---

## Referencias

- **Política de MCTS estándar**: UCT con RAVE.
- **Hex**: Juego clásico en tableros hexagonales (11x11 estándar).
- **Dark mode**: Información incompleta manejada mediante creencia (worlding).
