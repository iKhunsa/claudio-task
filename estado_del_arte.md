# Estado del arte: alternativas a TikTok LIVE Studio (stream key, protocolo webcast y eventos del live)

Fecha de referencia: 2026-10-16. Autor: Subagente B (solo investigación, sin código de la app).
Alcance: decidir si una app propia en Rust (Windows) equivalente a TikTok LIVE Studio es viable: obtener URL/stream key RTMP, capturar y codificar, y leer eventos del live.

## 0. Resumen ejecutivo

1. Hay dos problemas distintos y conviene separarlos:
   - **Publicar** (RTMP): la parte de captura/codificación/empuje RTMP es ingeniería normal y viable. Lo difícil es obtener credenciales de ingest "como LIVE Studio". La vía oficial documentada es que el usuario, con una cuenta elegible, copie Server URL y Stream Key desde LIVE Center (o "Cast to PC") y los pegue en la app. Esa vía es viable sin evadir nada.
   - **Leer eventos** (chat, regalos, etc.): el protocolo webcast (WebSocket + protobuf) está bien documentado por la comunidad. Lo frágil es la firma de requests (X-Bogus, X-Gnarly, msToken) y los bloqueos por dispositivo/IP. Hoy casi todos los clientes dependen de un servicio de firma de terceros (Euler Stream). Un proyecto en Rust (PirateTok/live-rs) afirma funcionar sin firma, solo con la cookie `ttwid`; no pude verificarlo en vivo.
2. El único proyecto que "genera stream key como LIVE Studio" y sigue activo (Loukious/TikTokStreamKeyGenerator) suplanta al cliente oficial: user-agent de LIVE Studio, `device_register` con identificadores falsos, cabeceras `x-argus`/`x-ladon` generadas por un firmador de pago de código cerrado (RapidAPI) y metadatos SEI "firmados" dentro del vídeo (`ttls_live_scene: live_studio`). Con las reglas fijas del encargo (no evadir antiabuso, no falsificar elegibilidad) eso es **no viable**. El propio autor reconoce que sin esa firma la cuenta recibe restricción de visibilidad por "integrity/authenticity".
3. La ruta "partner" (Streamlabs, SE.Live, Restream) existe, pero la elegibilidad la decide TikTok y la API de Streamlabs que usa el generador es privada. Para una app propia, solo sería limpia si TikTok la admite como partner (no pude verificar un programa público; ver huecos).
4. Mantenimiento: los repos de eventos de Isaac Kogan (Python y Node) están muy activos (99 y 112 commits en 12 meses) pero cambiaron en julio de 2026 de MIT a una "AGPL modificada" con cláusulas 18 a 21. El port C# está abandonado (último commit 2025-01-28) y el port Rust de jwdeveloper está archivado (último commit 2024-08-31).
5. Riesgos reales: restricciones "Integrity and authenticity" reportadas en issues, revocación de acceso LIVE, y términos de TikTok que prohíben scraping automatizado e ingeniería inversa. Ver sección 7.

## 1. Método, convenciones y límites

Niveles de confianza usados en cada afirmación:
- **[V]** verificado en código o historial git leído localmente (se indica ruta/commit).
- **[F]** verificado en documentación oficial (TikTok, Streamlabs, StreamElements, Euler) leída durante esta investigación.
- **[T]** reportado por terceros (issues, blogs, READMEs de otros proyectos). No lo comprobé yo.
- **[I]** inferido por mí a partir de lo anterior.

Qué hice: leí los cuatro repos locales (clones ampliados con `git fetch --depth=2000` en tres de ellos), cloné de forma anónima en `/tmp/claude-0/work/sota/` los repos adicionales (Streamlabs generator, obs-multi-rtmp de Loukious, TikTokLiveRust, gotiktoklive, TikTok-Webcast-Protobuf, PirateTok/live-rs), y leí issues por la web de GitHub. No ejecuté ni instalé nada de esos repos.

Límites:
- El egreso del sandbox bloquea `tiktok.com` y `eulerstream.com` (403 del proxy). Las páginas oficiales de TikTok y Euler las leí a través de la herramienta Exa (copias de texto). No pude abrir la API de GitHub (solo la web), así que el recuento de issues viene de las páginas HTML.
- No hay captura de tráfico real. Todo lo que dependa de qué hace hoy el servidor de TikTok figura en la sección 11 como hueco.
- Las fechas de commits y el número de commits en 12 meses (desde 2025-10-16) salen de `git log` local. Los clones superficiales pueden subestimar el total histórico, no el último año (todos cubren desde antes de esa fecha, salvo PirateTok y TikTok-Webcast-Protobuf donde el clon cubre todo lo clonado).
- Nada de esto es asesoría legal.

## 2. Mapa del ecosistema

```
PUBLICAR (RTMP)
  Oficial:   LIVE Center / "Cast to PC" -> Server URL + Stream Key (manual)  [F][T]
             TikTok LIVE Studio (Electron, Windows)                           [F]
  Partner:   Streamlabs Desktop, SE.Live (StreamElements), Restream           [F]
             -> OAuth del socio, TikTok decide elegibilidad
  No oficial:
     A) Loukious/StreamLabsTikTokStreamKeyGenerator -> API privada de Streamlabs (token)  [V]
     B) Loukious/TikTokStreamKeyGenerator -> suplanta LIVE Studio (webcast/room/create)
        + firmador RapidAPI + proxy FFmpeg con SEI firmado                                [V]
     C) forks/ports de A y B: fskhri/TiktokGenerateRTMP (Android), antiantidev/labsgen-tiktok (Electron),
        Loukious/obs-multi-rtmp (plugin OBS con token Streamlabs)                         [T][V]

LEER EVENTOS (webcast: WebSocket + protobuf)
  Esquema protobuf:  isaackogan/TikTok-Webcast-Protobuf (tiktok-live-proto, TikTokLiveProto)
  Clientes:          Python (TikTokLive), Node (TikTok-Live-Connector), C# (TikTokLiveSharp),
                     Java (jwdeveloper), Go (steampoweredtaco/gotiktoklive),
                     Rust (jwdeveloper/TikTokLiveRust archivado; PirateTok/live-rs)
  Firma:             Euler Stream (api.eulerstream.com, servicio de pago/freemium) o firma propia
  Comerciales:       Euler Stream, TikTool (tik.tools), Toktik, TikFinity
```

## 3. Fichas por repo

### 3.1 Loukious/TikTokStreamKeyGenerator (Python, "variante LIVE Studio")

Ruta local: `/home/user/loukious/tiktokstreamkeygenerator`. HEAD `373407f` (2026-09-07). Remoto: https://github.com/Loukious/TikTokStreamKeyGenerator

**1. Qué resuelve.** Crea una sala LIVE con los endpoints de TikTok LIVE Studio y obtiene el `rtmp_push_url` (servidor y clave). En la versión actual no entrega esa URL al usuario: levanta un proxy RTMP local con FFmpeg, OBS envía al proxy y el proxy reenvía a TikTok inyectando metadatos SEI "firmados" en el vídeo. [V] `TiktokStreamKeyGenerator.py:891-908` (`POST webcast/room/create/`, `_apply_room_info` en 482-498 separa `stream_url.rtmp_push_url` en base y clave), `Libs/ffmpeg_sei_proxy.py` (inyección SEI tipo 100 en H.264/HEVC dentro de FLV). README: "TikTok closes direct OBS connections immediately, and using the real TikTok URL/key without the LIVE Studio-style stream metadata can lead to a LIVE visibility restriction for integrity/authenticity." [V]

Extras: dual layout (`multi_stream_scene=1`), reanudar sala (`webcast/room/continue/`), heartbeats de anfitrión (`webcast/room/ping/anchor/`), estadísticas en vivo, lista de violaciones (`webcast/eco/violation_list/`), subida de miniatura. [V]

**2. Autenticación y datos que pide.**
- Login dentro de la app por QR (`/passport/web/get_qrcode/`, `/passport/web/check_qrconnect/`) o flujo OIDC (`/passport/oidc/prepare/`, `/passport/oidc/login/`, URL `ucenter_web/onetap_auth`). [V] `TiktokStreamKeyGenerator.py:2039-2435`.
- Alternativa: importar un JSON de cookies exportado del navegador (README, sección "Logging In With Exported Cookies"). Cookies de sesión reconocidas: `sessionid`, `sessionid_ss`, `sid_tt`, `sid_guard`, `multi_sids` (línea 2440). [V]
- Identidad de dispositivo: `device_id`/`install_id` registrados contra `/service/2/desktop/device_register/` con MAC, serie y UUID aleatorios y un "fingerprint" de escritorio inventado (modelos de placa base y resoluciones de una lista). [V] `Libs/device_gen.py:44, 155-255`.
- Pide además una **clave RapidAPI** del firmador del propio autor. [V] README y `Libs/signers.py:44-45`.
- Credencial sensible guardada: archivo de cookies en disco (el README avisa de que son credenciales vivas).

**3. Endpoints, dominios, cabeceras y firma.**
- Dominios: hosts webcast resueltos por TNC (`tnc0-normal-alisg.tiktokv.com/get_domains/v4/`, `tnc16-platform-*.tiktokv.com`), passport en `api16-normal-*.tiktokv.eu/.com/.us`, `api.tiktokv.com`, y comprobación de versión en `tron-sg.bytelemon.com/api/sdk/check_update`. [V] `Libs/domain_routing.py`, `Libs/device_gen.py:44-48`.
- Rutas webcast usadas: `room/create`, `room/create_info`, `room/continue`, `room/ping/anchor`, `room/anchor_pre_finish`, `room/info`, `room/upload/image`, `room/hashtag/list`, `ranklist/online_audience`, `game/basic/create_info`, `game/studio/realtime_stats`, `game/studio/trends_stats`, `anchor_tool/threshold/check`, `eco/violation_list`. [V]
- Parámetros que imitan al cliente oficial: `aid=8311`, `app_name=tiktok_live_studio`, `channel=studio`, `device_platform=windows`, `live_mode=6`, `version_code` (versión real de LIVE Studio descubierta online; mínimo codificado 1.27.0). [V] `TiktokStreamKeyGenerator.py:595-620`, `Libs/device_gen.py:41`.
- Cabeceras: user-agent `TikTokLIVEStudio/1.27.0 Chrome/136 Electron/36.4.0`, `sdk_aid: 8311`, `X-SS-DP`, `x-ss-stub` (MD5 del cuerpo), `x-tt-store-region`, y las firmas `x-khronos`, `x-ladon`, `x-argus`. [V] `Libs/signers.py:137-143`.
- Firma: **no es local**. Desde el commit `3d349b3` (2026-05-18) se borraron las librerías locales (`Libs/ttencrypt.py`, `Simon.py`, `Sm3.py`, `signature.py`...) y la firma pasa por un servicio alojado en RapidAPI (`tiktok-live-studio-api-signer1.p.rapidapi.com`) tanto para cabeceras como para la firma de fotogramas SEI (`Libs/XFrameSign.py`). El autor dice que no publica el algoritmo "because it could be abused" (issue #68). [V][T]
- Dato clave: el payload SEI lleva `"ttls_live_scene": "live_studio"` y un `signResult` obtenido del firmador. Es decir, una atestación en banda de que el vídeo viene de LIVE Studio. [V] `Libs/ffmpeg_sei_proxy.py:217-229`.

**4. Qué se rompió y por qué (con evidencia).**

| Fecha | Evento | Evidencia |
|---|---|---|
| 2024-05 | Modo "spoofing móvil" y device ID aleatorio para cuentas sin acceso a LIVE Studio | commits `5af5f0a`, `0b6a1c4`, `64d4cf4` [V] |
| 2024-05-29 | Usuario reporta ban de cuenta, sospecha del spoofing; el autor dice que lo más probable es shadowban por generar muchas claves | issue #23 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/23 [T] |
| 2024-06-21 | "Download and update LIVE Studio to the latest version": TikTok bloquea versiones viejas de LIVE Studio. El autor admite que las firmas siempre fueron necesarias para el spoofing | issue #36 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/36 [T]; commit `aac634c` (2025-01-27) añade lectura de la última versión [V] |
| 2024-06-26 | "Make it more clear that spoofing is broken" | commit `4381d76` [V] |
| 2024-07-21 | Cartel "This project is discontinued!" | commit `c882d92` [V] |
| 2024-06/07 | Shadowban / sin tráfico sin spoofing | issues #32, #37, #42, #44 (listado de issues, sin leer cuerpos) [T] |
| 2025-02-01 | Último release antiguo v1.0.9, luego 15 meses de silencio | tag `v1.0.9` en `fb78647` [V] |
| 2025-11-06 | "Integrity and authenticity violation" nada más empezar el directo; la causa sospechada es que la clave se generó fuera de LIVE Studio | issue #59 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/59 [T] |
| 2026-05-14 | Restricción "Integrity & Authenticity" en todas las cuentas durante 4 meses; sin respuesta del autor | issue #66 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/66 [T] |
| 2026-05-18/26 | Reescritura total: se descarta la firma local, se adopta RapidAPI + proxy SEI; se retira el cartel "discontinued" | commits `3d349b3`, `000bfc8` [V] |
| 2026-07-10 | v2.0.0 (tags v2.0.0, v2.0.1 y v2.0.2 apuntan al mismo commit `31a6906`) y la URL real de TikTok se oculta al usuario | `31a6906`, `a8be441` [V] |
| 2026-07-29 | 429 del firmador: el plan gratuito permite "around 1 hour of streaming per day" | issue #68 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/68 [T] |
| 2026-09-15/10-15 | Error de RapidAPI desde Bielorrusia (#72, abierto) y pregunta sobre `priority_region` de Taiwán que cambia las firmas (#76, abierto) | https://github.com/Loukious/TikTokStreamKeyGenerator/issues/72 y /76 [T] |
| 2026-09-26 | El autor no ha verificado `multi_stream_url` en salas de LIVE Studio y dice que "TikTok allows continuing a stream only from the same device" | issue #75 https://github.com/Loukious/TikTokStreamKeyGenerator/issues/75 [T] |

**5. Mantenimiento.** Último commit 2026-09-07T15:59:40+01:00 (`373407f`). Commits en los últimos 12 meses: 41, todos de un único autor. Tags en el periodo: 9 (v2.0.0 a v2.1.5). Estado: **activo**, pero con riesgo de punto único (un autor, firmador cerrado). Ojo: varios commits de mayo de 2026 comparten el mismo timestamp exacto (`2026-05-19T14:29:46`), señal de historial reescrito o rebase [I]. Hay 247 estrellas y 69 forks según la web. [T]

**6. Licencia.** El repo **no contiene archivo LICENSE** y la web de GitHub no muestra licencia. [V] `ls` del repo; [T] página del repo. Implicación [I]: por defecto todos los derechos reservados; no se debe copiar código. Los hechos técnicos (nombres de endpoints, parámetros) son observaciones sobre un servicio ajeno, pero reutilizar el código o su estructura sin licencia no es seguro.

**7. Riesgos de ban/legales.** Los propios usuarios reportan baneo (#23), shadowban (#32, #37, #42, #44) y restricción de integridad (#59, #66). El README advierte que sin firma válida el directo "can lead to a LIVE visibility restriction". El autor rechazó en #23 que la herramienta cause baneos completos y habló de pérdida de acceso LIVE. Además, el diseño suplanta al cliente oficial, fabrica identidad de dispositivo y falsifica atestación: choca con las cláusulas de TikTok sobre ingeniería inversa y evasión (sección 7). Issue #30 (2024-06) reporta un falso positivo antivirus `Trojan:Win32/Wacatac.H!ml` sobre el ejecutable empaquetado. [T]

**8. Lecciones para Rust.**
- Separar con claridad "obtener credencial" de "publicar vídeo". Lo segundo es independiente de TikTok y portátil.
- Un proxy RTMP local es un patrón válido para inyectar metadatos legítimos (FLV tag rewriting), pero aquí se usa para una atestación que no se debe falsificar.
- Detalles útiles y neutrales: sincronizar hora con la cabecera de servidor (`attach_webcast_ntp_t0`), reintentos entre hosts webcast alternativos ante 403/421/429/5xx, y limitar sondeos (commit `373407f` arregla un poller que agotaba el límite horario).
- Evitar: cualquier dependencia de un firmador de un solo tercero y de código cerrado para algo crítico.

### 3.2 Loukious/StreamLabsTikTokStreamKeyGenerator (variante con Streamlabs)

Clon: `/tmp/claude-0/work/sota/streamlabs_loukious`. HEAD `ef03fb2` (2026-04-01T15:38:25+01:00). Remoto: https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator

**1. Qué resuelve.** Con un token de Streamlabs Desktop, llama a la API privada de Streamlabs para arrancar un directo en TikTok y recibe `rtmp` y `key`. [V] `Stream.py:21-35` (`POST https://streamlabs.com/api/v5/slobs/tiktok/stream/start`, respuesta `response["rtmp"], response["key"]`), `end` en `/stream/{id}/end`, categorías en `/api/v5/slobs/tiktok/info`. README: "Streamlabs TikTok LIVE access is required." Autor y README recomiendan esta variante frente a la de LIVE Studio por tener menos piezas.

**2. Autenticación.**
- Opción "Load from PC": lee el `apiToken` de los logs LevelDB locales de Streamlabs Desktop (`%appdata%\slobs-client\Local Storage\leveldb\*.log`, regex `"apiToken":"..."`). [V] `StreamLabsTikTokStreamKeyGenerator.py:408-421`. Es leer el almacén de otra aplicación.
- Opción "Login from Web": OAuth con PKCE contra `https://streamlabs.com/slobs/login?...&code_challenge=...&port=<puerto local>` en el navegador por defecto, y canje del código en `https://streamlabs.com/api/v5/slobs/auth/data` (devuelve `oauth_token`). [V] `TokenRetriever.py:95-183`. Antes de abril de 2026 usaba Selenium/Chrome (`1cfd41e`, `55ec024`) y se reescribió en `00ddcaf`.
- Cabecera: `Authorization: Bearer <token>` y user-agent `StreamlabsDesktop/1.17.0 ... Electron/29.3.1`, es decir, se hace pasar por la app de escritorio de Streamlabs. [V]
- Datos del usuario: login de TikTok vía Streamlabs; nada de cookies de TikTok.

**3. Dominios y firma.** Solo `streamlabs.com`. Sin firmas de TikTok en el cliente; todo el trabajo con TikTok lo hace el backend de Streamlabs, que es socio de TikTok (nota de prensa: https://ir.logitech.com/news/news-details/2024/Logitechs-Streamlabs-Launches-TikTok-LIVE-Integration-to-Power-Seamless-Streaming/default.aspx [F]). No depende de Euler.

**4. Qué se rompió.** Obtención del token (Chrome, seleniumwire, Linux/Fedora): issues #94, #96, #100, #103, #104, #105 citados en mensajes de commit `3867014`, `7db9a96`, `99a6c4b`, `55ec024` [V]; #118 y #120 (Linux) [T]. Revocación del acceso LIVE:
- #114 (2025-12-24) "Can Go Live: False": el autor responde "Contact Streamlabs and ask why your live access is revoked." https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator/issues/114 [T]
- #122 (2026-02-18) 401 "You are not allowed to access this page at this time": el propio reportante descubre que le habían revocado los derechos. https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator/issues/122 [T]
- #134 (2026-06-25) "Failed to start stream: Expecting value": el autor lo atribuye a la cuenta y se resuelve con otra cuenta. https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator/issues/134 [T]
- #139 (2026-10-15): hay que generar una clave nueva cada vez. https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator/issues/139 [T]
- Política de TikTok desde 2025-07-07: quien use Streamlabs y emita menos de 50% de contenido de juegos puede perder el acceso; se puede volver a pedir tras 14 días. https://streamlabs.com/platforms/tiktok [F]. Streamlabs dice que "The approval process is entirely handled by TikTok" y que otros socios (StreamElements, Restream) tienen las mismas revocaciones. https://support.streamlabs.com/hc/en-us/articles/35259691237147-Streamlabs-x-TikTok-Streaming-Access-Revoked-Suspended [F]

**5. Mantenimiento.** Último commit 2026-04-01 (`ef03fb2`), 13 commits en 12 meses, última etiqueta v2.0.9 el mismo día. El autor sigue respondiendo issues en octubre de 2026 (133 issues en total según la web). Estado: **semiactivo**. [V][T]

**6. Licencia.** GPL-3.0 (`LICENSE.txt`). [V] Implicación: se puede estudiar y reimplementar la idea; copiar código obliga a publicar el derivado bajo GPL-3.0. En una app propia en Rust, usar solo los hechos de protocolo (URL y campos) sin copiar código evita el copyleft. Aparte, está el riesgo de condiciones de servicio de Streamlabs al usar su API privada con un cliente no oficial [I].

**7. Riesgos.** Dependencia de que TikTok mantenga la whitelist de Streamlabs para esa cuenta; revocaciones masivas documentadas por Streamlabs. Leer el token del LevelDB de otra app es técnicamente simple pero no es una API pensada para ello. El plugin `Loukious/obs-multi-rtmp` (GPL-2.0, commits `2b2c99f`, `d6281c2` de abril de 2025) guarda ese token en el campo de contraseña de OBS. [V]

**8. Lecciones para Rust.** Es la arquitectura más simple: OAuth con PKCE y puerto local de callback (fácil en Rust), una llamada REST y un RTMP push. Pero la elegibilidad no es tuya: depende del socio. Si la app propia quisiera esta vía legítimamente, hay que pedir al socio (o a TikTok) integración oficial, no copiar su token.

Forks relacionados: `fskhri/TiktokGenerateRTMP` (Android/Flutter, basado en el generador de LIVE Studio, https://github.com/fskhri/TiktokGenerateRTMP) y `antiantidev/labsgen-tiktok` (Electron, token de Streamlabs, https://github.com/antiantidev/labsgen-tiktok); no los analicé a fondo. [T]

### 3.3 isaackogan/TikTokLive (Python)

Ruta local: `/home/user/isaackogan/tiktoklive`. HEAD `fc73b8f` (2026-09-09T23:33:25-04:00, "release v7.0.1"). Remoto: https://github.com/isaackogan/TikTokLive

**1. Qué resuelve.** Cliente de solo lectura de los eventos de un directo ajeno o propio (chat, regalos, likes, follows, shares, viewers, batallas, etc.) dado el `@unique_id`. README: "No login, credentials or app are required" y "This is not a production-ready API. It is a reverse engineering project." [V] `README.md`.

**2. Autenticación.** Anónima por defecto. Opcional `client.web.set_session(session_id, tt_target_idc)` que fija cookies `sessionid`, `sessionid_ss`, `sid_tt`, `tt-target-idc` y `user_is_login=true`. [V] `TikTokLive/client/web/web_base.py:123-142`. Cuando hay sesión, el cliente avisa en el log: "Sending session ID to sign server for WebSocket connection. This is a risky operation." [V] `web/routes/fetch_signed_websocket.py`. Para enviar chat exige sesión (`send_room_chat.py`).

**3. Endpoints y firma.**
- TikTok: `https://www.tiktok.com/api-live/user/room/` (room id), `https://www.tiktok.com/@{user}/live` (scrape HTML), `https://webcast.tiktok.com/webcast/room/info/`, `.../room/info_by_user/`, `.../room/check_alive/`, `.../gift/list/`. [V] `web/routes/*.py`.
- Parámetros por defecto del cliente web: `aid=1988`, `app_name=tiktok_web`, `device_platform=web_pc`, y presets aleatorios de ubicación, dispositivo y pantalla elegidos al arrancar. [V] `web/web_settings.py:1-60`.
- WebSocket (v7): se pide a Euler Stream (`https://api.eulerstream.com`) la URL firmada y la primera respuesta protobuf de `/webcast/im/fetch`. De esa respuesta se extraen `push_server`, `route_params` y `cursor` para construir la URI del WebSocket. [V] `web/routes/fetch_signed_websocket.py`, `ws/ws_utils.py:build_webcast_uri`.
- Para otras rutas webcast: `TikTokSigner.webcast_sign` envía la URL, método, user-agent y opcionalmente `session_id` al servidor de firmas, quitando antes `X-Bogus`, `X-Gnarly` y `msToken`. [V] `web/web_signer.py`.
- Marco WebSocket: frames `WebcastPushFrame` (`payload_type`: `msg`, `ack`, `hb`, `im_enter_room`...), `need_ack`, latidos cada 10 s (`heartbeat_duration=10000`), gzip opcional, ping de websockets desactivado porque TikTok no responde pong. [V] `ws/ws_client.py`, `ws/ws_connect.py`.
- Dependencias: `EulerApiSdk==0.1.0`, `TikTokLiveProto==0.2.2`, `betterproto2`, `httpx`, `websockets`. [V] `pyproject.toml`.
- Límites de Euler: "freemium", se devuelve 429 al excederse; con API key (`SIGN_API_KEY` o `WebDefaults.tiktok_sign_api_key`) sube el límite. https://www.eulerstream.com/docs/api/rate-limits [F]

**4. Qué se rompió (con commits e issues).**
- 2022-10: se quitó la firma local de URLs y el envío de mensajes por requerir firmas (`0bb7348`, `d2e1674`) [V].
- 2024-09-13: cambio del enlace del servidor de firma (`ca91ed7`). 2024-09-19: `4a4fd24` "remove X-Bogus from default params to fix gift list" [V].
- 2025-03-26: se empieza a pasar el user-agent al servidor de firma (`ef0973b`) [V]. 2025-05-16: WS autenticados (`f0f67fc`) [V].
- 2025-07-06: `1a8e034` y comentario en código: las URLs firmadas caducan a los 30 s, y un 200 en vez de 101 es "detected by TikTok", por lo que se desactivan los reintentos del WebSocket [V] `ws/ws_connect.py`.
- 2025-09-16: error 500 del servidor de firma, issue #336 https://github.com/isaackogan/TikTokLive/issues/336 [T].
- 2025-10-24: 429 persistente por no usar clave, issue #342 https://github.com/isaackogan/TikTokLive/issues/342 [T].
- 2025-12-11: "User is offline" falso con el usuario en directo, issue #349, sin causa publicada https://github.com/isaackogan/TikTokLive/issues/349 [T].
- 2026-03-31: errores de protobuf y caídas del socket tras cambios de TikTok, issue #357, cerrado el 2026-04-30 con la migración a TikTokLiveProto v2/v3 y betterproto2 https://github.com/isaackogan/TikTokLive/issues/357 [T]; commits `c579a8e`, `4a7efec`, `e65a8c7` [V].
- 2026-04-28: se retira `curl_cffi` y se archivan `send_room_like/gift` (`63e897f`) [V].
- 2026-04-30: integración del SDK de Euler (`5ef674e`); 2026-06-08 el host pasa a `api.eulerstream.com` (`3d6d897`) [V].
- 2026-09-09/10: el servidor de firma devolvió un "ws-fallback" que rechazaba el handshake (400/200); la causa que da el mantenedor es que el sistema nuevo no reciclaba agentes; se arregla con v7.0.1 (`be3dceb`) en issues #376 y #379 https://github.com/isaackogan/TikTokLive/issues/379 [T][V].

**5. Mantenimiento.** Último commit 2026-09-09. Commits en 12 meses: 99 (55 `isaackogan`, 31 `Isaac Kogan`, el resto bots). Releases recientes: 7.0.0a1 (2026-04-30), 7.0.0b1 (2026-04-30), 7.0.0b2 (2026-07-22), 7.0.0 (commit `73c8391`, 2026-08-18), 7.0.1 (`fc73b8f`, 2026-09-09). Issues: 281 totales y 0 abiertos según la web (todos cerrados). Estado: **activo**, mantenido por una persona. [V][T]

**6. Licencia.** Hasta v6.2.0 era MIT (`git show v6.2.0:LICENSE` empieza con "MIT License"). El 2026-07-11 pasó a "Modified AGPL-3.0" (`c36191f`, `ecd9b43`, `27065fc`). `LICENSE` añade las secciones: 18 (permite enlazar/integrar el programa, sus bindings protobuf y esquemas en librerías o aplicaciones sin imponer AGPL, incluyendo "overlay services, games, bots"), 19 (la excepción NO aplica si se usa para un servicio SaaS/relay/API comercial cerrada), 20 (infracción revoca los permisos) y 21 (exceptuados de la 19: TikFinity/STV GmbH y Euler Stream Inc.). [V] `LICENSE:671-715`, `pyproject.toml` (`license = { text = "Modified AGPL-3.0" }`). Implicación para una app Rust de escritorio: portar la lógica es obra derivada de código bajo AGPL modificada; la excepción de la sección 18 cubre "downstream" que integren el código pero la redacción es ambigua para reimplementaciones; lo prudente es escribir desde cero a partir de capturas y del esquema propio, o pedir permiso por escrito. No es asesoría legal [I].

**7. Riesgos.** El README declara que no es producción. La librería advierte del riesgo de enviar `sessionid` al firmador de terceros. Euler reconoce en su documentación que las firmas existen porque TikTok trata el tráfico sin ellas como malicioso y que él evita responsabilidad legal dejando al usuario "solely responsible for data you scrape". https://www.eulerstream.com/docs/signatures [F]

**8. Lecciones para Rust.** Máquina de estados de conexión (fetch inicial, WS, ack/heartbeat, `need_ack`), tratar 200 en handshake como "bloqueado" sin reintentar, no depender de ping/pong de WebSocket, permitir proxy solo para HTTP o también WS, y aislar el firmador tras una interfaz sustituible (la propia librería permite `tiktok_sign_url` propio). Esquema protobuf versionado (v1/v2/v3) porque TikTok lo cambia sin aviso.

### 3.4 zerodytrash/TikTok-Live-Connector (Node.js/TypeScript)

Ruta local: `/home/user/zerodytrash/tiktok-live-connector`. HEAD `8a92330` (2026-09-22T00:41:17-04:00). Remoto: https://github.com/zerodytrash/TikTok-Live-Connector (el historial de 2026 lo hace principalmente Isaac Kogan: 77 commits `isaackogan` + 16 `Isaac Kogan` de los 112 del último año [V]).

**1. Qué resuelve.** Igual que la ficha anterior para Node: eventos de un directo con solo el `@uniqueId`. README: "TikTokLiveConnector reads TikTok's internal Webcast push service directly" y "This is not a production-ready API." [V]

**2. Autenticación.** Por defecto anónima. Nuevo objeto `session` (breaking change documentado en `BREAKING.md`): `session.cookie = { sessionId, ttTargetIdc }` y `session.oAuthToken` (token de una app OAuth de Euler). `authenticateWs: true` y `useMobile: true` requieren cookie. [V] `BREAKING.md` sección 4. Para WS autenticado exige además la variable de entorno `WHITELIST_AUTHENTICATED_SESSION_ID_HOST` igual al host del servidor de firma, para no filtrar el `sessionid` a un host no previsto. [V] `src/lib/web/routes/euler/fetch-signed-websocket-euler.ts`.

**3. Endpoints y firma.** Dominios `www.tiktok.com` y `webcast.tiktok.com`, parámetros `aid=1988`, `app_name=tiktok_web`, `device_platform=web_pc`, cookie `tt-target-idc=useast1a`. [V] `src/lib/web/defaults.ts`. Firma por Euler vía `tiktok-live-api-sdk`: la ruta `fetch-webcast-signature-euler.ts` quita `X-Bogus`, `X-Gnarly` y `msToken` de la URL y los vuelve a pedir; 403 se traduce en `PremiumFeatureError`. Desde el commit `1d88d87` (2026-06-14) la lista de regalos requiere firma ("swap gifts to use signing since now required"). [V]

**4. Qué se rompió.**
- 2025-10-04 `7c168cb`: arreglo del scraping HTML del room id, WebSocket móvil y sesiones autenticadas. 2025-05-12 `ea16e24`: opción `disableEulerFallbacks` para evitar que el fallback a Euler corra para usuarios gratuitos [V].
- 2025-12-12 `392dbbe`: `WebcastInRoomBannerMessage` con campos indefinidos (issue #286) [V].
- 2026-03-02 issue #310: 403 al pedir la lista de regalos; un comentarista dice que TikTok "locked down" el endpoint y el mantenedor responde (2026-04-03) que sin firma no es posible; sigue abierto https://github.com/zerodytrash/TikTok-Live-Connector/issues/310 [T].
- 2026-04-24/30: reescritura ESM (`tsdown`, `got`, `tiktok-live-proto`, protobuf v3, `19bf2e7`); breaking changes en `BREAKING.md` [V].
- 2026-05-27: salas en directo devueltas como `user_not_found`; el mantenedor confirma el 2026-06-14 que afectaba también a TikFinity y a Euler y que lo arregló en Euler, issue #323 https://github.com/zerodytrash/TikTok-Live-Connector/issues/323 [T].
- 2026-08-09 issue #327: 404 en `/webcast/im/fetch/` y duda de si bloquearon la IP en el firmador (abierto) [T]. 2026-08-14 issue #328: el evento `ENVELOPE` no llega en conexión anónima (abierto, sin respuesta) https://github.com/zerodytrash/TikTok-Live-Connector/issues/328 [T].
- 2026-09-10 `8577b74`: se conservan cabeceras de rate-limit de Euler (PR #331). 2026-09-16 `92dc913` arregla conexiones autenticadas [V].

**5. Mantenimiento.** Último commit 2026-09-22. Commits en 12 meses: 112. Tags en el periodo: 9 (v2.3.0-alpha2 a v2.5.0; v2.5.0 el 2026-09-16, `87fb46b`). Issues abiertos: 10 de 283. Estado: **activo**. [V][T]

**6. Licencia.** El repo nació MIT (2022, `b02426b`, "Copyright (c) 2022 zerodytrash") y desde el 2026-07-11 (`2742703`, `84efa3a`, `aaa51bb`) usa la misma "AGPL modificada" con secciones 18 a 21 que el repo Python. [V] Además, el issue #332 (2026-09-14) pregunta si se puede incluir la librería sin modificar en una app de escritorio de pago; Isaac Kogan responde el 2026-09-20 que "the 'hard line' is to not wrap the app up in a no-value-added SaaS API to sell TikTok LIVE connections", "You can treat this message as approval of your use" y que lo mismo vale para `tiktok-live-proto`. No es una enmienda formal de la licencia y no respondió a las preguntas sobre código fuente o el servicio de Euler. https://github.com/zerodytrash/TikTok-Live-Connector/issues/332 [T]

**7. Riesgos.** Igual que Python. El comentarista en #310 menciona que reponer datos de regalos vale una cuenta de pago de Euler de unos 50 USD al mes [T], coherente con el plan Business de 50 USD/mes de la web de Euler [T].

**8. Lecciones para Rust.** Patrón de "provider" (funciones sustituibles para obtener firma, URL de WebSocket y lista de regalos), whitelist del host que recibe credenciales, y tipos separados para sesión por cookie y por token OAuth (evitar pasar `sessionid` crudo a servicios externos).

### 3.5 frankvHoof93/TikTokLiveSharp (C#/Unity)

Ruta local: `/home/user/frankvhoof93/tiktoklivesharp`. HEAD `edbcb27` (2025-01-28T22:05:06+01:00). Remoto: https://github.com/frankvHoof93/TikTokLiveSharp

**1. Qué resuelve.** Port de TikTok-Live-Connector a C# y a Unity, con un esquema de mensajes propio (ver `tiktok_schema.proto`). [V] README.

**2. Autenticación.** Anónima; sin soporte documentado de sesión (issue #111 pregunta por login para contenido de edad restringida, sin respuesta) [T].

**3. Endpoints y firma.** `https://www.tiktok.com/`, `https://webcast.tiktok.com/webcast/`, firma por `https://tiktok.eulerstream.com/webcast/fetch`; ajustes para servidor de firma propio y API key. [V] `TikTokLiveSharp/Client/Config/Constants.cs:14-22`, `CHANGELOG.MD` 1.1.1.

**4. Qué se rompió.**
- 1.2.0 (2024-07-14): 403 al pedir regalos y room info; se quitó la cabecera `Origin`. [V] `CHANGELOG.MD`
- 1.2.1 (2024-12-14): TikTok cambió cabeceras; se quitó `version_code` y se añadió `Mozilla` al user-agent (PR #104, issue #103). [V]
- 1.2.2 (2025-01-28): se vuelve a poner `version_code`, con valor específico para cierto servidor. [V]
- Issue #108 (2025-01-28, abierto): el WebSocket responde 200 en lugar de 101 salvo que la query lleve `version_code=180800` (o 270000); en Sep 2025 otro usuario dice que en la versión Go se resolvió cambiando el user-agent del `wss://` para que no empiece por `Mozilla/`. https://github.com/frankvHoof93/TikTokLiveSharp/issues/108 [T]
- Issues #89 y #95 (2024-07): "Your IP or country might be blocked by TikTok"; el único apaño confirmado es conectar con el room id en lugar del nombre. https://github.com/frankvHoof93/TikTokLiveSharp/issues/95 [T]
- Issue #118 (2026-10-10): "does this still works?", sin respuesta. https://github.com/frankvHoof93/TikTokLiveSharp/issues/118 [T]

**5. Mantenimiento.** Último commit 2025-01-28. Commits en 12 meses: **0**. Último tag v1.2.2 (2025-01-28, `fe454cf`); cero tags en el último año. Estado: **abandonado** (más de 20 meses sin actividad; 100 issues, en su mayoría abiertos). [V][T]

**6. Licencia.** MIT (`LICENSE.md`, "Copyright (c) 2024 Frank van Hoof"). [V] Permite reutilizar código con aviso de copyright, pero está desactualizado.

**7. Riesgos.** Los de cualquier cliente anónimo; no hay avisos específicos de ban.

**8. Lecciones.** Es el mejor registro público de **cambios de cabeceras/parámetros que rompen el handshake** sin tocar la firma: `Origin`, `version_code`, prefijo del user-agent. Para una app propia: mantener esos parámetros fuera del binario (configurables) y detectar el caso "200 en lugar de 101".

### 3.6 Otros proyectos relevantes

#### 3.6.1 PirateTok/live-rs (Rust) y familia multilenguaje

Clon: `/tmp/claude-0/work/sota/piratetok_live_rs`. HEAD `ee60855` (2026-10-04T20:20:01+03:00), v0.5.0. https://github.com/PirateTok/live-rs

1. **Qué resuelve:** cliente de eventos del live en Rust, 64 tipos de evento decodificados. Afirma "No signing server, no x_bogus, no msToken. Just ttwid." [V] `README.md`, `docs/API.md`.
2. **Auth:** cookie `ttwid` anónima obtenida con un GET a `https://www.tiktok.com/` con hasta 8 reintentos separados 750 ms; cookies de sesión opcionales para datos con login (lista de espectadores). [V] `src/structs/config.rs:12-21`, `src/http/ttwid.rs`.
3. **Endpoints:** `GET /api-live/user/room?aid=1988&app_name=tiktok_web&device_platform=web_pc&...&uniqueId=<user>&sourceType=54` (room id) y WebSocket `wss://<cdn>/webcast/im/ws_proxy/ws_reuse_supplement/?version_code=180800&device_platform=web&aid=1988&live_id=12&room_id=...&identity=audience&heartbeat_duration=...&resp_content_type=protobuf&did_rule=3...` con cabeceras `Origin`/`Referer: https://www.tiktok.com/` y cookie `ttwid`. Envía heartbeat y `im_enter_room` protobuf al conectar y ack cuando `need_ack`. [V] `src/client.rs:245-283`, `src/websocket/connection.rs:64-120`. Sin servicio de firma de terceros. Un rechazo con cabecera `Handshake-Msg: DEVICE_BLOCKED` se trata como bloqueo del dispositivo. [V]
4. **Roturas:** sólo un issue (#1, 2026-07-03, "Error using on Windows", sin cuerpo visible) [T]. Proyecto creado el 2026-04-04 (primer commit `8962916`), con unos 6 meses de historia: no hay un historial largo de roturas que evaluar. **No verifiqué que el WebSocket sin firma funcione hoy** [I]: es el hueco más importante para la parte de eventos. Contrasta con TikTokLive v7, que sigue pidiendo a Euler la URL firmada y la primera respuesta de `/webcast/im/fetch`.
5. **Mantenimiento:** último commit 2026-10-04; 32 commits en total, todos dentro de los últimos 12 meses; 5 releases (v0.2.0 a v0.5.0). Estado: activo pero joven, un autor (Zmole Cristian). Hay réplicas en Go, Python, JS, C#, Java, Lua, Elixir, Dart, C, PowerShell y Shell bajo la organización PirateTok. [V][T]
6. **Licencia:** 0BSD (`LICENSE`, `Cargo.toml: license = "0BSD"`). Permisiva sin atribución obligatoria: es la fuente más reutilizable como referencia de arquitectura para Rust. [V] Los structs protobuf están escritos a mano con `prost`, sin `.proto`. [V]
7. **Riesgos:** ningún aviso legal en el README. **Evitar copiar** la rotación de user-agent aleatorio y de `ttwid` al recibir `DEVICE_BLOCKED` (`src/client.rs:202`, `reconnect.rs`): es una respuesta de evasión a un bloqueo de dispositivo y chocaría con las reglas del encargo [I].
8. **Lecciones:** política de reconexión testeable (`reconnect::judge`, presupuesto con reinicio tras sesión sana de 30 s), detección de conexión estancada (`stale_timeout`), `Endpoints` inyectables para pruebas offline contra servidores falsos, herramienta de grabación y reproducción de capturas (`src/bin/record_capture.rs`, `replay_capture.rs`).

#### 3.6.2 jwdeveloper/TikTokLiveRust

Clon: `/tmp/claude-0/work/sota/tiktokliverust`. HEAD `5b7950c` (2024-08-31T20:28:19+03:00). https://github.com/jwdeveloper/TikTokLiveRust
- Port Rust del connector; usa `https://tiktok.eulerstream.com/webcast/sign_url` y soporta API key (`eb09f64`). [V] `src/core/live_client_http.rs:20-22`.
- Estado: **abandonado**. Cero commits en 12 meses; la web de GitHub indica "archived by the owner on Jul 14, 2025". [V][T]
- Licencia: MIT (`license.txt`), `Cargo.toml` declara MIT OR Apache-2.0. Crate `tiktoklive` 0.0.19 (2025-03-26 según docs.rs). [V][T]
- Usa `reqwest 0.11`, `tokio-tungstenite 0.16` (antiguos) y protobuf con `protobuf-codegen`. Útil sólo como referencia histórica.

#### 3.6.3 steampoweredtaco/gotiktoklive (Go, fork de Davincible)

Clon: `/tmp/claude-0/work/sota/gotiktoklive`. HEAD `3be247c` (2025-09-14T15:56:20-06:00): "Fix user agent for websocket to tiktok to work again". Sin commits en los últimos 12 meses. README en estado "alpha". MIT (origen "Copyright (c) 2018 Ahmadreza Zibaei"). Usa servidor de firma configurable con clave y secreto (`const.go:24` `/webcast/fetch/`) y avisa: "your IP or country might be blocked by TikTok or Signer service". [V] Confirma el síntoma del prefijo `Mozilla/` del user-agent del `wss://` (misma pista que C# #108). Estado: semiabandonado.

#### 3.6.4 isaackogan/TikTok-Webcast-Protobuf (esquema `tiktok-live-proto`, `TikTokLiveProto`)

Clon: `/tmp/claude-0/work/sota/webcast_proto`. HEAD `8e7ccad` (2026-09-28). https://github.com/isaackogan/TikTok-Webcast-Protobuf
- Fuente canónica del esquema (`src/v1`, `v2`, `v3`, `slim`, `full`) con bindings TypeScript, Python y JVM; Go y Rust figuran como "Planned". Se presenta como "clean-room reverse engineering ... for interoperability" y cita Google v. Oracle. [V][T] README.
- Mantenimiento: actividad continua (107 commits en el clon superficial, todos del último año).
- **Licencia con contradicción:** `LICENSE` es AGPL-3.0 con las mismas secciones 18 a 21 (permite integración comercial salvo SaaS/relay), pero el README dice "intended for **non-commercial use only**". [V] Para una app comercial o distribuida, esa discrepancia debe aclararse con el autor antes de depender del esquema. Alternativa: derivar un esquema propio por captura (el crate PirateTok lo hace a mano, bajo 0BSD).

#### 3.6.5 Euler Stream (servicio) y competidores

- Euler Stream (https://www.eulerstream.com): "The top 3rd-party TikTok LIVE API", servidor de firmas para todas las librerías, API REST/WebSocket gestionada. Planes según la página del repo SDK (MIT): Community gratis (2.500 requests/día, 25 WebSockets en la nube), Business 50 USD/mes (10.000 requests/día), Enterprise a medida; extras "TikTok LIVE Signature Access" y "TikTok CAPTCHA Routes". [F][T]
- Documenta que las firmas son `X-Bogus`, `X-Gnarly` y `msToken`, que codifican el navegador y deben coincidir con `browser_version`/`User-Agent`, y permite montar un servidor de firma propio ("What we can't do is tell you how to generate signatures"). https://www.eulerstream.com/docs/sign-server/custom-sign-servers [F]
- **Captchas:** vende resolución de CAPTCHA de TikTok como complemento plano de 25 USD/mes ("CAPTCHA solving is a flat $25/mo add-on") con extensión de navegador. https://www.eulerstream.com/docs/captchas/captcha-extension [F]. Esto está **fuera de las reglas** del encargo (evasión de antiabuso).
- `TikTool` (https://tik.tools), `Toktik` (https://toktikhq.com) y `TikFinity` aparecen como alternativas comerciales. Algunas cifras de la competencia son de marketing y no las verifiqué. [T]
- Mantenimiento del backend: caídas documentadas en los issues de TikTokLive (#336 2025-09, #376/#379 2026-09-10).

#### 3.6.6 Proyectos que documentan la firma de requests (solo catalogados)

No usé ni evalúo su código. Existen repos públicos con reimplementaciones de `X-Bogus`, `X-Gnarly`, `a_bogus`, `msToken`, `X-Argus`, `X-Ladon`, `X-Gorgon` (por ejemplo `justscrapeme/tiktok-web-reverse-engineering`, `carcabot/tiktok-xgnarly-decoded`, `armxe/tiktok-api` / `int4444/tiktok-api`, `ylcangel/douyin_sign`). Algunos incluyen datos de CAPTCHA ("Edata") y fingerprint de login, y varios anuncian servicios comerciales de firma. Datos útiles para la decisión: la versión del algoritmo cambia varias veces al año (la lista de `justscrapeme` va de 5.1.0 en 2025-05 a 5.3.2 en 2026-09-13), lo que implica una carrera de mantenimiento permanente. [T] Mi recomendación: no incorporar nada de esto (ver sección 10).

#### 3.6.7 Socios oficiales: Streamlabs, SE.Live, Restream

- Streamlabs Desktop: login con TikTok, "Apply for TikTok Live Permission"; sin necesidad de stream key; si no hay permiso nativo puede ser sólo RTMP con clave. https://support.streamlabs.com/hc/en-us/articles/24730653249947-How-to-Go-Live-on-TikTok-from-a-Desktop [F]
- SE.Live (plugin OBS de StreamElements): inicio de sesión nativo en TikTok, "logging into your TikTok account here fast-tracks your review process with TikTok" y reconoce que TikTok no ofrece las APIs necesarias para el feed de actividad/chat de TikTok. https://support.streamelements.com/hc/en-us/articles/18771457842066-SE-Live-The-Complete-Guide-Setup-Features-Multistreaming-Troubleshooting [F]
- Restream: "auto-apply" y revisión acelerada. https://restream.io/integrations/tiktok/ [F]
- Observación [I]: todos esos socios delegan en TikTok la decisión de acceso y ninguno ofrece chat/regalos de TikTok vía API oficial, lo que apoya la tesis de que no existe una API pública para eventos.

## 4. El protocolo webcast, en síntesis

**Eventos (lectura):**
1. Resolver room id: `GET https://www.tiktok.com/api-live/user/room/?uniqueId=...&sourceType=54` (o scraping de `/@user/live`). [V] Python `fetch_room_id_api.py`, PirateTok `api.rs`.
2. Obtener cookie `ttwid` (anónima) o cookies de sesión. [V]
3. Primera petición `webcast/im/fetch` (protobuf `WebcastResponse` / `ProtoMessageFetchResult`) que devuelve `cursor`, `internal_ext`, `push_server`, `route_params` y `heartbeat_duration`. En Python/Node esta petición pasa por el firmador. [V]
4. WebSocket `wss://<push_server>/...?<params>&cursor=...`; al conectar, enviar `im_enter_room`; latido protobuf cada ~10 s; para cada respuesta con `need_ack`, enviar un `ack` con `internal_ext`. Los frames son `WebcastPushFrame` con `headers.compress_type=gzip` opcional. [V] `ws_client.py`, `connection.rs`.
5. Mensajes tipados dentro de `WebcastResponse.messages[]` (`WebcastChatMessage`, `WebcastGiftMessage`, `WebcastLikeMessage`, `WebcastMemberMessage`, `WebcastSocialMessage`, `WebcastRoomUserSeqMessage`, `WebcastControlMessage`, batallas, etc.); los regalos con racha traen `repeat_count` y `repeat_end`. [V] README Python y `proto_events.py`.
6. Cabecera de handshake `Handshake-Msg` explica rechazos (por ejemplo `DEVICE_BLOCKED`). [V] PirateTok.

**Firma (por qué existe):** parámetros `X-Bogus`, `X-Gnarly`, `msToken` en la URL generados por el JS ofuscado del sitio y atados al user-agent (Euler docs [F]). La mayor parte de rutas webcast de lectura requiere firma; el WebSocket anónimo parece admitirse sin firma según PirateTok [T/I], no verificado.

**Publicación (LIVE Studio, no oficial):** `POST /webcast/room/create/` (form con `title`, `hashtag_id`, `game_tag_id`, `live_studio=1`, etc.) responde `data.stream_url.rtmp_push_url`, `stream_id`, `id_str` y `share_url`; `room/continue` permite retomar; `room/ping/anchor` mantiene el estado; el vídeo lleva SEI. [V] código de Loukious. Para las claves obtenidas por el flujo de Streamlabs, `POST .../stream/start` devuelve `rtmp` y `key`. [V]

## 5. Flujo del stream key y requisitos de elegibilidad

Vías para conseguir un RTMP push:

| Vía | Cómo | Quién decide la elegibilidad | Fuente |
|---|---|---|---|
| Web LIVE Center | `livecenter.tiktok.com/producer`, "Go LIVE", "Save & Go LIVE", copiar Server URL y Stream Key | TikTok (cuenta) | [T] guías de Restream, StreamYard, TikFinity |
| App móvil "Cast to PC" | + > LIVE > Cast to PC | TikTok | [T] LiveU, OneStream |
| LIVE Studio (Windows) | app oficial, sin clave | TikTok (acceso LIVE Studio) | [F] |
| Socio (Streamlabs, SE.Live, Restream) | OAuth del socio | TikTok vía la whitelist del socio | [F] |
| Generador Loukious con Streamlabs | token Streamlabs | TikTok vía Streamlabs | [V] |
| Generador Loukious de LIVE Studio | suplanta LIVE Studio | TikTok (acceso LIVE Studio) + atestación | [V] |
| Agencias de "stream keys" | comercial, vía agencia | agencia/TikTok | [T] sin verificar; ver nota |

Nota: varias guías de terceros dicen que se puede obtener clave a través de agencias; TikTok no documenta ese camino en las páginas que leí [I].

Requisitos oficiales leídos:
- "Be at least 18 years old to go LIVE" y "Have 1,000 followers to go LIVE (may vary across regions)". https://support.tiktok.com/en/live-gifts-wallet/tiktok-live/what-is-tiktok-live [F]
- LIVE Studio: para ser "gaming creator" hacen falta 1.000 seguidores; creadores no gaming necesitan 10.000 (artículo del 2024-09-03). https://www.tiktok.com/live/creators/en-US/article/tiktok-live-studio-access_en-US [F]
- Ayuda de LIVE Studio (actualizada 2025-02-11): el acceso instantáneo exige cuenta creada hace cierto número de días, haber hecho LIVE recientemente, cierto número de seguidores y sin suspensión; "The requirements vary by country or region and are subject to change without notice." Streamers de otras plataformas pueden solicitar acceso con formulario. https://www.tiktok.com/live/studio/help/article/Before-you-go-LIVE/Apply-for-LIVE-access [F]
- Prueba: hay que hacer LIVE al menos 25 minutos, dos veces en la primera semana, o se revoca. https://www.tiktok.com/live/studio/help/article/FAQ/FAQ [F]
- Verificación de edad: ID y selfies, selfie con ID (Incode), tarjeta o estimación facial. https://support.tiktok.com/en/safety-hc/account-and-user-safety/age-requirements-for-tiktok-live [F]
- Ajustes por funciones: el generador de Loukious consulta `webcast/anchor_tool/threshold/check/` y declara que el dual layout se bloquea para cuentas nuevas hasta acumular sesiones de 25 minutos (`TiktokStreamKeyGenerator.py:1814-1877`); y consulta `ban_status`, `advanced_live_ban_status`, `block_status`, `golive_locale_restricted` en `room/create_info` para decidir `can_go_live`. [V]
- TikTok dice en su soporte de moderación que para hacer LIVE en navegador "you must download LIVE Studio (Windows) or OBS Studio (Linux, macOS, or Windows)". https://support.tiktok.com/en/live-gifts-wallet/tiktok-live/moderating-on-tiktok-live [F] Es decir, OBS aparece como destino aceptado por la propia TikTok.
- Contradicción a resolver con captura: Loukious sostiene que una clave real usada directamente en OBS se cierra "immediately" y lleva a restricción; las guías públicas de LIVE Center y las de socios describen OBS con clave de LIVE Center como flujo normal. La diferencia probable es que la clave de LIVE Center no es la misma que la de `room/create` de LIVE Studio [I]. Ver hueco 2.

## 6. Matriz resumida de mantenimiento

Ver sección 9 (tabla comparativa).

## 7. Riesgos de ban y legales (autores, comunidad y términos de TikTok)

**Evidencia de la comunidad y autores:**
- Baneo/shadowban/restricción por el generador de LIVE Studio: issues #23, #32, #37, #42, #44, #59, #66 (Loukious) [T]; README del propio autor sobre la restricción de integridad [V].
- Revocaciones de acceso LIVE vía socios: página de soporte de Streamlabs y issues #114, #122 [F][T].
- Cambio de política de TikTok desde 2025-07-07 (50% de juegos para Streamlabs): TikTok "is making the change to address Trust and Safety concerns with regard to content streamed on their platform through third-party tools". https://streamlabs.com/platforms/tiktok [F]
- Euler asume que el usuario es responsable de los datos que extrae y que opera un negocio de una sola persona "barely profitable" que no puede asumir consecuencias legales. https://www.eulerstream.com/docs/signatures [F]
- El repo del esquema protobuf apela a "interoperability" y a Google v. Oracle y declara que no pretende "bypass technical protection measures". [V][T]

**Términos de TikTok aplicables (citas cortas):**
- Terms of Service: prohíben "reverse engineer, disassemble, or decompile the Platform or any of its components, including its algorithms, code, or infrastructure" y "scrape, crawl, export or otherwise extract any data ... using any automated system or software, including automated bots, except as approved in writing". https://www.tiktok.com/legal/page/us/terms-of-service/en [F]
- Community Guidelines, Integrity and Authenticity: "We strictly prohibit automation tools, scripts, or other tricks designed to bypass our systems." https://www.tiktok.com/community-guidelines/en/integrity-authenticity [F]
- Community Guidelines, Privacy and Security: no se permite "Using unauthorized ways to access TikTok or creating fake versions of the platform" ni "Trying to reverse-engineer TikTok's code, systems, or algorithms--or create your own versions based on them". https://www.tiktok.com/community-guidelines/en/privacy-security [F]
- Community Guidelines, LIVE: "Repeated or serious violations, including attempts to bypass restrictions, may result in an account ban." y el creador es responsable de lo que ocurra con herramientas de terceros. https://www.tiktok.com/community-guidelines/en/accounts-features [F]
- Blog de privacidad de TikTok sobre scraping: usan CAPTCHAs, monitorización de dispositivos y red, y límites de tasa. https://www.tiktok.com/privacy/blog/how-we-combat-scraping/en [F]
- Developer Terms (si algún día se consigue integración oficial): prohíben eludir medidas de acceso y exigen respetar límites. https://www.tiktok.com/legal/page/global/tik-tok-developer-terms-of-service/en [F]

**Mi lectura [I]:** (a) un cliente que sólo lee eventos de un directo propio con sesión propia es el caso de menor riesgo para la cuenta, aunque técnicamente infringe la cláusula de automatización; (b) la suplantación de LIVE Studio con firmas y atestación falsificadas es el caso de mayor riesgo y está fuera de las reglas del encargo; (c) la vía de socio oficial o clave manual de LIVE Center no infringe nada de lo anterior.

## 8. Lecciones para una implementación en Rust

1. **Arquitectura por capas con contratos estrechos:** `capture/encode` (independiente de TikTok), `ingest` (RTMP/FLV push con la URL y clave que se le den), `credentials` (cómo se consigue la clave), `events` (WebSocket + protobuf). Así se puede entregar valor aunque `events` o `credentials` fallen.
2. **Credencial de ingest:** diseñar primero la entrada manual (campos "Server URL" y "Stream Key" + validación) y un modo "pegar desde LIVE Center". Después, si TikTok abre una vía oficial para tu app, enchufarla detrás de un trait.
3. **RTMP push en Rust:** FLV muxing con H.264 (y HEVC si se admite), AAC; los repos muestran que la ingest de TikTok acepta H.264/HEVC en FLV y que puede aceptar SEI (el proxy de Loukious hace reescritura de tags FLV). Límites de bitrate/resolución: no verificados (ver hueco 4). Para Windows hay que lanzar procesos hijos (FFmpeg) con `CREATE_NO_WINDOW` si se usa uno.
4. **Cliente de eventos:** copiar el diseño de PirateTok, no su evasión: reconexión con backoff exponencial limitado (2^n s, tope 30 s), contador reiniciado tras 30 s de sesión sana, detector de conexión muerta (`stale_timeout` 60 s), heartbeat configurable, ack según `need_ack`, gzip opcional, no usar pings WebSocket (TikTok no responde pong), tratar 200 de handshake como bloqueo definitivo sin bucle de reintentos.
5. **Protobuf resiliente:** el esquema cambia sin aviso (TikTokLive #357, Node #322, #286). Decodificar con `prost` conservando campos desconocidos y bytes crudos (o guardar el frame original) y exponer `UnknownEvent`; versionar el esquema; tests con capturas grabadas y reproducidas (como `record_capture` / `replay_capture`).
6. **Firma de lectura como componente enchufable y opcional:** definir `trait Signer`; implementar "sin firma" y "proveedor externo configurable por el usuario" (con URL base cambiable, tal y como permiten las librerías), nunca incrustar secretos de terceros ni claves en el binario. Reflexión sobre el riesgo: la dependencia de Euler ha tenido 3 incidentes públicos en 13 meses (#336, #376, #379).
7. **Higiene de credenciales:** nunca enviar `sessionid` a un tercero por defecto (Python avisa del riesgo; Node exige lista blanca del host). Guardar cookies/tokens con DPAPI o el Almacén de credenciales de Windows, no en JSON plano (el generador de Loukious deja un JSON de cookies en disco).
8. **Parámetros volátiles fuera del binario:** `version_code`, user-agent, `aid`, hosts webcast, etc. como configuración actualizable (C# #108 y los changelogs muestran que cambios mínimos rompen el handshake).
9. **Cuidado con los sondeos:** el último commit de Loukious (`373407f`) arregla un poller que agotaba un límite horario; usar cachés y sondeo sólo con la ventana visible.
10. **Sincronización de hora:** el generador usa la hora del servidor (cabeceras de respuesta) para estampar peticiones; si alguna API oficial exigiera marcas de tiempo, mantener un offset.
11. **Distribución en Windows:** el falso positivo antivirus (#30) obliga a firmar el ejecutable (Authenticode) y evitar empaquetadores sospechosos.
12. **Licencias:** no copiar código de repos AGPL modificada, GPL ni sin licencia. Para referencias de arquitectura usar PirateTok (0BSD) y documentación propia; construir el esquema protobuf mínimo a partir de capturas propias en vez de depender de un repo cuya licencia se contradice (AGPL con "non-commercial").
13. **Límite ético/técnico explícito:** no implementar resolución de CAPTCHA, rotación de huellas/UA/IP para esquivar bloqueos, registro de dispositivos falsos ni atestaciones SEI falsificadas.

## 9. Tabla comparativa

| Repo | Lenguaje | Resuelve | Auth | Firma | Último commit | Commits 12 m | Releases 12 m | Estado | Licencia |
|---|---|---|---|---|---|---|---|---|---|
| Loukious/TikTokStreamKeyGenerator | Python | Crear sala LIVE Studio y obtener RTMP, proxy con SEI | Login QR/OIDC o cookies; device falso; clave RapidAPI | Firmador RapidAPI de pago, cerrado (x-argus, x-ladon, SEI) | 2026-09-07 | 41 | 9 tags | Activo, 1 autor | Sin licencia |
| Loukious/StreamLabsTikTokStreamKeyGenerator | Python | Clave RTMP vía API privada de Streamlabs | Token Streamlabs (PKCE o leído del LevelDB local) | Ninguna en cliente | 2026-04-01 | 13 | 1 (v2.0.9) | Semiactivo | GPL-3.0 |
| isaackogan/TikTokLive | Python | Eventos del live | Anónima; sesión opcional | Euler Stream (URL WS y firmas) | 2026-09-09 | 99 | 5 (7.0.0a1 a 7.0.1) | Activo | AGPL modificada (antes MIT) |
| zerodytrash/TikTok-Live-Connector | TypeScript | Eventos del live | Anónima; cookie u OAuth Euler | Euler Stream | 2026-09-22 | 112 | 9 tags | Activo | AGPL modificada (antes MIT) |
| frankvHoof93/TikTokLiveSharp | C# | Eventos del live (Unity) | Anónima | Euler Stream (`/webcast/fetch`) | 2025-01-28 | 0 | 0 | Abandonado | MIT |
| jwdeveloper/TikTokLiveRust | Rust | Eventos del live | Anónima | Euler (`sign_url`) | 2024-08-31 | 0 | 0 | Abandonado (archivado 2025-07-14) | MIT / Apache-2.0 |
| PirateTok/live-rs | Rust | Eventos del live | `ttwid` anónimo; cookies opcionales | Ninguna (afirmado) | 2026-10-04 | 32 | 5 | Activo, joven | 0BSD |
| steampoweredtaco/gotiktoklive | Go | Eventos y descarga | Anónima | Servidor de firma configurable | 2025-09-14 | 0 | 0 | Semiabandonado | MIT |
| isaackogan/TikTok-Webcast-Protobuf | proto | Esquema webcast | n/a | n/a | 2026-09-28 | 107 (clon) | tags TS | Activo | AGPL modificada ("non-commercial" en README) |

Recuento de "Releases 12 m" calculado por tags de git desde 2025-10-16 donde existen; para TikTokLive cuento los hitos citados en commits.

## 10. Matriz "viable vs. no viable"

Criterio: reglas fijas del encargo (no evadir captchas ni antiabuso, no falsificar elegibilidad, no usar cuentas o credenciales de terceros).

### (a) Obtención de stream key

| Opción | Veredicto | Motivo | Confianza |
|---|---|---|---|
| Usuario copia Server URL y Stream Key desde LIVE Center o "Cast to PC" y los pega en la app | **Viable** | Flujo documentado por TikTok y socios; la cuenta debe ser elegible (18+, 1.000 seguidores, región) | [F][T] |
| Salto a TikTok LIVE Studio oficial como proceso externo | **Viable** (fuera de alcance de "app propia") | App oficial; no resuelve el requisito de una app propia | [F] |
| Integración oficial como socio de TikTok (OAuth) | **Desconocido, potencialmente viable** | No encontré programa público; requiere acuerdo con TikTok | [I] |
| Token de Streamlabs + API privada `streamlabs.com/api/v5/slobs/tiktok` | **Viable con reservas (no recomendado como base)** | Técnicamente simple [V]; depende de un backend privado de un socio, suplanta el user-agent de Streamlabs Desktop, lee el token de otra app y está sujeto a revocaciones | [V][F] |
| Generador estilo LIVE Studio (`webcast/room/create` con `aid=8311`, device registrado falso, `x-argus`/`x-ladon` del firmador, SEI `live_studio` firmado) | **No viable** | Suplanta al cliente oficial, fabrica identidad de dispositivo y falsifica atestación de autenticidad; el propio README admite restricción por "integrity/authenticity" sin ella | [V] código; [I] valoración |
| "Spoofing" para cuentas sin acceso a LIVE Studio (versiones antiguas de Loukious) | **No viable** | Falsifica elegibilidad (README antiguo: "use spoofing if you don't have access") | [V] historial |
| Comprar claves o acceso vía agencias | **No evaluado; no recomendable** | No hay base oficial verificada | [T] |

### (b) Eventos del live

| Opción | Veredicto | Motivo | Confianza |
|---|---|---|---|
| Cliente WebSocket anónimo con firma por proveedor externo configurable (Euler u otro) | **Viable con riesgos** | Funciona hoy según los repos activos; dependencia de tercero, cuotas (429), incidentes, y cláusula ToS contra automatización | [V][T] |
| Cliente WebSocket directo con `ttwid` sin firma (modelo PirateTok) | **Probablemente viable, sin verificar** | Código leído [V]; funcionamiento real no probado [I]; fragilidad alta | [V]/[I] |
| Eventos de **tu propio** directo con tu propia sesión (`sessionid` local, sin enviarla a terceros) | **Viable con riesgos** | Menor riesgo de cuenta; no enviar la cookie a terceros | [I] |
| Enviar `sessionid` a un servicio de firma de terceros | **No recomendable** | Riesgo de credencial; las propias librerías lo marcan como "risky operation" | [V] |
| Enviar chat, likes o regalos por la API de Euler (rutas premium) | **No viable para este proyecto** | Credencial de usuario a tercero, automatización de acciones, funciones archivadas en Python (`63e897f`) | [V][I] |
| API oficial de TikTok para chat/regalos | **No existe públicamente** | Euler y StreamElements lo afirman; posible integración OAuth por creador (blog de terceros) sin verificar | [T] |

### (c) Captcha o elegibilidad especial

| Elemento | Veredicto | Motivo |
|---|---|---|
| Resolver CAPTCHAs de TikTok (rutas CAPTCHA de Euler, extensión) | **No viable** | Evasión directa de antiabuso |
| Rotar user-agent/ttwid/IP/proxy para saltar `DEVICE_BLOCKED` o bloqueos | **No viable** | Evasión de bloqueo de dispositivo/red |
| Registrar dispositivos falsos (`device_register` con MAC y serie aleatorios) | **No viable** | Fabricación de identidad de dispositivo |
| Fabricar firmas SEI o `x-argus`/`x-ladon` para aparentar ser LIVE Studio | **No viable** | Falsifica atestación de cliente |
| Acceso a LIVE o LIVE Studio sin cumplir requisitos | **No viable** | Falsificar elegibilidad |
| Dual layout y funciones con umbral (`anchor_tool/threshold/check`) | **Solo si la cuenta lo habilita realmente** | Elegibilidad real de la cuenta |
| Verificación de edad (ID/selfie) | **Solo la hace el titular** | Proceso oficial de TikTok |

## 11. Huecos de información que solo se resuelven capturando tráfico real en el PC del usuario

Hacerlo con la propia cuenta del usuario, su LIVE Studio oficial instalado, y sin intentar reproducir firmas.

1. **Secuencia exacta de LIVE Studio al arrancar un directo** en la versión actual (posterior a 1.27.0): qué endpoints y en qué orden, qué cabeceras de firma lleva cada uno, y cuáles son realmente obligatorias.
2. **Qué pasa con una clave obtenida desde LIVE Center (web) usada con un codificador propio u OBS**, sin SEI: ¿se acepta?, ¿hay restricción de visibilidad? Esto contradice lo que afirma Loukious y decide si basta con "pegar la clave".
3. **Qué atestación en banda exige TikTok** (si algo más allá del SEI JSON `ttls_live_scene`) y para qué rutas de ingest.
4. **Parámetros de ingest:** formato exacto de Server URL y clave (rtmp o rtmps, puerto, vigencia de la clave, regeneración en cada directo, #139), codecs admitidos, bitrate/resolución máximos, intervalo de keyframe, audio.
5. **Si el WebSocket anónimo con solo `ttwid` funciona hoy** desde la IP y región del usuario, y con qué tasa de bloqueo (`DEVICE_BLOCKED`, 200 en handshake); si `/webcast/im/fetch` exige firma en 2026-10.
6. **Qué eventos requieren sesión del anfitrión**: `ENVELOPE` (Node #328), `GIFT` para ciertos usuarios (Node #316), nivel de gifters 25+ en `member` (Node #320), lista de espectadores, y si el propietario de la sala ve más datos.
7. **Binding por dispositivo:** `device_id`/`install_id`, "continuar solo desde el mismo dispositivo" (Loukious #75), y relación con `tt-target-idc` y el despacho TNC.
8. **Respuesta real de `room/create_info`** de la cuenta del usuario (campos `ban_status`, `block_status`, `golive_locale_restricted`, umbral) para saber con precisión qué elegibilidad tiene en su país.
9. **Existencia de un programa oficial de partners/API LIVE** y si una app independiente puede solicitarlo (revisar developers.tiktok.com y contactar con TikTok); un blog de terceros menciona una "TikTok Open API with Live Studio integration" por creador que no pude confirmar.
10. **Qué hace LIVE Studio para el heartbeat del anfitrión** (`room/ping/anchor`) y para finalizar (`anchor_pre_finish`): frecuencia, qué ocurre si falla, cierre automático (Loukious #45 reporta cortes a los 10 a 15 minutos con "mobile camera stream").
11. **Comportamiento por región:** `priority_region` (Loukious #76), IDC (`tt-target-idc`), países con LIVE bloqueado (Loukious #72, Bielorrusia).
12. **Si LIVE Studio fija el certificado (certificate pinning)** y cómo capturarlo legítimamente; el comentario del código de Loukious indica capturas con Charles (`TiktokStreamKeyGenerator.py:1798-1801`), lo que sugiere que es posible [I].

## 12. Fuentes

Repos analizados en local (lectura estática, sin ejecutar):
- https://github.com/Loukious/TikTokStreamKeyGenerator (HEAD `373407f`)
- https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator (HEAD `ef03fb2`)
- https://github.com/Loukious/obs-multi-rtmp (HEAD `4518eca`)
- https://github.com/isaackogan/TikTokLive (HEAD `fc73b8f`)
- https://github.com/zerodytrash/TikTok-Live-Connector (HEAD `8a92330`)
- https://github.com/frankvHoof93/TikTokLiveSharp (HEAD `edbcb27`)
- https://github.com/jwdeveloper/TikTokLiveRust (HEAD `5b7950c`)
- https://github.com/steampoweredtaco/gotiktoklive (HEAD `3be247c`)
- https://github.com/isaackogan/TikTok-Webcast-Protobuf (HEAD `8e7ccad`)
- https://github.com/PirateTok/live-rs (HEAD `ee60855`)

Otros proyectos mencionados:
- https://github.com/fskhri/TiktokGenerateRTMP
- https://github.com/antiantidev/labsgen-tiktok
- https://github.com/EulerStream/TikTok-Live-Api
- https://github.com/justscrapeme/tiktok-web-reverse-engineering
- https://github.com/carcabot/tiktok-xgnarly-decoded
- https://github.com/armxe/tiktok-api
- https://github.com/ylcangel/a_bogus
- https://tik.tools/docs
- https://toktikhq.com/for/developers

Issues citados:
- https://github.com/Loukious/TikTokStreamKeyGenerator/issues/23, /36, /59, /66, /68, /72, /75, /76
- https://github.com/Loukious/StreamLabsTikTokStreamKeyGenerator/issues/114, /122, /134, /139
- https://github.com/isaackogan/TikTokLive/issues/336, /342, /349, /357, /376, /379
- https://github.com/zerodytrash/TikTok-Live-Connector/issues/310, /323, /327, /328, /332
- https://github.com/frankvHoof93/TikTokLiveSharp/issues/95, /108, /118
- https://github.com/PirateTok/live-rs/issues/1

Documentación oficial y de socios:
- https://support.tiktok.com/en/live-gifts-wallet/tiktok-live/what-is-tiktok-live
- https://support.tiktok.com/en/live-gifts-wallet/tiktok-live/moderating-on-tiktok-live
- https://support.tiktok.com/en/safety-hc/account-and-user-safety/age-requirements-for-tiktok-live
- https://www.tiktok.com/live/studio/help/article/Before-you-go-LIVE/Apply-for-LIVE-access
- https://www.tiktok.com/live/studio/help/article/FAQ/FAQ
- https://www.tiktok.com/live/creators/en-US/article/tiktok-live-studio-access_en-US
- https://www.tiktok.com/legal/page/us/terms-of-service/en
- https://www.tiktok.com/community-guidelines/en/integrity-authenticity
- https://www.tiktok.com/community-guidelines/en/privacy-security
- https://www.tiktok.com/community-guidelines/en/accounts-features
- https://www.tiktok.com/privacy/blog/how-we-combat-scraping/en
- https://www.tiktok.com/legal/page/global/tik-tok-developer-terms-of-service/en
- https://developers.tiktok.com/docs/en/our-guidelines-developer-guidelines
- https://streamlabs.com/platforms/tiktok
- https://support.streamlabs.com/hc/en-us/articles/24730653249947-How-to-Go-Live-on-TikTok-from-a-Desktop
- https://support.streamlabs.com/hc/en-us/articles/35259691237147-Streamlabs-x-TikTok-Streaming-Access-Revoked-Suspended
- https://ir.logitech.com/news/news-details/2024/Logitechs-Streamlabs-Launches-TikTok-LIVE-Integration-to-Power-Seamless-Streaming/default.aspx
- https://support.streamelements.com/hc/en-us/articles/18771457842066-SE-Live-The-Complete-Guide-Setup-Features-Multistreaming-Troubleshooting
- https://restream.io/integrations/tiktok/
- https://www.eulerstream.com/docs/signatures
- https://www.eulerstream.com/docs/api/rate-limits
- https://www.eulerstream.com/docs/sign-server/custom-sign-servers
- https://www.eulerstream.com/docs/captchas/captcha-extension
- https://www.eulerstream.com/pricing

Guías de terceros sobre stream keys (usadas solo como [T]):
- https://blog.tikfinity.com/?p=7010
- https://restream.io/learn/platforms/how-to-find-tiktok-stream-key/
- https://support.streamyard.com/hc/en-us/articles/360051974452-Create-a-Live-Stream-on-TikTok
- https://studiosupport.liveu.tv/hc/en-us/articles/17094667853083-TikTok
- https://www.tikliveapi.com/blog/tiktok-live-streaming-api-landscape-2026/

Paquetes: https://crates.io/crates/piratetok-live-rs, https://docs.rs/tiktoklive/latest, https://www.npmjs.com/package/tiktok-live-proto, https://pypi.org/project/TikTokLiveProto/
