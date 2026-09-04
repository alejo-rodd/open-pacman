# Spec 01 — Cuatro fantasmas con personalidades distintas

**Estado:** Approved
**Fecha:** 2026-09-03
**Depende de:** —
**Objetivo (1 frase):** Ampliar de 2 a 4 fantasmas con personalidades diferenciadas
(perseguidor, emboscador, flanqueador, errático), distinguidos solo por color, con
salida escalonada de la pen cada 1.5 segundos y lógica clásica de objetivo por
fantasma.

## Scope

**Dentro:**
- 4 fantasmas en lugar de 2, cada uno con un `kind` distinto y un color distinto.
- Lógica de IA por `kind` (persigue, embosca, flanquea, errático) basada en la
  lógica clásica de Pac-Man.
- Salida escalonada de la pen: perseguidor → emboscador → flanqueador → errático
  a los 0 / 1.5 / 3 / 4.5 s.
- Movimiento intra-pen (mini-IA) para que cada fantasma camine hasta la puerta
  cuando le toque salir.
- Reset al perder una vida: vuelve a los 4 a la pen y reinicia el contador de
  salida escalonada.
- Misma velocidad para todos los fantasmas (`GHOST_SPEED = 0.1`).

**Fuera:**
- Power-pellets y estado `frightened` (los 4 nunca huyen ni son comibles).
- Diferenciación visual más allá del color.
- Velocidades distintas por fantasma.
- Variación de velocidad del perseguidor según distancia.
- Colisiones fantasma-fantasmo (siguen pudiéndose superponer).
- Sonidos, niveles extra, high-score persistente.

## Modelo de datos

**Nuevos valores de `kind`** (en `game.js`, ramifica `decideGhost`):

- `'perseguidor'` — apunta a la celda actual de Pac-Man (Manhattan, igual que el
  `hunter` actual).
- `'emboscador'` — apunta a 4 celdas delante de Pac-Man según `pacman.dir`;
  clampear a la última celda transitable si choca contra muro.
- `'flanqueador'` — objetivo = `2 * pacman - perseguidor`. Si el perseguidor
  está dentro de la pen, usa `(15, 11)` como referencia.
- `'erratico'` — si `dist(perseguidor, pacman) > 8` (Manhattan) persigue a
  Pac-Man; en otro caso va a la esquina inferior-izquierda `(1, 29)`.

**`GHOST_STARTS`** (en `maze.js`): pasa de 2 a 4 entradas dentro de la pen
(fila 14, columnas 12-15):

```js
const GHOST_STARTS = [
  { x: 12, y: 14, kind: 'perseguidor' },
  { x: 13, y: 14, kind: 'emboscador'  },
  { x: 14, y: 14, kind: 'flanqueador' },
  { x: 15, y: 14, kind: 'erratico'    },
];
```

**Campo nuevo en cada fantasma**: `releaseAt` — timestamp en ms (`performance.now()`)
en el que debe empezar a caminar hacia la puerta. Se inicializa en `createGame`
y se reinicia en `resetPositions`.

**Mapeo color por `kind`** (en `render.js`): se introduce `GHOST_COLOR_BY_KIND`
para que el color quede atado al arquetipo y no al índice:

```js
const GHOST_COLOR_BY_KIND = {
  perseguidor: '#ff0000', // rojo
  emboscador:  '#ffb8ff', // rosa
  flanqueador: '#00ffff', // cian
  erratico:    '#ffb852', // naranja
};
```

`draw()` deja de usar `GHOST_COLORS[ i ]` y pasa a `GHOST_COLOR_BY_KIND[ g.kind ]`.

## Plan de implementación

1. Ampliar `GHOST_STARTS` a 4 entradas en `src/js/maze.js`. Verificar que las 4
   celdas son transitables para un fantasma (`grid[14][12..15] === 0`).
2. Inicializar `releaseAt` en `createGame`: perseguidor a `0`, los demás a
   `1500`, `3000`, `4500` ms tras `performance.now()`.
3. Implementar `decideGhostPen` en `game.js`: si el fantasma está dentro de la
   pen (`y ∈ [13,15] && x ∈ [11,16]`) y `releaseAt > now`, no se mueve. Si está
   dentro y `releaseAt ≤ now`, elige la dirección que minimiza Manhattan a la
   puerta `(13, 12)` o `(14, 12)`. Si la celda objetivo la ocupa otro fantasma
   aún no liberado, se queda quieto hasta que se libere.
4. Reescribir `decideGhost` para los 4 `kind`. Mantener el filtro de no
   reversión (`OPPOSITE[ g.dir ]`) y la lógica de callejón sin salida (caer al
   opuesto). Añadir las 4 ramas:
   - `perseguidor`: minimiza Manhattan a `(round(pacman.x), round(pacman.y))`.
   - `emboscador`: objetivo = `pacman + 4 * DIRS[pacman.dir]`, clampeado a la
     última celda transitable.
   - `flanqueador`: objetivo = `2 * pacman - perseguidor`; si perseguidor en
     pen, usar `(15, 11)`.
   - `erratico`: si `manhattan(perseguidor, pacman) > 8`, objetivo = Pac-Man;
     si no, objetivo = `(1, 29)`.
5. Actualizar `resetPositions` para re-inicializar `releaseAt` y recolocar a
   los 4 fantasmas en sus `GHOST_STARTS`.
6. Reemplazar `GHOST_COLORS` por `GHOST_COLOR_BY_KIND` en `render.js` y
   cambiar `draw()` para usar el color por `kind`.
7. Probar manualmente abriendo `src/index.html` y jugar 1-2 partidas
   observando los 4 fantasmas.

## Criterios de aceptación

- [ ] `GHOST_STARTS` tiene 4 entradas, una por cada `kind`.
- [ ] En `createGame`, cada fantasma recibe un `releaseAt` con diferencia de
      1.5 s entre consecutivos, en el orden perseguidor → emboscador →
      flanqueador → errático.
- [ ] El primer fantasma sale de la pen en el frame 0 de la partida; los
      siguientes a los 1.5 s, 3 s y 4.5 s ±1 frame.
- [ ] Cuando un fantasma está en la pen y aún no le toca salir, no se mueve.
- [ ] Cuando le toca salir, camina por la pen hasta cruzar la puerta en
      `MAZE_STR[12][13..14]` y entra al pasillo central.
- [ ] Una vez fuera, cada fantasma respeta la regla de no reversión (excepto
      callejón sin salida) y aplica la lógica de objetivo de su `kind`.
- [ ] Al perder una vida, los 4 vuelven a la pen y la cuenta de salida vuelve
      a 0 s con el perseguidor primero.
- [ ] El renderer pinta cada fantasma con su color por `kind` (rojo / rosa /
      cian / naranja), independiente del orden en `game.ghosts`.
- [ ] Las globals expuestas en `window` (`MAZE`, `TUNNEL_ROW`, `PACMAN_START`,
      `GHOST_STARTS`, `createGame`, `update`, `draw`, `DIRS`) siguen existiendo.
- [ ] El juego sigue ganable: si Pac-Man se come todos los dots,
      `game.state === 'won'`.
- [ ] El juego sigue perdible: si `lives <= 0`, `game.state === 'lost'`.

## Decisiones tomadas y descartadas

- **Nombres: perseguidor / emboscador / flanqueador / errático** (mismo
  comportamiento que Blinky / Pinky / Inky / Clyde, sin nombres copyrighted).
- **Diferenciación solo por color**, no por silueta. Reduce el trabajo en
  `render.js` y respeta la estética del original.
- **Power-pellets y `frightened` fuera de alcance**. Las colisiones siempre
  cuestan una vida. Merece su propio spec si se quisiera añadir.
- **Salida escalonada por tiempo (1.5 s), no por puntos.** Más predecible y
  fácil de verificar.
- **Salir caminando por la pen, no teletransporte ni spawn exterior.** Más
  fiel al original; implica mini-IA intra-pen.
- **Reset completo al perder vida** (vuelven a la pen + reinicia escalonado).
  Descartada la alternativa "salir todos a la vez tras el reset" porque
  elimina la rítmica que da el escalonado.
- **Misma velocidad para todos.** Suficiente para que la diferenciación sea
  por objetivo. Descartada la opción de velocidad por personalidad porque
  complica el balance sin ganancia clara.
- **`releaseAt` en ms con `performance.now()`**, no en frames. Más robusto
  ante variaciones de RAF.
- **Mapeo color↔kind centralizado en `GHOST_COLOR_BY_KIND`** (kind-based, no
  index-based). Si el orden de `GHOST_STARTS` cambia, los colores no se
  desordenan.

## Riesgos identificados

- **Bloqueo en la pen**: dos fantasmas pueden ocupar la misma celda al ir
  hacia la puerta. Mitigación: en `decideGhostPen`, si la celda objetivo la
  ocupa otro fantasma aún no liberado, no avanzar.
- **Emboscador contra muro**: si Pac-Man mira a una pared, el objetivo
  4-adelante cae dentro del muro. Mitigación: clampear el objetivo a la
  última celda transitable en esa dirección (≤2 si hay pared antes de 4).
- **Frame skip en RAF**: un fantasma puede salir 1 frame tarde respecto a
  `releaseAt`. Aceptable dentro de ±1 frame; documentado, no se mitiga.
- **Cambio de orden de `GHOST_STARTS`**: si en el futuro alguien lo reordena,
  el mapeo por índice anterior habría roto los colores. Mitigado al pasar a
  `GHOST_COLOR_BY_KIND` (kind-based).
