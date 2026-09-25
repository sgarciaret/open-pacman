# SPEC 03 — Power Pellets y Modo Frightened (Fantasmas Vulnerables)

> **Status:** Aprobado
> **Depends on:** SPEC 01, SPEC 02
> **Date:** 2026-09-25
> **Objective:** Incorporar Power Pellets al laberinto para otorgar a Pac-Man la capacidad de volver vulnerables a los fantasmas y devorarlos para obtener puntos escalonados.

---

## 1. Por qué existe esta spec

Actualmente, Pac-Man solo consume dots estándar y es vulnerable a cualquier contacto con los fantasmas. Los **Power Pellets** (o Energizers) son la mecánica icónica de inversión de roles en Pac-Man: permiten a Pac-Man pasar de presa a cazador durante un tiempo limitado, proporcionando una vía crucial de escape, control del mapa y acumulación de puntuación.

---

## 2. Scope

**In:**

- **4 Power Pellets en el laberinto:** Ubicados en las 4 esquinas clásicas:
  - Superior izquierda: `(1, 3)`
  - Superior derecha: `(26, 3)`
  - Inferior izquierda: `(1, 23)`
  - Inferior derecha: `(26, 23)`
- **Nuevo tile tipo `4` ('O'):** Representación en `MAZE_STR` y parseo en `parseTile`.
- **Efecto visual de Power Pellets:** Círculo visiblemente más grande que los dots ordinarios (radio ~6px) con parpadeo rítmico según el frame del juego.
- **Mecánica de consumo del Power Pellet:**
  - Otorga **50 puntos**.
  - Decrementa `dotsRemaining` (necesario para ganar la partida).
  - Activa el modo *Frightened* durante **6 segundos** (o reinicia el contador a 6s si ya estaba activo).
  - Reinicia la racha de fantasmas devorados (`ghostsEaten = 0`).
- **Estado de fantasmas vulnerables (*Frightened*):**
  - Aplica a todos los fantasmas que estén fuera del corral (`inPen === false`).
  - **Velocidad reducida:** Se mueven al 50% de `GHOST_SPEED`.
  - **Coloración azul:** Se dibujan en color azul oscuro arcade (`#2121ff` o `#2121de`) con boca/ojos característicos.
  - **Parpadeo de aviso:** Durante los últimos 2 segundos del efecto, alternan rápidamente entre azul y blanco.
  - **Dirección al activarse:** Invierten su dirección de avance inmediatamente si la celda contraria es transitable.
- **Mecánica de devorar fantasmas:**
  - Si Pac-Man colisiona con un fantasma asustado, Pac-Man no sufre daño y el fantasma es devorado.
  - **Puntuación escalonada:** 
    - 1.º fantasma: 200 pts
    - 2.º fantasma: 400 pts
    - 3.º fantasma: 800 pts
    - 4.º fantasma: 1600 pts
  - **Respawn inmediato:** El fantasma devorado regresa instantáneamente al corral (`inPen: true`), se posiciona en su coordenada inicial de corral con `timer: 0`, reanudando su ciclo de rebote y salida escalonada (especificado en SPEC 02).
- **Fin del modo vulnerable:**
  - Al expirar los 6 segundos, los fantasmas que sigan vivos recuperan su color habitual, su velocidad estándar y su IA de persecución normal.

**Out of scope:**

- Modo "ojos flotantes" navegando de vuelta al corral mediante pathfinding (se implementa el respawn directo en corral).
- Marcador flotante de puntos efímero dibujado sobre el canvas en la coordenada de la colisión.
- Efectos de sonido (no contemplados en la arquitectura actual sin audio).

---

## 3. Data model

### Constantes y Mapa (`src/js/maze.js`)

- Tile `4` para el Power Pellet:
  ```javascript
  function parseTile(ch) {
    if (ch === '#') return 1;
    if (ch === '.') return 2;
    if (ch === '-') return 3;
    if (ch === 'O') return 4; // Power Pellet
    return 0;
  }
  ```
- En `MAZE_STR`, celdas `(1,3)`, `(26,3)`, `(1,23)` y `(26,23)` cambian su caracter de `.` a `O`.

### Estado del Juego (`src/js/game.js`)

Se amplía el objeto devuelto por `createGame`:

```javascript
{
  // Campos existentes...
  frightenedTimer: 0,   // Segundos restantes del modo asustado (0 = inactivo)
  ghostsEaten: 0,       // Cantidad de fantasmas devorados en la racha actual (0 a 4)
}
```

- `dotsRemaining`: contabiliza tanto celdas con valor `2` como celdas con valor `4`.

---

## 4. Implementation plan

1. **Definición de Power Pellets en el Laberinto (`src/js/maze.js`):**
   - Modificar `MAZE_STR` para reemplazar los dots por `'O'` en las posiciones `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`.
   - Ajustar `parseTile` para retornar `4` ante el carácter `'O'`.

2. **Inicialización y conteo en el Game State (`src/js/game.js`):**
   - En `createGame`, contar celdas con valor `2` y `4` para `dotsRemaining`.
   - Inicializar `frightenedTimer: 0` y `ghostsEaten: 0`.

3. **Consumo de Power Pellet en `movePacman` (`src/js/game.js`):**
   - Al alcanzar una celda con valor `4`:
     - Vaciar la celda (`grid[p.y][p.x] = 0`).
     - Sumar 50 puntos a `game.score`.
     - Decrementar `game.dotsRemaining`.
     - Establecer `game.frightenedTimer = 6`.
     - Reiniciar `game.ghostsEaten = 0`.
     - Invertir la dirección de los fantasmas activos fuera del corral (`g.inPen === false`).

4. **Actualización temporal y velocidad en `update` / `moveGhost` (`src/js/game.js`):**
   - Decrementar `game.frightenedTimer` en `1 / 60` cada tick si es mayor a 0.
   - Si `game.frightenedTimer > 0`, la velocidad efectiva de los fantasmas libres es `GHOST_SPEED * 0.5`.
   - Si `game.frightenedTimer <= 0`, restaurar velocidad normal.

5. **Resolución de colisiones Pac-Man vs Fantasma (`src/js/game.js`):**
   - En el bucle de detección de colisiones de `update(game)`:
     - Si colisiona con un fantasma `g`:
       - Si `game.frightenedTimer > 0` y `!g.inPen`:
         - Calcular puntos: `200 * Math.pow(2, Math.min(game.ghostsEaten, 3))`.
         - Incrementar `game.score` y `game.ghostsEaten++`.
         - Regresar a `g` a su corral (`resetGhostToPen(game, g)`), fijando sus coordenadas de `GHOST_STARTS`, `inPen: true`, `timer: 0`, `bounceDir: -1`.
       - Si no está asustado (o `g.inPen` es true): Pac-Man pierde una vida (`game.lives--`) y ejecuta `resetPositions(game)`.

6. **Renderizado de Pellets y Fantasmas (`src/js/render.js`):**
   - En `drawDots`: o crear `drawPellets` que renderice celdas con valor `4` como círculos de radio 6px parpadeantes (usando modulación senoidal o alternancia por frame).
   - En `drawGhost`: si `game.frightenedTimer > 0` y `!g.inPen`, calcular si debe parpadear (si `frightenedTimer <= 2` alternar cada ~10 frames entre blanco y azul) o permanecer en azul arcade (`#2121de`).

7. **Validación y condición de victoria:**
   - Comprobar que al consumir el último dot/pellet se dispara `game.state = 'won'`.

---

## 5. Acceptance criteria

- [ ] Las 4 esquinas clásicas contienen un Power Pellet visiblemente más grande que los dots ordinarios.
- [ ] Los Power Pellets parpadean continuamente en pantalla.
- [ ] Comer un Power Pellet suma 50 puntos y reduce `dotsRemaining`.
- [ ] Al comer un Power Pellet, los fantasmas fuera del corral pasan a color azul y reducen su velocidad al 50%.
- [ ] Durante los últimos 2 segundos del efecto (temporizador <= 2s), los fantasmas alternan entre blanco y azul como advertencia de expiración.
- [ ] Si Pac-Man toca a un fantasma asustado, Pac-Man no pierde vida; el fantasma es devorado y suma 200, 400, 800 o 1600 puntos correlativamente.
- [ ] El fantasma devorado reaparece en el corral y espera su turno de salida escalonada antes de volver a ingresar al laberinto.
- [ ] Si expira el temporizador de 6s, los fantasmas recuperan su color y velocidad original, y volverán a quitar vida si tocan a Pac-Man.
- [ ] Si se come un segundo Power Pellet mientras el primero está activo, el temporizador vuelve a 6s y el multiplicador de puntos se reinicia a 200.
- [ ] La partida se gana correctamente al comer todos los dots y los 4 Power Pellets.
- [ ] No se presentan errores en la consola del navegador.

---

## 6. Decisions taken and discarded

- **Sí: Respawn directo en corral:** Simplifica la implementación evitando cálculos complejos de pathfinding para el estado de ojos regresando al corral, manteniendo compatibilidad total con SPEC 02.
- **Sí: Multiplicador clásico (200, 400, 800, 1600):** Recompensa el riesgo de perseguir a los 4 fantasmas en una sola ráfaga, fiel a la fórmula arcade.
- **Sí: Parpadeo preventivo de 2 segundos:** Proporciona advertencia visual indispensable para que el jugador sepa cuándo retirarse antes de que los fantasmas vuelvan a ser letales.
- **Descartado: Puntuación flotante en canvas:** Se pospone para no sobrecargar el renderizador con colas de entidades temporales flotantes en este spec.

---

## 7. Identified risks

- **Alineamiento de cuadrícula a media velocidad:** Con `speed = GHOST_SPEED * 0.5`, la función de alineación `aligned(val)` en `game.js` debe verificar que los giros en intersecciones sigan reconociéndose correctamente sin desfasar el eje de la celda.
