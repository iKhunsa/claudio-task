# TikLiveTTS — Music Overlay Studio
## Especificación maestra de diseño, renderizado 2D/3D, animación y construcción en un solo archivo

**Versión:** 1.0  
**Estado:** Especificación de implementación (no equivale a overlays ya programados).  
**Destino:** Galería visual + editor + vista de OBS, compilados o empaquetados en **un único archivo HTML autocontenido**.  
**Referencias:** Las 17 imágenes de la conversación, enumeradas en el orden en que se recibieron. Las imágenes originales funcionan como dirección de arte; no asumir permisos para redistribuir fotografías, carátulas, tipografías o marcas ajenas.

---

## 1. Misión y alcance

Crear una colección de overlays musicales de calidad premium para TikLiveTTS, orientada a streamers y espectadores. **No repetir un mismo diseño de tarjeta con colores distintos:** cada familia debe conservar la silueta, el lenguaje material, la jerarquía, las proporciones, las capas y el movimiento característicos de su referencia.

**Resultado funcional buscado:**

1. **16 familias visuales** derivadas de las primeras 16 referencias; la referencia 14 contiene una **subcolección de seis microinterfaces** que deben preservarse (no reducirla a una sola tarjeta).
2. **5 variantes de apariencia** por familia: `white`, `black`, `transparent`, `glass`, `accent`. Total mínimo: **80 combinaciones** de las 16 familias, sin contar las seis subinterfaces del estilo 14 ni estados del reproductor.
3. **Un Style Picker** inspirado en la referencia 17, con vista previa real de cada familia y variante.
4. **Un solo HTML entregable**, con CSS, JS, fuentes y activos necesarios embebidos o empaquetados; sin llamadas obligatorias a CDN durante reproducción local. En desarrollo puede haber múltiples archivos fuente; el entregable final se empaqueta en uno.
5. **Dos modos separados:** `gallery/editor` (interactivo para el streamer) y `overlay` (sin controles falsos para el público, fondo transparente, apto para OBS).
6. **Datos de ejemplo claramente ficticios** en la galería y una capa de integración para datos reales de la app.

### 1.1 Regla de fidelidad

Para cada overlay se debe conservar, por orden de prioridad: (1) geometría y silueta; (2) composición y proporciones; (3) materiales y profundidad; (4) jerarquía tipográfica; (5) acentos originales; (6) animación específica. Cambiar el contenido musical está permitido; convertir un tocadiscos en una tarjeta genérica, no.

### 1.2 Regla de color

El rojo de TikLiveTTS `#F0213A` es **opcional**, nunca universal. En `accent`, elegir entre `reference`, `album-art` o `brand`. Ejemplos: turquesa para Walkman, lila para el modular oscuro, naranja para waveform cálido, azul pastel para tocadiscos. En variantes monocromáticas preservar legibilidad y materialidad, no teñir indiscriminadamente todas las capas.

---

## 2. Arquitectura: un solo archivo, librerías especializadas y renderizado por demanda

### 2.1 Stack recomendado

| Herramienta | Uso recomendado | Cuándo **no** usarla |
|---|---|---|
| **Svelte** (compilado) | UI reactiva del catálogo, editor, estados y datos musicales. Generar salida estática, integrable en HTML único. | No montar frameworks separados por overlay. |
| **CSS nativo + variables** | Tipografía, layout, tarjetas, máscaras, sombras, gradientes, glass, rotaciones simples, accesibilidad. | No usar WebGL para una simple barra o sombra. |
| **GSAP** | Transiciones de portada, carruseles, apariciones, secuencias, reordenamiento de capas y reproducción/pausa de animaciones. | No crear un timeline permanente por elemento cuando CSS basta. |
| **Three.js** | Escenas realmente 3D: tocadiscos, mecanismo metálico, carruseles espaciales con perspectiva; materiales, iluminación y cámara. | No cargarlo para Minimal, Capsule o Arc Cover. |
| **PixiJS** | Partículas 2D, destellos, visualizadores de alta densidad y fondos reactivos en estilos que lo requieran. | No crear sistemas de partículas en todos los overlays. |
| **p5.js** | Arte generativo especializado y algunos efectos de música experimental. | Evitar un loop p5 adicional si PixiJS/Canvas ya resuelve el efecto. |
| **Lottie-web** | Microanimaciones vectoriales prediseñadas: estados e iconos con archivos JSON embebidos. | No usarlo para animación procedural de portada/3D. |
| **Canvas-confetti** | Efecto puntual **solo** al llegar un evento explícito de celebración (no en cada cambio de canción). | Nunca confeti continuo en un reproductor musical normal. |
| **Anime.js** | Alternativa modular para transformaciones pequeñas en un subsistema aislado. | No competir con GSAP en la misma propiedad/nodo. |
| **Popmotion** | Física de resortes/rebotes donde sea imprescindible. | No introducir otra librería de animación si GSAP cubre el caso. |
| **React / Vue / Framer Motion** | Alternativas si el producto anfitrión las exige. Framer Motion aplica a React. | No cargarlos además de Svelte en el HTML final. |
| **Tailwind CSS** | Opcional durante desarrollo como herramienta de autoría. | Evitar runtime innecesario; en el HTML final usar CSS compilado. |
| **Animate.css / Hover.css** | Inspiración o módulos opcionales en el **editor**. | No usar hover en el overlay público ni duplicar animaciones de GSAP. |

**Decisión arquitectónica:** Svelte + CSS + GSAP como núcleo; Three.js y PixiJS por **importación condicional de la familia activa**. Las otras librerías son complementos opcionales, **no todas deben incluirse en la versión final**, porque un HTML gigante eleva memoria, tiempo de inicio y consumo de GPU.

### 2.2 Restricciones del archivo único

- `index.html` debe poder abrirse localmente y mostrar la galería **sin conexión**.
- Incluir assets mediante `data:` URIs, SVG embebidos, arrays tipados, JSON embebido o empaquetador tipo single-file. Evitar depender de fuentes remotas y CDNs.
- La página inicial **no** debe iniciar simultáneamente 16 contextos WebGL. Mostrar previews estáticas (imagen/snapshot, SVG o CSS) de los diseños inactivos; activar renderizadores solo en el preview seleccionado.
- Separar las rutas de ejecución mediante query string o hash: `?mode=gallery`, `?mode=editor&style=04&variant=glass`, `?mode=overlay&style=04&variant=glass`.
- Si las carátulas reales se obtienen por URL, la versión empaquetada seguirá siendo un archivo único **de código**, pero su contenido dinámico puede requerir red. Soportar fallback local para carátulas desconectadas.
- El HTML autocontenido puede crecer significativamente si se embeben Three.js, PixiJS, fotografías y texturas. Registrar el tamaño comprimido y decidir si se ofrece adicionalmente una versión modular. No afirmar que una versión monolítica siempre es más rápida.

### 2.3 Módulos conceptuales (empaquetados dentro del único HTML)

```text
MusicOverlayStudio
├── DataAdapter            # Track actual, progreso, solicitudes, cola, conexión
├── OverlayStore           # Estado normalizado y suscripción de cambios
├── StyleRegistry          # 16 familias y subvariantes de Watch Widget
├── ThemeEngine            # white / black / transparent / glass / accent
├── RendererManager        # HTML/CSS, WebGL o Canvas, solo uno activo por vista
├── MotionController       # GSAP + reduced motion + estados play/pause
├── Gallery               # Previews estáticas y selector
├── Editor                # Inspector, controles, tamaños, simulación
├── OBSView               # Sin UI administrativa, fondo alpha
└── Diagnostics           # FPS, memoria aproximada, recursos activos, fallbacks
```

### 2.4 Contrato de datos

```ts
type TrackState = {
  id: string;
  title: string;
  artist: string;
  album?: string;
  coverUrl?: string;
  source?: 'spotify' | 'youtube-music' | 'apple-music' | 'local' | 'other';
  durationMs?: number;
  positionMs?: number;
  isPlaying: boolean;
  updatedAt: number;
  requestedBy?: { displayName: string; platform?: string };
  nextTrack?: { title: string; artist: string; coverUrl?: string };
  playlist?: string;
  volume?: number;
  metadataAvailable: boolean;
};

type VisualOptions = {
  style: string;
  substyle?: string;
  variant: 'white' | 'black' | 'transparent' | 'glass' | 'accent';
  accentSource: 'reference' | 'album-art' | 'brand' | 'custom';
  accentColor?: string;
  coverMode: 'square' | 'disc' | 'canvas' | 'none';
  showProgress: boolean;
  showPlatform: boolean;
  showRequest: boolean;
  showQueue: boolean;
  motion: 'off' | 'subtle' | 'full';
  scale: number;
};
```

**Tiempo/progreso:** usar marca temporal de última actualización más reloj monotónico local solo mientras `isPlaying === true`; limitar a `durationMs`. Resíncronizar periódicamente con la fuente. Si no hay posición o duración, **ocultar progreso** en vez de inventarlo. El visualizador de amplitud debe representar datos reales únicamente cuando el audio esté disponible con permiso; de lo contrario usar una animación decorativa discreta, etiquetada en el editor como **no reactiva**.

---

## 3. Variantes globales: especificación verificable

| Variante | Material | Contraste y lectura | Acento | Uso habitual |
|---|---|---|---|---|
| `white` | Blanco/hueso y superficies claras; sombras grises delicadas | Texto oscuro, bordes sutiles | Original o muy contenido | Escena minimalista |
| `black` | Negro/grafito, relieves o bordes oscuros | Texto blanco y secundarios grises claros | Original con luminancia ajustada | Gameplay oscuro o neón |
| `transparent` | Alpha real, sin bloque opaco de fondo; se conservan siluetas esenciales | Sombra/text-shadow local o microplacas detrás del texto si hace falta | Original | Sobre gameplay |
| `glass` | Superficie translúcida, borde resaltado, transparencia y desenfoque cuando el navegador lo soporte | Contraste sobre fondo dinámico; fallback semitransparente | Original/reflejos | Just Chatting / BRB |
| `accent` | Material de referencia conservado | No sacrificar legibilidad | `reference`, `album-art`, `brand` o color personalizado | Branding personalizado |

**No** implementar transparent como `opacity: 0` del widget completo. Debe existir canal alfa **real**, y la música seguir visible. **No** imponer glass sobre vinilos o metal: el efecto puede aplicarse a la **base**, mientras las piezas físicas siguen siendo sólidas.

### 3.1 Estados obligatorios de cada familia

- `playing`: información, animación de estado y progreso cuando exista.
- `paused`: detener giro y animaciones relacionadas con reproducción; indicar pausa sin duplicar botones falsos.
- `loading`: skeleton discreto o carátula placeholder; sin saltos de layout.
- `disconnected`: “Sin reproducción” o esconder según opción.
- `track-change`: transicionar imagen y metadata con entrada/salida suave.
- `no-cover`: fallback abstracto coherente con la familia.
- `long-title`: truncar elegantemente o hacer marquee **solo después de una espera**, con modo reduced-motion.
- `no-duration`, `no-request`, `no-queue`: ocultar bloques inexistentes, sin dejar huecos.

---

## 4. Especificación individual de cada referencia

> Orden original de imágenes. Dimensiones recomendadas corresponden al widget **visible** en el lienzo de OBS, no al tamaño del frame de edición. Se permite escalar manteniendo proporciones y márgenes seguros.

### 01 — Compact Disc Left · disco sobresaliente

**Lectura de la referencia.** Tarjeta horizontal blanca de esquinas redondeadas, cuerpo bajo y alargado. Un disco muy grande y oscuro aparece a la izquierda, solapando gran parte del panel y proyectando sombra. Carátula impresa sobre el disco, surcos y centro perforado. A la derecha: título, artista, volumen pequeño, progreso fino en morado y controles negros. Contraste entre pieza circular texturizada y panel liso.

**Objetivo:** overlay de esquina extremadamente reconocible por su silueta, sin parecer una barra genérica.

- **Geometría:** widget aproximado `460 × 166 px`, panel derecho ~`340 × 140`, disco `160 × 160`, solape de ~`45 px`. Mantener márgenes suficientes para evitar corte del disco en OBS.
- **Jerarquía:** título 16–19, artista 12–14, progreso 3–5 px; opcional estado de reproducción en lugar de iconos de control.
- **Material:** blanco satinando, sombra de contacto bajo el disco, surcos finos radiales/concéntricos, etiqueta fotográfica central, reflejo especular leve.
- **Movimiento:** disco gira lentamente (CSS/GSAP); al pausar se frena con easing. En cambio de canción, swap de etiqueta mediante crossfade.
- **Tecnología:** CSS + SVG de surcos + GSAP. Three.js **no necesario**.
- **Variantes:** white conserva el original; black usa grafito y disco contrastado; transparent deja título flotando con disco intacto; glass aplica vidrio al panel derecho; accent cambia únicamente progreso/reflejo y detalles de etiqueta.
- **Validación:** el disco debe sobresalir, no quedar reducido a carátula cuadrada; panel derecho legible a escala 75%.

### 02 — Glass Sunset · cristal nocturno

**Referencia.** Tarjeta cuadrangular gris humo, bordes suaves y material translúcido. Fondo de atardecer con tonos naranjas, fotografía pequeña a la izquierda, título grande y artista debajo. Progreso blanco con tiempos a los extremos. Cinco controles en fila, uno con acento amarillo. La luz ambiental colorea el vidrio.

**Objetivo:** interfaz cinematográfica elegante sobre escenas oscuras.

- **Geometría:** ~`400 × 270`, padding 22–26; carátula `80 × 80`, metadata a su derecha, progreso ancho.
- **Material:** `backdrop-filter` cuando disponible; gradiente humo superior/inferior, stroke interior blanco de baja opacidad y sombra ambiental cálida. El fondo de ejemplo no debe formar parte obligatoria del overlay OBS.
- **Adaptación:** en modo OBS reemplazar fila de controles por indicador `♪ Now playing`, plataforma y tiempo; conservar espaciado de la referencia.
- **Movimiento:** ligero brillo sobre borde al cambiar canción, entrada de carátula y texto escalonada.
- **Tecnología:** CSS + GSAP; Canvas/Pixi solo si se decide agregar motas ambientales muy discretas en escena BRB.
- **Variantes:** white = cristal lechoso; black = humo denso; transparent = elementos sin placa central; glass = expresión principal; accent = ámbar/coral según fondo.
- **Validación:** el contenido debe funcionar sobre fondo claro y oscuro; contraste medido en preview de escena real.

### 03 — Arc Cover Card · arco fotográfico

**Referencia.** Tarjeta casi cuadrada con gran imagen superior recortada en semicírculo/óvalo y un volumen gris claro de esquinas muy amplias debajo. Parte inferior centrada: pequeñas barras de audio, artista gris, canción en negro, indicador fino, tiempo transcurrido y duración. Superposición de un elemento circular pequeño arriba del arco.

**Objetivo:** widget vertical editorial para BRB, contenido musical o cámaras laterales.

- **Geometría:** `330 × 360`; zona imagen ocupa ~50–55%, cuerpo inferior restante; radio inferior amplio.
- **Material:** recorte vectorial (`clip-path`/SVG mask), sombra sutil debajo del arco y separación mínima entre foto y base.
- **Contenido:** título, artista, visualizador ornamental o dato real; reloj de reproducción; evitar forzar botones inexistentes.
- **Movimiento:** transición con mascarilla curva, barras muy suaves y microescala de imagen.
- **Tecnología:** CSS/SVG + GSAP; no usar WebGL.
- **Variantes:** white gris nacarado; black antracita; transparent conserva arco y texto flotante; glass reemplaza cuerpo por placa esmerilada; accent deriva del color del álbum.
- **Validación:** conservar **el gran recorte superior curvo**, no transformarlo en carátula cuadrada convencional.

### 04 — Pastel Turntable · tocadiscos azul

**Referencia.** Gran tocadiscos cuadrado azul pastel de apariencia tridimensional suave: vinilo turquesa gigantesco con múltiples círculos surcados, imagen central y **brazo realista negro** anclado arriba a la derecha, con aguja hacia el borde interior. Esquinas amplias, relieves neumórficos, interruptor on/off, botones y un pequeño corazón.

**Objetivo:** pieza hero de gran calidad para escenas musicales.

- **Geometría:** `430 × 430` principal; disco 65–75% del ancho; brazo articulado ocupa el cuadrante superior derecho.
- **Material:** plástico mate azul claro, surcos concéntricos y rim highlights, juntas mecánicas, brazo grafito con reflejos y sombra propia.
- **Movimiento:** plato girando, brazo que se levanta/gira al iniciar o detener; rebote mínimo del botón; durante track-change cambiar etiqueta central.
- **Tecnología:** **Three.js recomendado** para versión premium realista, materiales y luces; alternativa CSS/SVG 2.5D sin WebGL.
- **Variantes:** white = azul pastel original muy claro; black = hardware negro mate y disco azul; transparent = solo disco/brazo y labels flotantes; glass = carcasa de cristal translúcido con piezas opacas; accent = azul hielo original o acento configurable.
- **Adaptación stream:** título/artista aparecen discretamente al pie o en miniplaca lateral, sin destruir la estética del objeto.
- **Validación:** se ven claramente surcos, brazo, cápsula/aguja, unión mecánica y profundidad: **no basta un círculo plano**.

### 05 — Spatial Carousel · biblioteca espacial

**Referencia.** Portadas de álbum apiladas con perspectiva 3D, elegida central en primer plano, laterales desplazadas hacia atrás e inclinadas. Navegación discreta arriba, descripción del álbum seleccionada al centro y pequeña barra flotante inferior, sobre salón oscuro con profundidad de campo.

**Objetivo:** escena de música/BRB con cambio de canción espectacular.

- **Geometría:** overlay expandido `900 × 440` (adaptable), carátula central ~`230–290`, secundarias con perspectiva y ocultamiento parcial.
- **Contenido:** actual al centro; izquierda historial si existe; derecha cola real si existe; si no hay cola, mostrar portadas de demostración solo en editor, nunca inventar canciones al público.
- **Material:** carátulas rígidas con cantos, sombras de contacto y profundidad; barra inferior de vidrio ligero.
- **Movimiento:** GSAP coordina traslación/escala/rotación en cambios; easing cinematográfico; cámara fija en modo OBS para estabilidad.
- **Tecnología:** Three.js para tarjetas espaciales con iluminación; GSAP timeline. CSS transform 3D como fallback.
- **Variantes:** white tipo visionOS luminoso, black cinematográfico, transparent solo carrusel, glass en controles y rótulos, accent tomado de carátula central.
- **Validación:** portadas laterales realmente detrás del álbum central; no hacer cinco rectángulos alineados.

### 06 — Modular Dark Purple · módulos neumórficos

**Referencia.** Lienzo dividido entre pequeñas tarjetas oscuras y un gran reproductor modular inferior. Superficies negro grafito, relieve suave, anillos circulares púrpura, acentos lila, un objeto giratorio abstracto y panel de canción rectangular violeta. Titular decorativo pixel-dot arriba en la composición principal.

**Objetivo:** reproductor técnico/gamer con piezas claramente diferenciadas.

- **Geometría:** player principal `610 × 210`: bloque circular izquierdo 33%; módulo de información derecho 67%; opcional cinta pixel-text superior.
- **Material:** negros con varias elevaciones y superficies violetas satinadas; evitar sombras exageradas tipo glow permanente.
- **Movimiento:** anillo especular lento, pequeñas esferas orbitales en el objeto, lectura de tiempo con transiciones digitales.
- **Tecnología:** CSS + SVG; Three.js opcional solo para el objeto circular dinámico; GSAP; Pixi opcional para destellos.
- **Variantes:** white con bordes grafito y lila; black original; transparent objetos + metadata; glass módulos transparentes con núcleo sólido; accent preferencia púrpura, con alternativa brand.
- **Validación:** mantener composición **bimodular** y motivo pixelado sin convertirlo en un player común.

### 07 — Blue Transparent · panel azul translúcido

**Referencia.** Gran placa horizontal azul translúcida con borde blanco fino, texto de canción y artista centrados, progreso verde/cian y cinco iconos blancos. Fondo con degradado de azules que aporta una atmósfera limpia, ligeramente futurista.

**Objetivo:** información flotante legible sobre streams con colores intensos.

- **Geometría:** `460 × 255`, título centrado, barra a 80% del ancho, espacio respirable.
- **Material:** cristal azul, esquinas redondeadas medias, borde interior fino y halo tenue.
- **Movimiento:** entrada vertical mínima, fill de barra continuo, leve vibración de ecualizador opcional.
- **Tecnología:** CSS + GSAP; sin necesidad de WebGL.
- **Variantes:** white cristal hielo, black azul petróleo, transparent texto/barra sin fondo, glass azul claro con blur, accent azul/cian/verde propios.
- **Validación:** conserva centrado y amplitud de márgenes; nunca apretar elementos hasta formato de cápsula.

### 08 — Mechanical 3D Player · reproductor industrial

**Referencia.** Composición escultórica gris metálica: un gran panel frontal con texto y controles, disco negro que emerge por arriba, carátula cuadrada en carcasa que sobresale por la izquierda, volumen/perilla circular a la derecha, tornillos y piezas laterales. Luces suaves hacen visibles los biseles.

**Objetivo:** overlay skeuomórfico mecánico convincente.

- **Geometría:** conjunto `760 × 380`, placa frontal ~`560 × 300`, disco parcial y piezas sobresalientes.
- **Material:** aluminio oscuro, plásticos, bordes biselados, oclusión ambiental, contactos reales y sombra propia bajo cada pieza.
- **Movimiento:** rotación de disco, mínimo movimiento de perilla de volumen si hay dato real, desplazamiento de carátula al cambiar canción.
- **Tecnología:** Three.js **preferido**, materiales PBR moderados; GSAP para transiciones; alternativa pre-renderizada + CSS en hardware limitado.
- **Variantes:** white metal cepillado, black metal oscuro, transparent conserva **mecanismos** sin panel trasero, glass solo placa frontal, accent luces/LED del álbum o marca.
- **Validación:** profundidad perceptible y piezas separadas; no reducirlo a un rectángulo gris con círculo encima.

### 09 — Retro Walkman · cinta de casete

**Referencia.** Dispositivo horizontal azul claro con bordes oscuros y elementos de teléfono retro. A la izquierda información musical pequeña y controles circulares; a la derecha una ilustración grande de casete negro, líneas geométricas y rótulo tipográfico `WALK·MAN` con acentos cian/rosa.

**Objetivo:** estética retro-tech reconocible y distinta de las demás.

- **Geometría:** `640 × 270` horizontal; cassette ocupa ~60%; área informativa restante.
- **Material:** cartucho, bobinas, etiqueta, tornillos, cintas, bandas y diagonales 80s.
- **Movimiento:** bobinas que giran con velocidad consistente y cinta que avanza; cambio de etiqueta en track-change.
- **Tecnología:** SVG/CSS + GSAP; Canvas/p5 solo para grano muy ligero y opcional.
- **Variantes:** white azul hielo, black carcasa negra/cian, transparent cassette y texto sin dispositivo exterior, glass dispositivo translúcido con cassette opaco, accent cyan/magenta originales.
- **Validación:** tipografía gráfica y cassette deben dominar; no reemplazar por una carátula cuadrada.

### 10 — Ambient Album Carousel · galería de carátulas con iluminación

**Referencia.** Varios álbumes flotan sobre fondo oscuro con manchas difusas rojas, naranjas y amarillas. Álbum central más grande; laterales escalonados en profundidad. Barra musical al pie, muy baja, horizontal, con mini-carátula, controles e información.

**Objetivo:** identidad visual emocional que adapta el ambiente al color de la canción.

- **Geometría:** escena expandida `900 × 520`; carrusel superior 75%, barra inferior 15%, aire entre ambos.
- **Material:** portadas con borde suave y reflejo ligero; iluminación ambiental derivada de paleta extraída de portada.
- **Movimiento:** crossfade de luces del ambiente y transición 3D del carrusel al cambiar pista.
- **Tecnología:** GSAP + CSS gradientes; Three.js opcional para la perspectiva, PixiJS opcional para partículas luminosas tenues.
- **Variantes:** white fondo blanco con halo pastel, black profundo con luces de carátulas, transparent sin fondo difuso (con carrusel), glass barra flotante esmerilada, accent de color dominante del álbum.
- **Validación:** ambiente cambia con el álbum; no fijar siempre rojo/naranja.

### 11 — Immersive Photo Card · foto a sangre

**Referencia.** Widget cuadrado oscuro con fotografía de una persona ocupando casi todo el plano; bordes negros gruesos redondeados. Franja superior superpuesta de perfil/acciones y franja inferior con tiempo, progreso y tres controles translúcidos. Estética musical-social inmersiva.

**Objetivo:** una carátula de alto impacto que siga siendo legible en stream.

- **Geometría:** `335 × 335`; foto ocupa todo el interior, UI superior/inferior sobrepuesta.
- **Material:** overlay degradado en parte inferior, borde grafito, botones con vidrio local y sombras de texto.
- **Adaptación:** el encabezado social se reemplaza por origen o `Requested by`, no se inventa identidad de artista/perfil; los botones visibles en OBS se convierten a iconos de estado o se eliminan.
- **Movimiento:** zoom lentísimo de carátula, revelado de texto y barra.
- **Tecnología:** CSS + GSAP.
- **Variantes:** white borde marfil con texto protegido por gradiente, black cercano a original, transparent imagen sin tarjeta/borde pesado, glass overlays locales de texto, accent solo microdetalles de la foto.
- **Validación:** la foto domina, no queda minimizada a miniatura.

### 12 — Black Capsule · cápsula monocromática

**Referencia.** Pastilla negra ancha de esquinas extremadamente redondas, carátula cuadrada en esquina superior izquierda, título y artista a su derecha, indicador sutil, línea de progreso horizontal y grandes controles blancos centrados abajo. Muchísimo contraste y simplicidad.

**Objetivo:** overlay legible con rapidez durante gameplay.

- **Geometría:** `620 × 235`, radio alto 48–62, portada `100–116`.
- **Material:** negro mate con elevación suave y línea de progreso de bajo relieve.
- **Movimiento:** transición de datos corta (<500 ms), pulso leve en estado de reproducción, sin showreel exagerado.
- **Tecnología:** CSS + GSAP; ningún motor de partículas.
- **Variantes:** white invierte monocromía, black fiel a referencia, transparent deja bloques tipográficos anclados a carátula, glass cápsula ahumada, accent solo barra e indicador en color.
- **Validación:** los radios prominentes y escala tipográfica son esenciales; no volver al radio genérico de 12px.

### 13 — Pastel Layered Disc · capas menta

**Referencia.** Diseño pastel menta con varios rectángulos blancos translapados en escalera: bloque alto izquierdo con disco pequeño, bloque superior derecho con título y artista y franja inferior para reproducción. Sombras suaves, turquesa y blancos.

**Objetivo:** overlay lúdico de capas físicas de papel/acrílico.

- **Geometría:** `500 × 205`; panel izquierdo alto, panel superior derecho semitranslúcido, panel inferior en blanco opaco; elementos con solapes claros.
- **Material:** capas de diferente elevación y sombras de contacto, disco CD negro/plata con agujero central.
- **Movimiento:** discos y capas entran escalonados; el CD gira durante playback.
- **Tecnología:** CSS/SVG + GSAP.
- **Variantes:** white menta y blanco original, black menta profundo sobre negro, transparent deja capas mínimas y CD, glass capas esmeriladas, accent turquesa o color del álbum.
- **Validación:** deben ser tres planos distinguibles, no una sola tarjeta rectangular.

### 14 — Watch Widget Set · seis microinterfaces independientes

**Referencia.** Hoja con **seis** widgets cuadrados en blanco/negro, todos de radios grandes y contenidos distintos: (A) reloj grande y reproductor mini; (B) progreso circular tipo esfera; (C) volumen grande; (D) controles sobre esfera oscura; (E) indicador de pista con anillo parcial; (F) círculo con información y llamada a reproducir.

**Objetivo:** subcolección de micro-overlays para streamers que necesitan ocupar poquísimo espacio.

- **Geometría:** `230 × 230` por widget; también `160 × 160` compact con reglas de truncado.
- **Subestilos obligatorios:** `14A-clock`, `14B-radial`, `14C-volume`, `14D-orb`, `14E-next`, `14F-playlist`.
- **Adaptación:** en OBS 14C se muestra solo si el nivel de volumen es real; 14F no ofrece botón de reproducir al espectador sino etiqueta de estado/playlist.
- **Movimiento:** progreso como arco SVG, microdesplazamiento entre estados; sin movimientos largos para widgets pequeños.
- **Tecnología:** CSS + SVG + GSAP ligero.
- **Variantes:** cada subestilo admite white/black/transparent/glass/accent, manteniendo las proporciones internas. Si se implementan las seis, esta familia aporta **30 combinaciones adicionales**.
- **Validación:** cada composición debe reconocerse como reloj/widget diferente; no convertir las seis en copias con cambio de contenido.

### 15 — Warm Waveform Card · tarjeta cálida con audio

**Referencia.** Tarjeta cálida blanca/translúcida, bordes muy suaves, carátula cuadrada ilustrada a la izquierda con sol naranja y dunas. A la derecha, pequeño rótulo `NOW PLAYING`, canción y artista, waveform fino naranja/gris, duraciones y fila inferior con un botón central naranja grande, círculo luminoso y controles minimalistas.

**Objetivo:** música expresada visualmente mediante una línea de onda elegante y cálida.

- **Geometría:** `550 × 270`, carátula 135–150, columna de datos generosa.
- **Material:** vidrio nacarado, sombras largas muy suaves, naranja cálido de alto rango y luminosidad suave alrededor del botón.
- **Movimiento:** waveform reactivo solo con señal de audio real disponible; fallback como forma decorativa animada suavemente, sin pretender sincronización. Respetar reduce-motion.
- **Tecnología:** SVG/Canvas 2D o PixiJS si hay muchos segmentos, GSAP en cambio de pista.
- **Variantes:** white original, black negro cálido/coral, transparent waveform y carátula flotantes, glass vidrio coral, accent naranja original o personalizado.
- **Validación:** waveform y naranja deben dominar; una barra simple roja no sustituye a la onda.

### 16 — Extended Circular Cover · portada gigante al extremo

**Referencia.** Tarjeta horizontal gris azulado con portada redonda **enorme** sobresaliendo a la izquierda; el panel aparece como prolongación de la portada. Título/artista en la esquina superior izquierda del panel, pequeños símbolos de favorito/repetición/aleatorio, controles y slider. Barra de progreso integrada cerca del borde inferior.

**Objetivo:** formato widescreen compacto pero expresivo para stream de gameplay.

- **Geometría:** `675 × 195`, disco `220` y placa empieza por detrás cerca del centro del disco.
- **Material:** gris pizarra mate con sombras de oclusión y un leve brillo de borde.
- **Movimiento:** carátula giratoria opcional, entrada lateral del panel, progreso integrado con continuidad visual.
- **Tecnología:** CSS/SVG + GSAP.
- **Variantes:** white gris hielo; black pizarra profunda; transparent disco+texto/progreso; glass panel de vidrio; accent coral de iconos o tono derivado de carátula.
- **Validación:** disco debe superar en altura al panel y mantener el gran solape lateral.

### 17 — Overlay Style Picker · interfaz del editor, NO overlay público

**Referencia.** Panel de configuración oscuro con dos grupos: `PLAYER APPEARANCE` y `COVER APPEARANCE`. El primer grupo presenta tarjetas-miniatura en cuadrícula para Compact, Boxy, Gallery, Minimal, macOS, Shell y otra opción deshabilitada. La selección actual muestra borde/acento violeta. El segundo grupo elige Square, Canvas, Vinyl o None.

**Objetivo:** que el streamer configure el overlay y vea previsualizaciones auténticas, no iconos genéricos.

- **Layout:** sidebar de categorías, panel de galería, inspector de propiedades y preview de OBS con fondo claro/oscuro/video simulado; cada thumbnail renderiza el diseño correspondiente en miniatura.
- **Controles:** familia (16 + subestilos), variante de tema, fuente del acento, color personalizado, portada, información visible, motion, escala, posición, padding, duración de transición, modo preview, copiar URL/HTML local si aplica.
- **Estados:** seleccionado, hover/focus, deshabilitado, error de fuente, vista previa cargando; teclado y accesibilidad.
- **Tecnología:** Svelte + CSS; thumbnails estáticos/cacheados, **sin 16 renderizadores WebGL simultáneos**; instancia activa en la vista principal.
- **Variantes:** el selector puede adoptar tema claro/oscuro; **no** producir 5 overlays de este panel, puesto que es el configurador.
- **Validación:** cambiar familia y variante realmente cambia la preview; cada miniatura conserva el aspecto del estilo final.

---

## 5. Diseño visual y tokens

Las recomendaciones de Apple Human Interface Guidelines se aplican como **principios** (claridad, jerarquía, consistencia, adaptación, contraste y movimiento significativo), no como obligación de que todas las referencias parezcan componentes nativos de iOS.

```css
:root {
  --brand: #F0213A;
  --bg-app: #F7F7F8;
  --bg-panel: #FFFFFF;
  --bg-subtle: #F0F0F2;
  --text-primary: #17171A;
  --text-secondary: #606068;
  --border-default: #E5E5E8;
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --radius-control: 8px;
  --radius-card: 12px;
  --radius-modal: 16px;
  --overlay-scale: 1;
  --overlay-accent: #F0213A;
}
```

**Importante:** estos tokens son base de **editor** y valores por defecto; los overlays **pueden y deben sobrescribir radios, colores, sombras y tipografías** cuando su estética original lo demande: cápsula negra de radio muy alto, casete retro, mecánico metálico, menta pastel, vidrio azul, etc.

- **Legibilidad:** apuntar a contraste WCAG AA para texto funcional (4.5:1 normal, 3:1 grande); verificar sobre escenarios de fondo real en transparent y glass.
- **Tipografía:** Inter como fallback uniforme; estilo retro permite display lettering específico embebido con licencia, sin sacrificar texto dinámico.
- **Portadas:** `object-fit: cover`; nunca deformar relaciones de aspecto; transición sin parpadeo.
- **Desbordamiento:** preparar max-width de título, elipsis y desplazamiento opcional; no cortar el contenido por el borde del viewport.
- **Safe area:** mínimo 12–24px de margen alrededor de piezas que sobresalen; el frame de OBS debe incluir sombras y discos sobresalientes.
- **UI de edición:** controles de configuración deben tener tamaños tocables/clicables generosos; la UI **pública del overlay** no precisa targets de interacción si es informativa.

---

## 6. Movimiento de alta gama: reglas por material

| Material / estilo | Lenguaje cinético | Límite |
|---|---|---|
| Vinilo (01, 04, 06, 08, 13, 16) | Rotación continua mientras `isPlaying`; desaceleración al pausar | No girar si `paused` |
| Glass (02, 07, 15) | Reflejos de baja intensidad, profundidad, aparición suave | Sin glows agresivos ni blur excesivo |
| Carrusel (05, 10) | Cambio de carátula con profundidad real; stagger | No marear con movimiento perpetuo |
| Casete (09) | Movimiento de bobinas coherente con reproducción | No usar partículas para fingir cinta |
| Micro widgets (14) | Animación de progreso y microestados | Sin parallax que dificulte lectura |
| Waveform (15) | Amplitud/FFT real, o animación ornamental explícita | Nunca afirmar reactividad sin datos |
| Controles visibles (todos) | Estados gráficos, no controles falsos en OBS | Editor sí puede controlar demo |

### 6.1 Máquina de estados de animación

```text
load -> loading -> playing <-> paused
                         |
                         +--> track-change --> playing
                         |
                         +--> disconnected
```

Requisitos: cancelar GSAP timelines viejos en cada cambio de pista; liberar recursos de Three/Pixi al desmontar; desacoplar la frecuencia de datos de la frecuencia de renderizado; detener bucles cuando la escena no se ve; modo `prefers-reduced-motion` / `motion=off`.

### 6.2 Presupuestos de rendimiento orientativos (deben probarse)

- Vista de overlay simple: objetivo **60 fps si la máquina lo permite**, con degradación a 30 fps en animaciones no esenciales. No prometer 60 fps idénticos en cualquier PC/OBS.
- En modo galería: **máximo un renderizador complejo activo**; el resto son previews cacheadas.
- Limitar pixel ratio de WebGL a un valor configurable, con valor por defecto moderado (p. ej. máximo 1.5–2).
- Pausar render si la pestaña o fuente no está visible cuando el navegador lo notifique; liberar `WebGLRenderer`, texturas y listeners al cerrar el estilo.
- En efectos de partículas usar un presupuesto dinámico; desactivar en equipos limitados.
- Deshabilitar sombras 3D de coste alto y postprocesado intenso en el modo `low`.
- Ejecutar una prueba de estrés mientras OBS captura 1080p60 y TikLiveTTS ejecuta el chat; medir CPU, GPU, RAM y latencia, no solo FPS aislados.

---

## 7. Galería, editor y exportación

### 7.1 Galería obligatoria

Mostrar una ficha por familia, con miniatura **fiel**, identificador `01–16`, descripción en una frase, tecnología usada y botones para probar las cinco variantes. La familia 14 despliega sus seis subestilos. La familia 17 es el **configurador**.

En el modo galería pueden aparecer fondos de demostración para mostrar el glass, pero **ninguno debe incluirse por defecto dentro del overlay transparente para OBS**. Tener tres escenarios de prueba: fondo claro, fondo oscuro y gameplay en movimiento.

### 7.2 Editor

- Modos `visual`, `performance` y `data`.
- Selector de familia y variante + selector de acento por referencia/álbum/marca/custom.
- Ajustes independientes de portada, sombra, intensidad glass, tamaño, escala, posición, visibilidad de metadatos, animación.
- Simulador de canciones con nombres largos, carátulas ausentes, petición de espectador, pausa, desconexión y lista siguiente.
- Inspector de consumo visible, con aviso cuando el estilo use Three.js/PixiJS.
- Previsualización pixel-perfect con canvas transparente y guías opcionales.

### 7.3 Salidas

- **`MusicOverlayStudio.html`**: archivo único, autónomo para galería, editor y modo overlay.
- **`TikLiveTTS_Music_Overlays_Especificacion.md`**: documento técnico maestro (este archivo).
- URL/query de configuración reutilizable dentro de la app/OBS.
- Persistencia: URL serializada o configuración de TikLiveTTS; en fichero local, las preferencias del usuario podrían necesitar guardarse externamente según políticas del navegador. No depender de `localStorage` como única fuente de verdad.

---

## 8. Plan de construcción y control de calidad

### Fase A — Dirección de arte fiel
1. Extraer y agrupar las 16 referencias de player y la 17 de picker; anotar silueta/material/layout.
2. Construir **primero cada diseño base**, sin tema obligatorio ni efectos desmedidos.
3. Comparar diseño con referencia en paralelo y aprobar composición. Evitar tarjetas provisionales de forma genérica.

### Fase B — Materiales y animación
4. Implementar assets de calidad: surcos reales, brazos/casete, mecanismos, recortes, portadas de ejemplo con licencia.
5. Añadir animación que corresponda al objeto real o gesto del estilo; usar Three solo donde la profundidad aporte calidad clara.
6. Optimizar con fallback 2D para cada overlay 3D.

### Fase C — Variantes
7. Crear `white`, `black`, `transparent`, `glass`, `accent` **sin cambiar la silueta base**.
8. Verificar legibilidad en tres fondos y escalas 100% / 75% / 50%.
9. Revisar iconografía, contraste, sombras, antialiasing y recortes.

### Fase D — Integración
10. Vincular datos musicales reales disponibles; no simular fechas, tiempos o solicitudes en modo producción.
11. Montar Style Picker con previews fieles y configuración persistente.
12. Empaquetar HTML y verificar sin internet, en OBS Browser Source y en la ventana de TikLiveTTS.

### 8.1 Criterios de aceptación

- [ ] Las 16 familias son visualmente distintas y reconocibles frente a sus referencias.
- [ ] Los seis subestilos Watch se implementan por separado.
- [ ] Cada familia tiene cinco variantes coherentes y legibles.
- [ ] Transparent presenta alfa real.
- [ ] Glass tiene fallback sin backdrop blur.
- [ ] No hay controles de reproducción falsos para espectadores.
- [ ] Track-change funciona sin parpadeos ni fugas de recursos.
- [ ] Animaciones paran cuando playback se pausa y respetan reduced motion.
- [ ] Faltantes de información no se rellenan con datos inventados.
- [ ] Un solo HTML abre la galería sin red y el modo overlay es compatible con OBS.
- [ ] Los estilos 3D ofrecen degradación/fallback para equipos sin WebGL.
- [ ] Hay auditoría de CPU/GPU/RAM en condiciones similares a un stream real.

---

## 9. Reglas negativas — lo que el implementador NO debe hacer

1. No replicar 16 veces el mismo panel con carátula cuadrada.
2. No sustituir mecanismos 3D por simples círculos planos cuando son el elemento central de la referencia.
3. No introducir rojo TikLiveTTS en todos los diseños por defecto.
4. No imponer animaciones vistosas que contradigan un estilo minimalista.
5. No poner un fondo blanco, negro o gradiente dentro del modo `transparent` si el usuario quiere el juego visible.
6. No cargar todos los motores gráficos a la vez, ni crear un canvas WebGL por miniatura.
7. No mostrar cola musical, petición de espectador ni barras de audio como datos reales si no los entrega la integración.
8. No usar marquesinas constantes para títulos cortos.
9. No confundir el panel de editor (referencia 17) con un overlay destinado a OBS.
10. No entregar un HTML que necesite una CDN para arrancar si se promete autónomo.

---

## 10. Enlaces técnicos de referencia

- Apple Human Interface Guidelines — https://developer.apple.com/design/human-interface-guidelines/foundations
- OBS Browser Source — https://obsproject.com/kb/browser-source
- Three.js — https://threejs.org/docs/
- PixiJS — https://pixijs.com/
- GSAP — https://gsap.com/docs/v3/GSAP/
- Svelte — https://svelte.dev/docs
- p5.js — https://p5js.org/reference/
- Lottie-web — https://github.com/airbnb/lottie-web
- canvas-confetti — https://github.com/catdad/canvas-confetti
- Anime.js — https://animejs.com/documentation/
- Popmotion — https://popmotion.io/

**Nota final:** este documento especifica qué hay que construir. No pretende que el Markdown por sí solo renderice Three.js, PixiJS ni produzca overlays interactivos. El resultado ejecutable descrito es `MusicOverlayStudio.html`, que necesita su implementación y empaquetado posteriores.
