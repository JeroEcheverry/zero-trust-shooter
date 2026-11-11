# FIREWALL: Zero Trust

Juego arcade escrito 100% en **Jack** (Nand2Tetris) para la plataforma **Hack**. Proyecto final de Organización de Computadores, EAFIT. Es un juego, no una herramienta de seguridad.

Eres el firewall de una red. Por la pantalla rebotan paquetes: algunos son tráfico legítimo y otros son malware. Con una mira disparas **solo al malware**. Lo difícil es distinguir bajo presión: se castigan tanto los falsos positivos (bloquear tráfico legítimo) como los falsos negativos (dejar pasar malware), igual que en un sistema de detección de intrusos.

**Video:** _pendiente: pegar aquí el enlace de YouTube_

## Controles

| Tecla | Acción |
|---|---|
| Flechas | Mover la mira |
| Espacio | Disparar (hay que soltarla entre disparos) |
| Enter | Avanzar en las pantallas de inicio, nivel y game over |
| `1` / `2` / `3` en la pantalla de inicio | Atajo de demo: empezar en el nivel 1, 2 o 3 (0 / 100 / 250 puntos) y luego Enter |

## Reglas

Empiezas con 3 vidas.

| Acción | Resultado |
|---|---|
| Disparar a malware | +10 puntos |
| Disparar a tráfico legítimo | −1 vida |
| Malware que se queda hasta desaparecer | −1 vida |
| Tráfico legítimo que pasa sin ser bloqueado | +2 puntos |

| Nivel | Desde | Escenario | Paquetes | Velocidad |
|---|---|---|---|---|
| 1 | 0 pts | Correo: sobre con sello vs. sobre con cara de malvado | 3 | 1 px/frame |
| 2 | 100 pts | Red interna: paquete de datos vs. gusano ondulado | 4 | hasta 2 px/frame |
| 3 | 250 pts | Servidor: verificado (chulo) vs. con un bug | 5 | hasta 3 px/frame |

Al empezar cada nivel se muestra cómo se ven el legítimo y el malware. Los paquetes rebotan en los bordes y chocan entre sí.

## Cómo ejecutarlo

1. Abrir el [IDE web de nand2tetris](https://nand2tetris.github.io/web-ide/) → **Jack Compiler**.
2. Cargar la carpeta `FirewallZeroTrust/` y presionar **Compile**.
3. Presionar **Run** y, en el VM Emulator, **Enable Keyboard**.

## Estructura

`FirewallZeroTrust/` contiene las 9 clases: `Main`, `Game`, `Level`, `Crosshair`, `Packet`, `Random`, `Sprites`, `Background`, `Hud`. La arquitectura y las convenciones están en [`docs/WORKFLOW.md`](docs/WORKFLOW.md).

## Modo desarrollador

Compilar `FireWallZeroTrust-devMode/` en vez de `FirewallZeroTrust/`. En la partida, **M** abre el menú: `1`-`3` ir a un nivel · `4` restaurar vidas · `5` modo dios · `6` game over · `ENTER` volver.

Es una copia del juego; solo cambian `Game.jack` y `Hud.jack`.
