# claudio-task

**TikLiveTTS Music Overlay Studio** — frontend estático en un único archivo: `MusicOverlayStudio.html`.
Especificación: `TikLiveTTS_Music_Overlays_Especificacion.md`.

- 106 familias (01–16 de las referencias, 17–76 propuestas nuevas, 77–96 estilo macOS/iOS, 97–101 Linux, 102–106 Windows; la 14 incluye 6 subestilos A–F) × 5 variantes (`white`, `black`, `transparent`, `glass`, `accent`).
- `?mode=gallery` · `?mode=editor&style=04&variant=glass` · `?mode=overlay&style=04&variant=glass` (fondo transparente, para OBS).
- Sin dependencias ni red. Los objetos "3D" usan capas CSS 2.5D (Three.js/PixiJS quedan como mejora opcional).
- Datos reales: `window.MusicOverlayStudio.setTrack({...})`, `postMessage({type:'track',track})` o `BroadcastChannel('tiklivetts-music')`. Sin datos el overlay no muestra nada; `&demo=1` carga un ejemplo solo para pruebas.

Las 90 familias nuevas están descritas en `PROPUESTAS_20_DISENOS_NUEVOS.md`.

Cada familia tiene categoría (**UI/UX · limpio** o **Gráficos · decorativo**) y etiquetas; la galería y el editor permiten filtrar por ambas (`&cat=ui&tag=Apple`).
