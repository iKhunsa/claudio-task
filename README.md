# claudio-task

**TikLiveTTS Music Overlay Studio** — frontend estático en un único archivo: `MusicOverlayStudio.html`.
Especificación: `TikLiveTTS_Music_Overlays_Especificacion.md`.

- 16 familias + 6 subestilos Watch (14A–F) × 5 variantes (`white`, `black`, `transparent`, `glass`, `accent`).
- `?mode=gallery` · `?mode=editor&style=04&variant=glass` · `?mode=overlay&style=04&variant=glass` (fondo transparente, para OBS).
- Sin dependencias ni red. Los objetos "3D" usan capas CSS 2.5D (Three.js/PixiJS quedan como mejora opcional).
- Datos reales: `window.MusicOverlayStudio.setTrack({...})`, `postMessage({type:'track',track})` o `BroadcastChannel('tiklivetts-music')`. Sin datos el overlay no muestra nada; `&demo=1` carga un ejemplo solo para pruebas.
