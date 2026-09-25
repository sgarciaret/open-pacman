# SPEC 02 — Salida escalonada forzada y bloqueo de reentrada al corral de fantasmas

> **Status:** Borrador
> **Depends on:** SPEC 01
> **Date:** 2026-09-25
> **Objective:** Garantizar la salida escalonada fluida de todos los fantasmas con oscilación previa dentro del corral y prohibir estrictamente su reentrada a través de la puerta una vez en el laberinto.

---

## 1. Por qué existe esta spec

En la versión actual, los fantasmas dentro del corral permanecen estáticos hasta su tiempo de salida y, en caso de perturbaciones de alineación o colisión con la puerta (tile `3`), pueden quedar atrapados en el interior. Además, es necesario garantizar que una vez que un fantasma abandone el corral y comience su IA de laberinto, la puerta actúe como un muro unidireccional infranqueable hacia abajo para evitar que reingrese al recinto.

---

## 2. Scope

**In:**

- **Oscilación vertical en el corral:** Pinky, Inky y Clyde rebotan suavemente arriba y abajo dentro de su celda en el corral mientras `timer < exitDelay`.
- **Maniobra forzada y determinista de salida:** Al cumplirse `timer >= exitDelay`, el fantasma se traslada en el eje X hacia la columna central (columna `13`), asciende en el eje Y atravesando la puerta (`y = 12`, tile `3`) hasta colocarse fuera en `(13, 11)`, momento en el que pasa a `inPen = false`.
- **Bloqueo de reentrada al corral (Puerta unidireccional):** Para cualquier fantasma en el laberinto (`inPen === false`), la puerta (tile `3`, fila `12`, columnas `13` y `14`) es tratada como muro infranqueable en `canMove` / `isWall`, evitando que vuelvan a entrar.
- **Reinicio limpio de ronda tras muerte:** Al perder Pac-Man una vida (`resetPositions`), se restablecen las posiciones iniciales, los estados de corral (`inPen: true` para Pinky, Inky, Clyde; `false` para Blinky) y se reinician los temporizadores (`timer = 0`) para repetir la secuencia escalonada (2s, 5s, 8s).

**Out of scope (para futuras specs):**

- Reingreso de fantasmas al corral en modo ojos (*Eaten*) tras ser devorados por consumir un Energizer / bola de poder.
- Modos globales cíclicos de temporización *Chase / Scatter*.
- Velocidad reducida específica para el interior del corral diferente a `GHOST_SPEED`.

---

## 3. Data model

Se extiende el modelo de cada fantasma en `game.ghosts` (`src/js/game.js`):

```javascript
// Cada fantasma dentro de game.ghosts:
{
  id: 'blinky',          // 'blinky' | 'pinky' | 'inky' | 'clyde'
  x: 13,
  y: 11,
  dir: 'left',
  speed: GHOST_SPEED,
  color: '#ff0000',
  inPen: false,          // true mientras esté en el corral o en proceso forzado de salida
  timer: 0,              // Segundos acumulados en el corral
  exitDelay: 0,          // Retardo antes de iniciar la salida (Pinky: 2, Inky: 5, Clyde: 8)
  bounceDir: -1          // Dirección de rebote vertical en el corral (-1 para 'up', 1 para 'down')
}
```

- **Límites de rebote en el corral:**
  - Rango en Y para oscilación: entre `y = 13.5` y `y = 14.5` (centro base en `14`).
  - Velocidad de rebote: la mitad de `GHOST_SPEED` para una oscilación visual suave mientras esperan.

---

## 4. Implementation plan

1. **Ampliación del estado del fantasma con rebote (`src/js/game.js`):**
   - Inicializar `bounceDir: -1` (subiendo) en `createGame` y en `resetPositions` para cada fantasma.
2. **Lógica de oscilación en el corral (`src/js/game.js`):**
   - En `moveGhost`, si `g.inPen === true` y `g.timer < g.exitDelay`, incrementar `g.timer += 1 / 60` y desplazar verticalmente a `g` entre `13.5` y `14.5`, invirtiendo `bounceDir` al tocar los extremos.
3. **Trayectoria garantizada de salida (`src/js/game.js`):**
   - Cuando `g.inPen === true` y `g.timer >= g.exitDelay`:
     - Centrar en X hacia `13` (columna de la puerta).
     - Una vez centrado en X (`Math.abs(g.x - 13) < 1e-3`), forzar movimiento vertical hacia arriba (`dir = 'up'`).
     - Al alcanzar `g.y <= 11`, fijar `g.y = 11`, `g.dir = 'left'`, y desactivar `g.inPen = false`.
4. **Regla de bloqueo unidireccional de puerta (`src/js/game.js`):**
   - Modificar la validación en `isWall` / `canMove`: si el actor es `'ghost'` y `!g.inPen`, cualquier intento de moverse hacia una celda con valor `3` (puerta del pen) o entrar en fila 12 cols 13-14 devuelve `false` (muro).
5. **Verificación y validación de reinicio (`src/js/game.js`):**
   - Verificar que al invocar `resetPositions` los fantasmas regresan a sus posiciones y estados iniciales definidos en `GHOST_STARTS`.

---

## 5. Acceptance criteria

- [ ] Blinky inicia fuera del corral en (13, 11) patrullando inmediatamente con su IA de persecución.
- [ ] Pinky, Inky y Clyde oscilan verticalmente (arriba y abajo) en sus celdas del corral mientras esperan su tiempo de salida.
- [ ] A los ~2s Pinky se alinea a `x = 13`, sube por la puerta hasta `(13, 11)` y se incorpora al laberinto.
- [ ] A los ~5s Inky se desplaza hacia `x = 13`, sube por la puerta hasta `(13, 11)` y se incorpora al laberinto.
- [ ] A los ~8s Clyde se desplaza hacia `x = 13`, sube por la puerta hasta `(13, 11)` y se incorpora al laberinto.
- [ ] Ningún fantasma se atasca en los muros del corral ni en la puerta durante el ascenso.
- [ ] Ningún fantasma que ya esté fuera del corral (`inPen === false`) puede cruzar la puerta hacia abajo o reingresar al recinto.
- [ ] Al perder Pac-Man una vida, los 4 fantasmas vuelven a sus puestos iniciales y se repite la secuencia de salida escalonada.
- [ ] No se producen errores en la consola del navegador.

---

## 6. Decisions taken and discarded

- **Sí: Rebote suave dentro del corral:** Proporciona retroalimentación visual al jugador de que los fantasmas están activos y esperando su salida, recreando fielmente la experiencia arcade.
- **Sí: Puerta infranqueable para fantasmas libres (`inPen === false`):** Evita que los fantasmas reingresen accidentalmente por la puerta del corral durante su navegación habitual por el laberinto.
- **Descartado: Modos Chase/Scatter globales:** Se posterga para una spec posterior con el fin de mantener el alcance enfocado exclusivamente en la corrección del flujo del corral y la salida.
