# Descripción general del proyecto

**FIREWALL: Zero Trust** es un videojuego escrito 100% en **Jack** (lenguaje de Nand2Tetris) que corre sobre la arquitectura **Hack**, en el VM Emulator. Es la entrega final (Evaluación 4, 25%) del curso Organización de Computadores, EAFIT, prof. Mauricio Arias-Correa. Equipo de 3 integrantes (A, B, C).

**Fechas:** anuncio público en Teams hasta el **3-nov-2026** · sustentación en aula el **17-nov-2026** (18 min: 8 presentación + 5 demo + 5 preguntas).

**El juego:** el jugador es el firewall de una red. Con una mira dispara solo al malware que rebota por la pantalla, sin bloquear el tráfico legítimo. Empieza con 3 vidas.

**Controles:** flechas = mover mira · espacio = disparar (disparo por flanco: hay que soltarla entre disparos) · Enter = avanzar en pantallas de inicio, nivel y game over.

| Acción | Resultado |
|---|---|
| Disparar a malware | +10 puntos |
| Disparar a tráfico legítimo (falso positivo) | −1 vida |
| Dejar escapar malware (falso negativo) | −1 vida |
| Tráfico legítimo que sale sin ser bloqueado | +2 puntos |

| Nivel | Desde | Escenario | Paquetes máx. | Velocidad |
|---|---|---|---|---|
| 1 | 0 pts | Bandeja de entrada (sobre con sello vs. sobre malvado) | 3 | 1 px/frame |
| 2 | 100 pts | Red interna (paquete recto vs. gusano ondulado) | 4 | hasta 2 px/frame |
| 3 | 250 pts | Servidor bajo ataque (servidor con chulo vs. con bug) | 5 | hasta 3 px/frame |

Al iniciar cada nivel se muestra cómo luce el legítimo y el malware de ese nivel.

**Requisitos del enunciado y dónde se cumplen:**

| Requisito | Dónde |
|---|---|
| Teclado en tiempo real, sin pausas | `Game.run` lee `Keyboard.keyPressed()` una vez por frame, nunca `readChar` |
| Física calculada (rebote) | `Packet.update`: suma (vx, vy) y refleja en bordes, como la pelota de Pong |
| Fondo móvil continuo | `Background.step`: rota palabras de 16 bits en la memoria de video |
| Puntaje visible | `Hud.update`: puntos y vidas en la fila superior |

**Diferenciador (va en Teams y README):** la mecánica central es distinguir bajo presión, castigando falsos positivos y falsos negativos, como un sistema de detección de intrusos.

**Criterios de evaluación que condicionan el trabajo:**
- Respuesta a preguntas: **40%**. Cualquiera del equipo puede recibir la pregunta, así que todos deben entender todas las clases.
- Sustentación 20% (prohibido proyectar código), interrogación a pares 20%, funcionamiento 20%.
- Si el juego no coincide con lo anunciado en Teams o falla en clase, **todas las demás notas se reducen al 50%**.
- Video de YouTube (juego + arquitectura) en el README: obligatorio, no da puntos.

# Arquitectura

9 clases, una responsabilidad cada una. `Game` orquesta la partida; no hay `PacketPool`: `Game` guarda los 5 `Packet` en un `Array`.

Solo 3 clases crean objetos (`Game`, `Crosshair`, `Packet`). `Level`, `Hud`, `Background`, `Sprites` y `Random` son solo de `function`: no tienen objetos ni `dispose`. En el heap viven 8 bloques: 1 `Game`, 1 `Crosshair`, 1 `Array` de 5 y 5 `Packet`. Nada se crea ni se destruye durante la partida.

**Capas de dependencia:**
- Capa 0 (sin dependencias): `Sprites`, `Random`, `Level`, `Crosshair`, `Background`
- Capa 1: `Packet` (usa Sprites, Random, Level), `Hud` (usa Sprites)
- Capa 2: `Game` (usa todas)
- Capa 3: `Main` (usa Game)

**Contrato de interfaces (fuente de verdad: el código en `FirewallZeroTrust/`):**

```
Sprites    function void draw(int x, int y, int level, boolean threat)  // 16x12
           function void erase(int x, int y)
Random     function void setSeed(int seed)  // si es 0 usa 12345: el LFSR se traba en 0
           function int  range(int n)       // 0..n-1, nunca negativo
Level      function int speed(int l), maxPackets(int l), lifetime(int l)
           function int forScore(int s), frameDelay()
Crosshair  constructor Crosshair new(int ax, int ay)
           method void move(int dx, int dy) // limitado al campo de juego
           method int getX(), getY()
           method void draw(), erase(), flash(), dispose()
Packet     constructor Packet new()         // nace inactivo
           method void spawn(int level)     // posicion, velocidad y tipo aleatorios
           method void update()             // suma (vx,vy), rebota en bordes, ttl--
           method void collide(Packet o)    // choque elastico entre dos paquetes
           method boolean isActive(), isThreat(), expired(), contains(int px, int py)
           method void draw(int level), erase(), deactivate(), dispose()
Hud        function void print(int row, int col, String s)  // imprime y libera el String
           function void showStart(), showLevel(int l), showGameOver(int score)
           function void frame(int level)   // rotulos fijos y lineas del campo
           function void update(int score, int lives)       // solo numeros
Background function void init(), step()
Game       constructor Game new()
           method void run(), dispose()
Main       function void main()             // new, run, dispose, en loop
```

**Mapa de pantalla (512×256):**
- y 0–10: HUD (fila de texto 0). y 12: línea separadora.
- y 13–240: campo de juego (mira y paquetes).
- y 242: línea. y 245–246 y 250–251: dos carriles de `Background`.
- Sprites de 16×12; x máximo 496, y máximo 229.

**Estados de `Game.run`:** INICIO → INTRO_NIVEL → JUGANDO → GAME_OVER. Al subir de nivel vuelve a INTRO_NIVEL; desde GAME_OVER vuelve a INICIO con Enter (`Main` libera la partida y crea otra).

**Orden del game loop (`Game.run`):**
1. Leer `Keyboard.keyPressed()`
2. Borrar y mover la mira
3. Borrar todos los paquetes activos
4. Disparo por flanco (solo cuenta si espacio pasó de suelta a presionada): el primer paquete que contiene la mira
5. Por cada slot del nivel: nacer (1 de cada 40 frames), `update`, o morir si `expired`
6. Choques entre pares de paquetes (`collide`)
7. Dibujar paquetes y, encima, la mira
8. `Hud.update` si cambió puntaje o vidas; ¿cambio de nivel? (`Level.forScore(score)`)
9. `Background.step` y esperar `Level.frameDelay`

**Decisiones técnicas clave (se preguntan en la sustentación):**
- **Rebote en bordes:** posición += velocidad; si pasa el borde, se refleja (ej. `992 - x` en el borde derecho) y la velocidad cambia de signo.
- **Choque entre paquetes:** cajas de 16×12 que se solapan; masas iguales y choque elástico = intercambian la velocidad en el eje del choque, solo si se están acercando (si no, se quedarían pegados). `collide` lee los campos del otro paquete tratándolo como `Array` (`b[0..3]` = x, y, vx, vy), por eso no se puede cambiar el orden de los `field` de `Packet`.
- **Fondo móvil:** cada carril repite un patrón de 16 bits en las 32 palabras de su fila; rotar la palabra 1 bit mueve todo el carril 1 píxel, con solo 128 escrituras por frame. El patrón se lee de la propia pantalla, así que `Background` no guarda estado. Dirección de fila: `16384 + fila*32`.
- **Aleatorios:** LFSR de 16 bits, polinomio 0xB400 (en Jack: `-19456`). Semilla = vueltas del loop de inicio hasta que el jugador pulsa Enter.
- **Memoria:** los 5 paquetes se crean una vez y se reutilizan; todo texto pasa por `Hud.print`, que libera el `String`; en cada frame solo se imprimen números.
- **Atajo de depuración/demo:** en la pantalla de inicio, teclas `1`, `2`, `3` eligen el nivel de arranque (0 / 100 / 250 pts). Documentado en el README.

# Stack tecnológico

- **Lenguaje:** Jack (solo `.jack` en el juego). Sin dependencias externas.
- **Plataforma:** Hack, ejecutado en el **VM Emulator** del IDE web de nand2tetris.
- **Compilación:** Jack Compiler del IDE web (`.jack` → `.vm`).
- **SO de Jack:** `Screen`, `Keyboard`, `Output`, `Memory`, `String`, `Array`, `Math`, `Sys` vienen dentro del VM Emulator; no están en el repo.
- **Control de versiones:** Git + GitHub (repo público). Comunicación y anuncio: Microsoft Teams.

# Convenciones de código

**Regla de Jack que siempre causa errores:**
- Toda asignación lleva `let`; toda llamada sin valor de retorno lleva `do`.
- Todo método y función termina en `return;`, incluso los `void`.
- **No hay precedencia de operadores:** `x + 2 * 3` se evalúa de izquierda a derecha. Usar paréntesis siempre.
- Una clase por archivo; el archivo se llama igual que la clase.
- `field` = atributo de cada objeto; `static` = compartido por toda la clase (ej. estado de `Random`).
- Enteros de 16 bits con signo: literales hasta 32767. Un valor mayor se escribe en su equivalente negativo (`0xB400` → `-19456`).
- No existen `>>`, `<<`, `^`, `%`. Se arman con división, suma, `&`, `|`, `~`.
- **Trampa del shift a la derecha:** `Math.divide` trunca hacia cero con negativos, así que `x/2` no es un shift lógico. Para `x < 0` usar `((x & 32767) / 2) | 16384`.
- Rotar 1 bit a la izquierda: `(p + p) + (p < 0 ? 1 : 0)` expresado con `if`, porque Jack no tiene operador ternario.
- Cada texto entre comillas crea un `String` nuevo en cada ejecución: **nunca** imprimir texto literal dentro del loop (llena el heap). Imprimir solo números.
- Después de pintar con `Screen.setColor(false)`, volver a `Screen.setColor(true)`.
- **Sin tildes ni ñ en `.jack`** (textos y comentarios): la fuente de Hack no las tiene y algunos compiladores fallan.
- Indentación de 4 espacios, sin tabs. Comentarios breves en cada clase y método público (el repo se evalúa por "estructura de clases comentada y organizada").
- Cada `new` tiene su `dispose()`; `Main` libera la partida antes de crear otra.
- Nombres: clases en `PascalCase`, métodos y variables en `camelCase`, igual que el archivo.

**Git:**
- Una clase = un dueño. Solo el dueño edita ese archivo.
- Ramas cortas `feat/<clase>` y `fix/<tema>`; `main` protegida; PR con revisión cruzada (A revisa a B, B a C, C a A).
- Commits pequeños con formato `Clase: qué se hizo`.
- Checklist de PR: compila en el Jack Compiler, probado en el VM Emulator, sin tildes/ñ, todo método con `return;`, sin strings literales en loops.

# Estructura de carpetas

```
FirewallZeroTrust/          las 9 clases .jack (una por archivo, mismo nombre que la clase)
  Main.jack  Game.jack  Level.jack  Crosshair.jack
  Random.jack  Packet.jack
  Sprites.jack  Background.jack  Hud.jack
tests/Visual/Main.jack      Main de prueba: muestra los 6 sprites y la mira
docs/
  WORKFLOW.md               este archivo
README.md                   descripción, controles y enlace al video de YouTube
.gitignore                  *.vm
```

**Propiedad de archivos:**

| Integrante | Clases | Extra |
|---|---|---|
| **A: Núcleo e integración** | `Main`, `Game`, `Level`, `Crosshair` | Dueño de `main`, merges, tags |
| **B: Física y paquetes** | `Random`, `Packet` | Prueba de fuga de memoria |
| **C: Gráficos y video** | `Sprites`, `Background`, `Hud` | README, video, Teams |

# Comandos

**Compilar y probar en el IDE web de nand2tetris:**
1. Crear una carpeta con solo los `.jack` de esa prueba (aunque sea un archivo, debe ir dentro de una carpeta).
2. Abrir el **Jack Compiler**, cargar la carpeta en el panel Source.
3. Presionar **Compile** (compila todos los archivos abiertos y muestra errores de sintaxis).
4. Presionar **Run** (carga el código en el VM Emulator).
5. En el VM Emulator presionar **Enable Keyboard**; sin esto las flechas no llegan al juego.
6. Para el ajuste de velocidad, usar `Level.frameDelay` y el control de animación/velocidad del emulador.

**Prueba visual de sprites y mira:** compilar una carpeta con `tests/Visual/Main.jack`, `Crosshair.jack` y `Sprites.jack`.

**Git:**
```bash
git checkout -b feat/<clase>
git add FirewallZeroTrust/<Clase>.jack
git commit -m "<Clase>: descripcion corta"
git push -u origin feat/<clase>     # luego abrir PR a main
```

**Cronograma:** 3-nov Teams + repo + stubs + contrato · 6-nov clases probadas por separado · 9-nov integración nivel 1 (`v0.1`) · 12-nov niveles 2-3 y congelar funcionalidad · 13/14-nov pruebas completas y fugas de heap · 15-nov video · 16-nov ensayo con cronómetro (`v1.0`).

**Errores comunes:**

| Síntoma | Causa | Solución |
|---|---|---|
| "Class X doesn't exist" | Falta `X.jack` en la carpeta | Agregarlo (ej. `Packet` necesita `Level`) |
| Panel VM Code vacío | Se cargó la carpeta `.jack` directo en el VM Emulator | Compilar primero en el Jack Compiler y dar Run |
| Error de `Screen` al mover la mira | Dibujo fuera de 512×256 | Revisar límites en `Crosshair.move` y `Packet.update` |
| El juego se cae tras un rato | Heap lleno por Strings sin liberar | `dispose()` y solo números en el loop |
| Flechas sin efecto | Teclado no activado | Enable Keyboard |
| Muy lento o muy rápido | Depende del computador | Ajustar `Level.frameDelay` en el computador de la demo |

# Reglas importantes

## ⚠️ OBLIGATORIO: usar la skill `pony-tail` para escribir la menor cantidad de código posible

**Antes de escribir o modificar cualquier archivo `.jack`, invocar la skill `pony-tail`.** El objetivo es resolver cada requisito con el mínimo de código: menos clases auxiliares, menos métodos, menos líneas, sin abstracciones "por si acaso" ni duplicación.

- Cumplir primero el contrato de la sección Arquitectura; ahorrar código **dentro** de ese contrato, nunca cambiándolo por tu cuenta.
- No eliminar requisitos del enunciado ni del anuncio en Teams para ahorrar código (teclado sin bloqueo, rebote calculado, fondo móvil, puntaje visible, 3 niveles).
- El código corto debe seguir siendo **explicable en la sustentación**: si una línea ahorrada no se puede defender con "por qué + qué pasaría si fuera de otra forma", no se usa.
- Reutilizar en vez de duplicar (ej. los 5 `Packet` reciclados; lógica de niveles en la tabla de `Level`, no repartida).
- Si la skill `pony-tail` no está disponible en el entorno, avisar al usuario y aplicar igual el mismo principio manualmente.

## Reglas del proyecto

1. **El juego debe coincidir con lo anunciado en Teams.** No agregar ni quitar mecánicas anunciadas sin avisar al equipo; si no funciona como se declaró, todas las notas bajan al 50%.
2. **Congelamiento el 12-nov:** después solo `fix/` con issue. Nada de funcionalidades nuevas.
3. **Antes de editar, confirmar quién es el usuario (A, B o C).** Solo editar los archivos de ese integrante. Si se necesita un cambio en una clase ajena, proponerlo como issue o comentario al dueño, no editarla.
4. **No cambiar firmas del contrato** sin PR aprobado por los 3.
5. **Prohibido proyectar código** en la presentación: solo diagramas y texto mínimo.
6. **Nunca usar `Keyboard.readChar`** en el loop (bloquea). Solo `keyPressed()`.
7. **Nunca imprimir strings literales dentro del loop**; solo números. Liberar con `dispose()` todo lo creado con `new`.
8. **El LFSR nunca queda en 0** (se quedaría fijo): `Random.setSeed` cambia una semilla 0 por 12345.
9. **No commitear `.vm`.** Se regeneran al compilar.
10. **Sin tildes ni ñ en ningún `.jack`.**
11. **Probar siempre en el VM Emulator** con las flechas habilitadas antes de abrir un PR. Probar en el computador donde se presentará antes de la demo.
12. **Plagio:** un juego repetido se considera plagio. No copiar juegos de Jack disponibles en la web; la lógica debe ser propia y defendible línea por línea.
13. **Todos entienden todo:** cada dueño debe poder explicar sus clases sin mirar el código, y los otros dos deben poder responder preguntas sobre ellas.
14. **Idioma:** documentación y comunicación en español; identificadores y comentarios en `.jack` sin tildes.