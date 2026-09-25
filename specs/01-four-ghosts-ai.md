# SPEC 01 — 4 fantasmas con IA clásica diferenciada y salida escalonada

> **Status:** Aprobado
> **Depends on:** None
> **Date:** 2026-09-25
> **Objective:** Incorporar cuatro fantasmas (Blinky, Pinky, Inky y Clyde) con sus respectivas personalidades clásicas de persecución y un sistema de salida escalonada del corral.

---

## Scope

**In:**

- Cuatro fantasmas simultáneos en el juego con sus colores clásicos:
  - **Blinky** (Rojo): Perseguidor agresivo directo.
  - **Pinky** (Rosa): Emboscador anticipado.
  - **Inky** (Cian): Flanqueador con estrategia combinada de pinza.
  - **Clyde** (Naranja): Tímido / errático según proximidad.
- Posición inicial diferenciada: Blinky comienza fuera del corral listo para la persecución; Pinky, Inky y Clyde inician en el interior del corral.
- Salida escalonada del corral mediante temporizador individual (Pinky a los ~2s, Inky a los ~5s, Clyde a los ~8s).
- IA de navegación en intersecciones según celda objetivo (*target tile*) para cada personalidad.
- Detección de colisiones y Game Over con cualquiera de los 4 fantasmas.
- Renderizado de los 4 fantasmas con su color representativo.

**Out of scope (para futuras specs):**

- Modos globales cíclicos Chase/Scatter cronometrados para todo el laberinto.
- Bolas de poder (*Energizers / Power Pellets*) y modo asustado (*Frightened* azul).
- Consumo de fantasmas y regreso de ojos al corral.
- Animación direccional de las pupilas según el vector de movimiento.

---

## Data model

```javascript
// GHOST_STARTS en src/js/maze.js
const GHOST_STARTS = [
  { id: 'blinky', color: '#ff0000', x: 13, y: 11, dir: 'left', inPen: false, exitDelay: 0 },
  { id: 'pinky',  color: '#ffb8ff', x: 13, y: 14, dir: 'up',   inPen: true,  exitDelay: 2 },
  { id: 'inky',   color: '#00ffff', x: 12, y: 14, dir: 'up',   inPen: true,  exitDelay: 5 },
  { id: 'clyde',  color: '#ffb852', x: 15, y: 14, dir: 'up',   inPen: true,  exitDelay: 8 },
];

// Estructura de cada fantasma en game.ghosts (src/js/game.js)
{
  id: 'blinky',          // 'blinky' | 'pinky' | 'inky' | 'clyde'
  x: 13,
  y: 11,
  dir: 'left',
  speed: GHOST_SPEED,
  color: '#ff0000',
  inPen: false,
  timer: 0,              // Segundos acumulados para controlar salida
  exitDelay: 0           // Segundos antes de abandonar la pen
}
```

Conventions:
- Coordenadas: origen arriba-izquierda (x: col 0..27, y: fila 0..30).
- Velocidades en celdas por fotograma (`GHOST_SPEED`).

---

## Implementation plan

1. **Actualizar configuración de fantasmas (`src/js/maze.js`):**
   - Definir `GHOST_STARTS` con los 4 fantasmas (Blinky, Pinky, Inky, Clyde), especificando coordenadas iniciales, color, dirección inicial y retraso de salida.
2. **Inicialización y ciclo de salida del corral (`src/js/game.js`):**
   - En `createGame`, inicializar el array de fantasmas con `timer: 0` e `inPen`.
   - En la actualización, sumar tiempo a los fantasmas que estén en el pen. Al cumplir `timer >= exitDelay`, guiar al fantasma verticalmente hacia la celda de salida (13, 11) atravesando la puerta (tile 3) y marcar `inPen = false`.
3. **Algoritmos de selección de objetivo por IA (`src/js/game.js`):**
   - Implementar función `getGhostTarget(game, ghost)` que devuelva `{ x, y }`:
     - **Blinky:** `(pacman.x, pacman.y)`.
     - **Pinky:** 4 casillas delante de Pac-Man según `pacman.dir`.
     - **Inky:** Calcular punto 2 casillas delante de Pac-Man, trazar vector desde Blinky hasta ese punto y duplicarlo.
     - **Clyde:** Si distancia euclídea a Pac-Man > 8 casillas, objetivo Pac-Man; si <= 8 casillas, objetivo su esquina inferior izquierda `(0, 30)`.
   - Actualizar `decideGhost` para elegir el movimiento legal (sin giro de 180° salvo callejón) que minimice la distancia al objetivo calculado.
4. **Renderizado de fantasmas (`src/js/render.js`):**
   - Modificar la llamada de renderizado para asegurar que `drawGhost` utilice el color asignado a cada fantasma (`g.color`).

---

## Acceptance criteria

- [ ] Se muestran 4 fantasmas en pantalla con sus colores distintivos (rojo, rosa, cian, naranja).
- [ ] Blinky comienza patrullando fuera del corral en (13, 11) y persigue agresiva y directamente a Pac-Man.
- [ ] Pinky, Inky y Clyde esperan dentro de la pen y salen en orden escalonado (~2s, ~5s, ~8s).
- [ ] Al salir del corral, los fantasmas ascienden por la puerta (tile 3) sin atascarse antes de pasar a su IA de laberinto.
- [ ] Pinky intenta anticiparse buscando posiciones al frente del movimiento de Pac-Man.
- [ ] Inky presenta comportamiento de flanqueo coordinado con Blinky y Pac-Man.
- [ ] Clyde persigue a distancia pero rehúye hacia la esquina inferior izquierda al aproximarse a menos de 8 celdas de Pac-Man.
- [ ] El contacto de Pac-Man con cualquiera de los 4 fantasmas provoca la pérdida de vidas / Game Over.
- [ ] No se producen errores ni advertencias en la consola de JavaScript.

---

## Decisions

- **Sí: Comportamiento clásico original de Namco:** Se preserva la esencia identitaria de Blinky (caza directa), Pinky (emboscada), Inky (pinza) y Clyde (distancia crítica), aportando una experiencia de juego rica y no repetitiva.
- **No: Movimiento aleatorio para todos:** Descartado para cumplir el requisito de personalidades y evitar que el juego se sienta caótico sin propósito.
- **Sí: Salida escalonada:** Permite que Pac-Man tenga espacio de maniobra al inicio de la ronda y evita aglomeraciones en la puerta del corral.
- **Sí: Velocidad uniforme (`GHOST_SPEED`):** Mantiene la simetría y balance del juego base sin alterar la física de colisiones ni el refresco de celdas.

---

## Risks

| Risk | Mitigation |
| --- | --- |
| Atasco en la puerta del corral al salir varios fantasmas | Los fantasmas en estado de salida (`inPen = true` con tiempo cumplido) se fuerzan a subir hacia la celda (13, 11) ignorando giros laterales hasta haber cruzado completamente el tile 3. |
