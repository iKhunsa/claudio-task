# Revisión de movimiento

Aplicado con los skills de [emilkowalski/skill](https://github.com/emilkowalski/skill) (instalados en `.agents/skills/`): `emil-design-eng`, `review-animations` (con `STANDARDS.md`), `improve-animations`, `find-animation-opportunities` y `break-ui`. Los valores salen de `STANDARDS.md`: `--ease-out: cubic-bezier(0.23, 1, 0.32, 1)`, `--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1)` y `--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1)`.

## Hallazgos corregidos

| Antes | Después | Por qué |
| --- | --- | --- |
| `@keyframes rise` definido dos veces; el de la lámpara de lava (familia 53) pisaba al de entrada de texto | El de la lámpara pasa a llamarse `lava` | Al cambiar de pista el texto de todas las familias subía 220 px y no volvía |
| Relleno de progreso animado con `transition: width .3s linear` | `clip-path: inset(0 X% 0 0)` con `transition: clip-path .3s linear` | `width` fuerza layout; el movimiento constante va en `linear` |
| Aguja del dial (familias 27 y 61) movida con `style.left` | `transform: translateX(px)` con `transition: transform .3s linear` | `left` fuerza layout; solo `transform` y `opacity` |
| Carruseles, cajas y portadas con `transition: transform .8–1.2s` | `.5s var(--ease-drawer)` (familias 05, 10, 20, 33, 95), `600ms var(--ease-io)` (04, 17) y `.4s var(--ease-out)` (40) | La UI se queda en ≤ 500 ms y en movimiento en pantalla usa ease-in-out |
| Interruptor del tocadiscos con `transition: transform .4s` | `200ms var(--ease-out)` | Respuesta de sistema: rápida |
| Caída de la máquina expendedora con `ease-in` (`cubic-bezier(.5,0,1,.6)`, 800 ms) | `480ms var(--ease-out)` | Nunca `ease-in` en UI: retrasa justo el momento que el usuario mira |
| Entradas de firma de 700–1800 ms (`stp`, `typ`, `dev`, `flick`, `draw`, `print`, `glow`) | 420–900 ms, la mayoría ≤ 500 ms, con `var(--ease-out)` donde no es por pasos | Los eventos ocasionales admiten movimiento estándar, no largo |
| Entrada con `--tc: 450ms` por defecto y deslizador hasta 1500 ms | `--tc: 280ms` y deslizador máximo de 600 ms | La UI de entrada va por debajo de 300 ms |
| Entrada de todo el bloque a la vez (`.meta`) | Título, artista y progreso escalonados a 0, 60 y 110 ms con `enterUp` (`translateY(6px) scale(.97)` + opacidad) | Entrada escalonada de 30–80 ms; nunca `scale(0)` |
| Pausa: `animation-play-state: paused` (parada en seco) | Desaceleración de 700 ms y aceleración de 350 ms con la Web Animations API, interrumpible | Las físicas no frenan de golpe; el movimiento se retoma desde la velocidad actual |
| `setTrack` reconstruía el DOM aunque la pista fuera la misma | Misma pista: solo se actualizan el estado y el tiempo | Las animaciones (keyframes) reiniciaban cada segundo |
| Salida inmediata al limpiar | `leaving` de 180 ms (`ease-out`) antes de vaciar | Salida más rápida que la entrada |
| Cambio de variante sin transición | `swapIn` de 220 ms con `blur(2px)` | El blur disimula el cruce de dos estados |
| `prefers-reduced-motion` eliminaba todo, incluso en la interfaz del editor | Se quita el movimiento y se conserva el fundido de opacidad (150–160 ms) | "Reducido, no cero" |
| Botones del editor sin retroalimentación de pulsación | `:active { transform: scale(.97) }` con `transition: transform 160ms var(--ease-out)` | Feedback inmediato; el hover va tras `@media (hover: hover) and (pointer: fine)` |

## Datos extremos (`break-ui`)

Probado en las 136 familias con 10 conjuntos de datos reales y extremos: CJK largo, emoji, árabe de derecha a izquierda, título y artista de una letra, sin artista, 3 horas de duración, posición mayor que la duración, portada rota, HTML y URLs maliciosas, palabras largas sin espacios. Resultado: 0 desbordes del lienzo, 0 errores de JS y 0 inyecciones.

Correcciones que salieron de ahí:

| Antes | Después | Por qué |
| --- | --- | --- |
| `url('${coverUrl}')` sin sanear | Solo `http(s):`, `data:image/` y `blob:`, con `' " ( ) \ < >` codificados | Una URL con `')` rompía el estilo en línea |
| Paleta sin validar en `style` | Solo `#hex`, `rgb()` y `hsl()` | Evita inyección de CSS |
| Duración con formato `m:ss` | `h:mm:ss` desde 1 hora | 3 h mostraba `180:00` |
| Valores numéricos sin límites | Duración, posición y volumen acotados; posición ≤ duración | 150 % de volumen o posición > duración rompían barras |
| Texto árabe/hebreo sin dirección | `unicode-bidi: plaintext` | Orden correcto en RTL |
| Artista vacío dejaba un hueco | `.art:empty { display: none }` | Sin huecos vacíos |

## Qué se dejó como está y por qué

- `stroke-dashoffset` del anillo de progreso (SVG): animación de atributo de un solo elemento pequeño, sin layout; no hay equivalente con `transform`.
- Movimientos decorativos continuos (discos, bobinas, ecualizador): van en `linear` o `ease-in-out`, se detienen al pausar y se desactivan con `prefers-reduced-motion`.
- `@keyframes` en las entradas por cambio de pista: el DOM se sustituye entero y la pista cambia de forma ocasional, así que no hay interrupciones a mitad de animación que importen.
- Caídas de la polaroid y del recibo (familias 24 y 26): desplazamientos intencionados de entrada, con 520 y 480 ms.

## Verificación

- Pausa: la velocidad de reproducción pasa de 1 a 0,52 a los 150 ms y a 0 en 1 s; al reanudar vuelve a 1.
- Misma pista: se conservan los nodos del DOM.
- Movimiento reducido: 0 animaciones infinitas.
- Entrada: ninguna familia se desplaza más de 24 px salvo la 24 y la 26, que terminan en su sitio.
