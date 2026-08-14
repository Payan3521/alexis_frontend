# Strangeroses

Sitio web estático de Strangeroses — modelos destacadas con imágenes y videos exclusivos.
Publicado en **https://strangeroses.com** (desplegado en **Vercel** desde el repo de GitHub).

---

## 1. Estructura del proyecto

```
alexis_frontend/
├── index.html          # Landing: hero, tarjetas de modelos, panel de videos + lightbox
├── modelo1.html        # Perfil Modelo 1 (Valeria)
├── modelo2.html        # Perfil Modelo 2 (Camila)
├── modelo3.html        # Perfil Modelo 3 (Lua)
├── modelo4.html        # Perfil Modelo 4 (Ashly)
├── modelo5.html        # Perfil Modelo 5 (Próximamente)
├── img/                # Imágenes del sitio
│   ├── modelo1.webp    # Fotos de perfil optimizadas (WebP)
│   ├── modelo2.webp
│   ├── modelo3.webp
│   ├── modelo4.webp
│   ├── *.png           # Logos, fondo del hero, píldora de enlace (con transparencia)
├── videos/
│   ├── valeria/        # 3 videos (IMG_2007, IMG_2051, IMG_2052)
│   ├── camila/         # 2 videos (IMG_6040, IMG_2067)
│   ├── lua/            # 2 videos (IMG_2085, IMG_2086)
│   ├── ashly/          # 1 video (ashly.mp4)
│   └── */thumbs/       # Miniaturas / poster de cada video (JPEG ~40-110 KB)
├── favicon.* / apple-touch-icon.png
└── README.md
```

### Stack

- **Lenguaje:** HTML5 + CSS3 + JavaScript vanilla (sin frameworks, sin dependencias, sin build).
- **Imágenes:** PNG/JPEG originales y versiones optimizadas en WebP.
- **Video:** MP4 (H.264 + AAC), 1080p, `faststart`.
- **Hosting:** Vercel (CDN, sirve archivos estáticos con soporte de peticiones por rango `Range`).
- **Repo:** GitHub (`main`). Cada `push` dispara el deploy automático en Vercel.

---

## 2. Arquitectura

### Flujo de la página principal (`index.html`)

1. **Hero**: fondo con imagen + título + barra social (Instagram, OnlyFans, Pornhub, Telegram).
2. **Sección Modelos**: tarjetas enlazadas a `modelo1..5.html`. Cada tarjeta tiene:
   - Foto de perfil (`loading="lazy"`).
   - Panel de videos (`.video-panel`) con **miniaturas en imagen** (no `<video>`).
3. **Lightbox de video** (`#lightbox`): overlay a pantalla completa con un único `<video>`.

### Cómo se reproduce un video

- **Desktop:** el panel aparece al pasar el ratón (`:hover`). Se ve la miniatura estática.
- **Móvil:** el panel siempre es visible (no hay hover en táctil).
- En ambos casos, **hacer clic/tap en una miniatura abre el lightbox**, se asigna `src` al `<video>`, se carga y reproduce. Cerrar (botón ✕, clic fuera o `Esc`) pausa y **libera la fuente** para no consumir datos.

### Páginas de modelo (`modelo1..5.html`)

Perfil con foto, descripción y grid de videos con `<video controls>` + atributo `poster` + `preload="none"` (no descargan video al entrar; lo cargan solo al pulsar play).

---

## 3. El problema de rendimiento (y cómo se solucionó)

### El problema original

La página era **muy lenta** y en móvil **no cargaba**. Las causas:

1. **Videos RAW de iPhone (`.MOV` / codec HEVC H.265)**: 37 a 97 MB cada uno, **336 MB en total**, subidos tal cual.
2. **`preload="metadata"` en cada `<video>` del index**: los `.MOV` de iPhone guardan la metadata (atom `moov`) **al final del archivo**, así que el navegador debía descargar el archivo **entero** (¡66-97 MB!) solo para mostrar una miniatura. Al abrir `index.html` se disparaban descargas de cientos de MB.
3. **Códec HEVC no compatible** en Chrome/desktop y muchos Android → videos en negro.
4. **Bitrate excesivo**: tras una primera conversión, los videos quedaron a 10-25 Mbps → imposibles de transmitir en móvil (spinner de "cargando" eterno).
5. **Imágenes pesadas**: PNG de hasta 913 KB.
6. **Archivo casi ilegal para GitHub**: `IMG_2067.MOV` pesaba 97 MB (GitHub bloquea >100 MB y advierte >50 MB).

### Solución aplicada (3 fases)

#### Fase 1 — Transcodificar los videos (la clave)

Convertir todos los `.MOV` (HEVC) a **MP4 H.264 1080p** con `-movflags +faststart`:

```bash
ffmpeg -i entrada.MOV \
  -vf "scale=-2:'min(1920,ih)'" \
  -c:v libx264 -profile:v high -level 4.0 -b:v 3500k -maxrate 4500k -bufsize 9000k \
  -preset medium -g 100 -pix_fmt yuv420p \
  -c:a aac -b:a 128k -movflags +faststart \
  salida.mp4
```

Por qué funciona:
- **H.264 + `yuv420p`**: compatible con iOS, Android, Chrome, Safari, Firefox.
- **`faststart`**: coloca el `moov` al inicio → el navegador obtiene metadata con una descarga mínima (verificado: `moov` en el byte 36).
- **Bitrate controlado (3.5 Mbps, máx. 4.5)**: transmisión fluida en 4G, arranca en 1-2 s. Las peticiones `Range` (que el navegador usa para reproducir) ahora se satisfacen al instante.
- **Keyframes cada 2 s (`-g 100`)**: mejor búsqueda y arranque.

**Resultado:** de **336 MB → 56 MB** en videos.

| Video | Antes | Después | Bitrate |
|---|---|---|---|
| IMG_2007 | 66 MB | 13 MB | 3.5 Mbps |
| IMG_2051 | 25 MB | 4.8 MB | 3.5 Mbps |
| IMG_2052 | 40 MB | 8.3 MB | 3.5 Mbps |
| IMG_6040 | 37 MB | 7.2 MB | 3.3 Mbps |
| IMG_2067 | 97 MB | 5.8 MB | 3.5 Mbps |
| IMG_2085 | 50 MB | 5.7 MB | 3.2 Mbps |
| IMG_2086 | 16 MB | 2.9 MB | 3.3 Mbps |
| ashly | 5 MB | 4.6 MB | 1.5 Mbps |

> Los **originales** `.MOV` quedaron respaldados fuera del repo (p. ej. `/tmp/opencode/originales/videos/`). No se suben al repo para no inflarlo.

#### Fase 2 — Lazy-loading (código)

- **`index.html`**: las miniaturas del panel ya **no son `<video>`**, son `<img>` (~40 KB). El `<video>` real se crea **solo al hacer clic** en el lightbox. Quitar `controls`, `preload` y `poster` pesados del markup de la landing.
- **`modeloX.html`**: `preload="none"` + `poster` → entrar al perfil no descarga video; se descarga solo al pulsar play.
- **Lightbox robusto**: `preload="auto"`, `play()` sobre el evento `canplay`, y liberar el `src` al cerrar (no consume datos mientras no se usa).

#### Fase 3 — Miniaturas y WebP

- **Miniaturas/poster** extraídas de cada video:

```bash
ffmpegthumbnailer -i video.mp4 -o thumbs/NOMBRE.jpg -s 640 -q 9 -t 5
```

- **Imágenes** convertidas a WebP (Python/Pillow, ancho máx. 640 px, calidad 82):

| Imagen | Antes | Después |
|---|---|---|
| modelo1 | 913 KB (PNG) | 39 KB (WebP) |
| modelo2 | 758 KB (PNG) | 36 KB (WebP) |
| modelo3 | 372 KB (JPEG) | 27 KB (WebP) |
| modelo4 | 91 KB (JPEG) | 42 KB (WebP) |

### Resultado final

- **Antes:** la landing intentaba descargar ~300 MB al abrir → no cargaba en móvil.
- **Ahora:** la landing carga ~150 KB (HTML + miniaturas) al instante; el video se descarga solo cuando el usuario lo abre, y transmite sin cortes.

---

## 4. Cómo ejecutar en local

Como es HTML estático, basta con servir la carpeta (un `file://` directo no sirve el video con peticiones `Range` en todos los navegadores):

```bash
python3 -m http.server 8000
# abre http://localhost:8000
```

> Nota: el `http.server` de Python no responde `Range`; para probar streaming real usa Vercel o un servidor con soporte de rangos.

---

## 5. Despliegue en Vercel

1. El repo está en GitHub: `https://github.com/Payan3521/alexis_frontend`.
2. En Vercel: **New Project → Import** desde GitHub → ajustes por defecto (Framework: *Other / Static*, Build: none, Output: repo root).
3. Cada `push` a `main` redespliega automáticamente.

```bash
git add -A
git commit -m "mensaje"
git push origin main
```

Vercel sirve los `.mp4` con `Content-Type: video/mp4` y soporta peticiones por rango (`Accept-Ranges: bytes`, respuestas `206`), requisito para el streaming HTML5. **Verificado en producción.**

---

## 6. Mantenimiento: agregar un video nuevo

1. Convierte el archivo (si viene de iPhone `.MOV`):

```bash
ffmpeg -i NUEVO.MOV \
  -vf "scale=-2:'min(1920,ih)'" \
  -c:v libx264 -profile:v high -level 4.0 -b:v 3500k -maxrate 4500k -bufsize 9000k \
  -preset medium -g 100 -pix_fmt yuv420p \
  -c:a aac -b:a 128k -movflags +faststart \
  NUEVO.mp4
```

2. Genera la miniatura:

```bash
mkdir -p videos/modelo/thumbs
ffmpegthumbnailer -i NUEVO.mp4 -o videos/modelo/thumbs/NUEVO.jpg -s 640 -q 9 -t 5
```

3. Referéncialo en `index.html` (con `data-src`) y/o en `modeloX.html` (con `preload="none"` + `poster`).
4. Commit + push.

---

## 7. Notas técnicas

- **Por qué funciona el streaming:** el navegador pide el archivo en trozos (`Range: bytes=...`) y el host responde `206`. El `faststart` garantiza que la primera petición trae ya el índice (`moov`) y el primer fotograma clave, así el video arranca sin esperar la descarga completa.
- **Bitrate objetivo:** 3.5 Mbps es el punto óptimo para 1080p/50fps en móvil. Bajarlo a 720p reduce a la mitad el peso si se necesita más velocidad.
- **CDN:** Vercel cachea los assets en su borde (edge), por lo que los videos se entregan desde la ubicación más cercana al visitante.
- **Archivos no usados** (`videos/camila/IMG_8273.JPG.jpeg`, `videos/lua/IMG_2087.PNG`): imágenes sueltas que no se referencian; se pueden eliminar para aligerar el repo.