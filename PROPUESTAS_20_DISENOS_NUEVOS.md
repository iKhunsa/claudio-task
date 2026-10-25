# Familias nuevas 17–76 (implementadas)

**Estado:** las 60 familias están implementadas en `MusicOverlayStudio.html`: 17–36 son las 20 propuestas originales de este documento y 37–76 son 40 adicionales (lista al final). Las 76 tienen las 5 variantes y 0 fallos de contraste AA en las variantes sólidas.

## Las 20 propuestas originales (17–36)

Todas con silueta, material y movimiento distintos de las 16 familias y los 6 subestilos Watch actuales. Cada una hereda las 5 variantes (`white`, `black`, `transparent`, `glass`, `accent`), el contrato de datos y las reglas de contraste de la auditoría (mínimo 12 px, AA sobre fondo claro, oscuro y medio).

"Datos reales" indica qué campo necesita para mostrar algo que no sea decorativo; si falta, ese bloque se oculta.

| # | Nombre | Silueta y material | Movimiento | Tecnología | Datos reales |
|---|---|---|---|---|---|
| 17 | Vinyl Sleeve Slide | Funda de vinilo cuadrada de cartón; medio disco asoma por la derecha con la etiqueta de la carátula | El disco sale de la funda al empezar y entra al pausar; gira solo fuera de ella | CSS | título, artista |
| 18 | Boombox 80s | Radio portátil horizontal con dos altavoces de rejilla, asa y cassette visible | Los conos vibran; el cassette gira | CSS + SVG | título, estado |
| 19 | Neon Tubes | Título en tubos de neón sobre pared de ladrillo oscuro | Parpadeo de encendido al cambiar de pista; luego estable | SVG filter + CSS | título, artista |
| 20 | Cassette Shelf | Estante con lomos de casete; el de la pista actual sobresale y se lee | El lomo actual se desliza hacia fuera al cambiar | CSS 3D | título, cola |
| 21 | Jukebox Arch | Arco de jukebox clásico con tubos de luz y ventana central de la carátula | Burbujas que suben por los tubos (lentas) | CSS + SVG | título, artista |
| 22 | Pixel Handheld | Consola portátil con pantalla LCD verde de 4 tonos; el progreso es una barra de vida | Texto que aparece letra a letra, una vez por pista | CSS (pixelado) | título, progreso |
| 23 | VU Meter Analógico | Medidor de aguja con esfera color papel; carátula en el cuadrante | Aguja con rebote suave; decorativa si no hay audio real | SVG | duración (opc. audio) |
| 24 | Polaroid Drop | Polaroid con borde blanco; título "escrito a mano" en el margen inferior | La foto cae y se revela al cambiar de pista | CSS | título, artista, carátula |
| 25 | Concert Ticket | Entrada perforada con talón desprendible; la hora es el progreso y hay un código de barras | El talón se rasga y sale al cambiar de pista | CSS + SVG | título, duración |
| 26 | Thermal Receipt | Recibo térmico estrecho con la pista como renglón; sale de una ranura | Se imprime línea a línea una vez por pista | CSS | título, artista, petición |
| 27 | Radio Dial | Dial FM con escala numerada y aguja roja; el ancho del dial es la duración | La aguja avanza con el progreso | SVG | progreso, duración |
| 28 | CRT Terminal | Monitor de tubo con scanlines tenues y prompt `> now_playing` | Cursor intermitente; ligera curvatura | CSS | título, artista |
| 29 | Rack Hardware | Módulo de rack 2U con perillas, LEDs y pantalla de siete segmentos | Los LEDs reflejan estado; el display muestra el tiempo | SVG + CSS | tiempo, estado |
| 30 | Paper Collage | Recorte de papel rasgado con cinta adhesiva; la carátula pegada torcida | Entrada tipo stop-motion a pocos fotogramas | CSS | título, artista |
| 31 | Brutalist Type | Título gigante en bloques de color plano, borde grueso negro, sin sombra | Corte seco de color al cambiar de pista | CSS | título, artista |
| 32 | Liquid Orb | Esfera de líquido cuyos colores salen de la paleta de la carátula | Deformación lenta (filtro "gooey") | SVG filter / Pixi opcional | paleta (`pal`) |
| 33 | Record Crate | Caja de discos de madera vista de frente; el disco actual se levanta | Los discos vecinos se inclinan al cambiar | CSS 3D | cola, historial |
| 34 | Ring Stack | Tres anillos concéntricos (progreso, volumen, cola) estilo fitness | Los anillos se rellenan con ease largo | SVG | progreso, volumen |
| 35 | Hologram Card | Tarjeta con reflejo iridiscente que desplaza un degradado cónico | Barrido lento del brillo (por tiempo, no por puntero) | CSS | título, carátula |
| 36 | Mixtape Label | Etiqueta de mixtape manuscrita con reglas de renglón y el título subrayado | El subrayado se dibuja al cambiar de pista | SVG | título, playlist |

## Orden sugerido de implementación

1. **Bajo costo y alto contraste** (CSS/SVG puros): 17, 24, 25, 26, 31, 36.
2. **Con identidad fuerte**: 18, 19, 22, 28, 29.
3. **Con más trabajo de profundidad o filtros**: 20, 21, 32, 33, 35.
4. **Que dependen de datos que hoy pueden faltar**: 23, 27, 34 (se ocultan si no hay audio, duración o volumen).

## Notas de diseño

- Las familias 26 y 28 usan tipografía monoespaciada porque el recibo y la terminal son literalmente ese objeto; es la excepción justificada a la regla de no usar monoespaciada en datos pequeños.
- Nunca se inventan colas, peticiones ni letras: 11, 33 y 36 se ocultan sin datos.
- Ninguna usa confeti ni partículas continuas; 32 y 35 llevan un modo `motion=off` estático.

## 40 familias adicionales (37–76)

| # | Nombre |
|---|---|
| 37 | Equalizer Wall |
| 38 | Classic MP3 Window |
| 39 | Click Wheel Player |
| 40 | Vending Machine |
| 41 | Gramophone Horn |
| 42 | Reel to Reel |
| 43 | Minimal Line |
| 44 | Newspaper Clipping |
| 45 | Vintage Postcard |
| 46 | Airport Board |
| 47 | Film Strip |
| 48 | Bento Grid |
| 49 | Sticker Sheet |
| 50 | Comic Panel |
| 51 | Notebook Doodle |
| 52 | Trading Card |
| 53 | Lava Lamp |
| 54 | Planet Orbit |
| 55 | Sunrise Poster |
| 56 | Retro OS Window |
| 57 | Menu Bar Pill |
| 58 | Dynamic Island |
| 59 | Lock Screen Widget |
| 60 | Car Gauge |
| 61 | Transistor Radio |
| 62 | Nixie Clock |
| 63 | Flip Clock |
| 64 | Subway Sign |
| 65 | Road Sign |
| 66 | Cinema Marquee |
| 67 | Stage Spotlight |
| 68 | Skateboard Deck |
| 69 | Guitar Pick |
| 70 | Piano Keys |
| 71 | Metronome |
| 72 | Speaker Tower |
| 73 | Headphones Frame |
| 74 | Vintage Microphone |
| 75 | Matchbox |
| 76 | Radar Sweep |
