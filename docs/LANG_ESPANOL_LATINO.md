# Idioma Español Latino y control de tamaño/calidad

Guía de cómo se configuró el stack para que Sonarr/Radarr prefieran contenido con
**audio en español latino**, no bajen remuxes gigantes sin el idioma, y Bazarr agregue
subtítulos en español latino cuando el audio es inglés.

> **Contexto para IA u operador**: el stack completo (Jellyfin + Sonarr + Radarr +
> Prowlarr + Bazarr + qBittorrent + Jellyseerr + Homarr + Caddy + FlareSolverr) corre
> con Docker Compose en el servidor `192.168.1.34`, en la carpeta `~/media-stack`.
> Acceso: `ssh jmro@192.168.1.34`. Operar: `cd ~/media-stack && docker compose ps`,
> `docker compose logs -f sonarr`, `docker compose restart sonarr`, `docker compose up -d`.
> Las URLs: `/sonarr`, `/radarr`, `/bazarr`, `/jellyfin` (proxy Caddy en el puerto 80);
> Prowlarr `:9696`, qBittorrent `:8080`, Jellyseerr `:5055`, Portainer `:9000`.
> Ver `README.md`, `docs/INDEXERS.md` y `docs/DATA_LAYOUT.md` para el resto del stack.

## Por qué pasaba el problema

El comportamiento original era el típico de los *arr sin configurar:

1. **Radarr/Sonarr ordenan por "calidad" = mayor tamaño gana.** Un remux 4K de
   40–80 GB puntuaba más que un WEB-DL 1080p de 2–6 GB, aunque el remux casi nunca
   tuviera audio latino (vienen en inglés/DTS).
2. **Los perfiles tenían `upgradeAllowed: false`**, así que además nunca se buscaba
   una versión mejor ni se reemplazaba la existente.
3. **Bazarr no tenía idiomas ni perfiles configurados** (`serie_default_enabled`,
   `movie_default_enabled` en `false`), por eso no descargaba subtítulos en español.

## La solución en dos partes

- **Sonarr/Radarr**: Custom Formats con puntaje, que hacen que el idioma pese más
  que el tamaño al elegir release. Con `minFormatScore: 0` no se bloquea la descarga:
  si no existe versión con español, baja la mejor en inglés.
- **Bazarr**: perfil de idioma `espaniol` con **Spanish (Latino)** como primera opción,
  asignado como default para series y películas. Con `single_language: true` solo busca
  el primer idioma del perfil (= latino).

## Custom Formats creados

Los mismos en **Sonarr** y **Radarr** (`Settings → Custom Formats`):

| Nombre | Implementación | Regex (Release Title) | Score |
| --- | --- | --- | --- |
| Español Latino | `ReleaseTitleSpecification` | `\b(Latino|LAT)\b` | **+1000** |
| Español Castellano | `ReleaseTitleSpecification` | `\b(Español|Castellano|Spanish)\b` | **+500** |

### Por qué esas regex

- **`\b(Latino|LAT)\b`**: captura el tag que usan los releasers para indicar audio
  español latino. Los `\b` (word boundaries) evitan falsos positivos tipo `Latitude`
  o `Latvia`.
- **`\b(Español|Castellano|Spanish)\b`**: captura el español en general como
  alternativa de menor puntaje. Se usa `Español` y no solo `Spanish` porque muchos
  releases latinos usan `ESP`/`Español` en el título.

> La regex de Sonarr/Radarr es case-insensitive, no hace falta escribir `LATINO`.

## Perfil de calidad: `1080p Latino Preferido`

Se reemplazó el perfil que usaban las apps (Sonarr: `Any`; Radarr: `Ultra-HD`) por el
nuevo perfil **`1080p Latino Preferido`** (id 5 en Radarr, id 1 en Sonarr):

- **Resoluciones permitidas**: hasta **1080p** (SD, 720p, 1080p). **4K desactivado.**
- **Cutoff**: `WEBDL-1080p` (id de calidad 7). No se busca más allá de un WEB-DL 1080p.
- **`upgradeAllowed: true`**: si aparece una versión con mejor puntaje de idioma,
  la reemplaza.
- **`minFormatScore: 0`**: no exige un puntaje mínimo. Es deliberado — si no existe
  release con español latino/castellano, **descarga igual** en inglés (el audio se
  conserva y Bazarr pone los subs).
- **`formatItems` asignados**: `Español Latino` = 1000, `Español Castellano` = 500.
- **Cutoff**: Radarr apunta al ítem `WEBDL-1080p` (id 7); Sonarr apunta al grupo
  `WEB 1080p` (id 1002) porque ahí viven sus WEB-DL/WEBRip 1080p.

### Cómo decide ahora (orden de prioridad)

1. Mayor puntaje de Custom Formats (1000 > 500 > 0).
2. Mayor calidad (hasta 1080p).
3. Tamaño.

Resultado práctico: un WEB-DL 1080p con audio latino (score 1000, ~3 GB) **le gana**
a un remux 4K de 60 GB sin español (score 0). El "tamaño más grande" deja de mandar.

### Por qué 1080p y no 4K

En una TV LED de 50" a distancia de sillón (~2.5–3 m) el ojo no distingue 4K de 1080p.
Los remux 4K con audio latino son raros, pesan 10–30× más y saturan el disco/red.
Si más adelante cambia el display (TV grande, proyector), alcanza con subir el cutoff
del perfil y re-habilitar las calidades 2160p.

## Bazarr

- **Perfil de idioma `espaniol`** (en la DB `table_languages_profiles`), con los items
  en este orden:
  1. `ea` = **Spanish (Latino)** ← primer idioma, el que se descarga
  2. `es` = **Spanish** (castellano), como segundo por si no hay latino
- **`serie_default_enabled: true`** y **`serie_default_profile: espaniol`**
- **`movie_default_enabled: true`** y **`movie_default_profile: espaniol`**
- **`single_language: true`**: Bazarr solo descarga el primer idioma del perfil (latino),
  evita bajar subs duplicados latino+castellano.
- Providers activos: `opensubtitlescom`, `yifysubtitles` (sin cambio).

Con esto, cada serie/película nueva o existente se asocia automáticamente al perfil
`espaniol`, y Bazarr busca subtítulos en español latino. Para contenido en inglés
bajado por el perfil 1080p (score 0), los subs en español llegan por Bazarr.

### Si querés más idiomas o hi

- Quitar `single_language` baja subs de **todos** los idiomas del perfil.
- Agregar más lenguas al perfil `espaniol` (items de `table_languages_profiles`).
- Los flags `hi` / `forced` en los items controlan subtítulos para hipoacúsicos y
  forzados.

## Aplicación por API (comandos usados)

Todo se configuró por API HTTP (con `X-Api-Key`), no editando archivos a mano:

```bash
# Custom Formats (igual en Sonarr :8989 y Radarr :7878)
curl -X POST http://192.168.1.34/sonarr/api/v3/customformat \
  -H "X-Api-Key: <SONARR_KEY>" -H "Content-Type: application/json" \
  -d '{"name":"Español Latino","specifications":[{"name":"Audio Latino",
       "implementation":"ReleaseTitleSpecification","negate":false,"required":false,
       "fields":[{"name":"value","value":"\\b(Latino|LAT)\\b"}]}]}'

# Perfil de calidad: PUT /api/v3/qualityprofile/{id} con el JSON completo del perfil,
# ajustando upgradeAllowed=true, minFormatScore=0 y formatItems con scores.
# Ojo con el cutoff: Radarr WEBDL-1080p=7; Sonarr usa el grupo WEB 1080p=1002.

# Migrar películas al perfil nuevo
curl -X PUT http://192.168.1.34/radarr/api/v3/movie/editor \
  -H "X-Api-Key: <RADARR_KEY>" -H "Content-Type: application/json" \
  -d '{"movieIds":[2,3,5,6,7],"qualityProfileId":5,"monitored":true}'
```

### Bazarr por API (ojo: usa form-data, no JSON)

```bash
curl -X POST http://192.168.1.34/bazarr/api/system/settings \
  -H "X-API-KEY: <BAZARR_KEY>" \
  --data-urlencode "settings-general-serie_default_enabled=true" \
  --data-urlencode "settings-general-serie_default_profile=espaniol" \
  --data-urlencode "settings-general-movie_default_enabled=true" \
  --data-urlencode "settings-general-movie_default_profile=espaniol" \
  --data-urlencode "settings-general-single_language=true"
```

El reordenamiento del perfil (Latino primero) se hizo directo en la DB de Bazarr:

```sql
UPDATE table_languages_profiles
SET items='[{"id": 2, "language": "ea", "audio_exclude": "False",
  "audio_only_include": "False", "hi": "False", "forced": "False"},
 {"id": 1, "language": "es", "audio_exclude": "False",
  "audio_only_include": "False", "hi": "False", "forced": "False"}]'
WHERE profileId=1;
```

> Después de tocar la DB hay que reiniciar el container (`docker restart bazarr`).

## Indexers: separación series vs películas

Prowlarr es la central de indexers y sincroniza automáticamente a las apps por
**categoría**, no de a un indexer entero:

- **Sonarr** (series) recibe solo las categorías `TV`.
- **Radarr** (películas) recibe solo las categorías `Movies`.
- Un indexer con ambas categorías (ej. 1337x, The Pirate Bay) aparece en las dos
  apps; uno solo `Movies` (ej. YTS) aparece únicamente en Radarr.

Estado actual del stack:

| App | Indexers sincronizados |
| --- | --- |
| Sonarr | 1337x, The Pirate Bay |
| Radarr | 1337x, The Pirate Bay, YTS |

### Por qué no aparecen releases latinos

Los trackers públicos (1337x, TPB, YTS) casi no publican contenido con audio
**español latino**: son en inglés/ita/hindi/french. En una búsqueda real, 0 de 179
releases para *The Batman* matcheó el custom format Latino.

El contenido latino se publica mayormente en **trackers privados latinos**. Los
definiciones ya vienen incluidas en Prowlarr (aparecen al buscar "latino/hispano"):

- **Privados (requieren registro/invitación)**: `BTArg` (es-AR), `Lat-Team` (es-MX),
  `ChileBT` (es-CL), `F1Carreras` (es-MX), `HD-Olimpo`, `Torrenteros`,
  `ParabellumHD`, `NOBS` (es-ES), etc.
- **Públicos/semiprivados**: `GamesTorrents` (es-ES, solo juegos), `PuntoTorrent`
  (es-ES), `Union Fansub` (semi-privado, anime).

### Cómo agregar un tracker latino

1. En **Prowlarr** (`http://192.168.1.34:9696`): `Settings → Indexers → Add Indexer`,
   elegí el tracker (ej. `BTArg`) y cargá credenciales (passkey/cookies).
2. Prowlarr lo sincroniza solo a las apps cuyas categorías soporte el tracker
   (`Movies`, `TV` o ambas). No hace falta tocarlo en Sonarr/Radarr.
3. Los Custom Formats (Latino/Castellano) ya están en ambas apps, así que el score
   se aplica apenas aparezca un release latino.
4. Si el tracker usa Cloudflare, asigná el tag `flaresolverr` (igual que los indexers
   actuales, todos lo tienen: tags `[1]`).

> Los indexers públicos se sincronizan igual: si querés separar más, podés usar
> **tags** en Prowlarr para filtrar qué indexers llegan a qué app.

## Formatos de request que acepta cada API

Cada app tiene su propio contrato de API. Esto importa porque **no todas aceptan JSON**:

| App | Base de la API | Header auth | Formato de request | Content-Type |
| --- | --- | --- | --- | --- |
| **Sonarr** | `/sonarr/api/v3` | `X-Api-Key` | **JSON** | `application/json` |
| **Radarr** | `/radarr/api/v3` | `X-Api-Key` | **JSON** | `application/json` |
| **Prowlarr** | `:9696/api/v1` | `X-Api-Key` | **JSON** | `application/json` |
| **Bazarr** | `/bazarr/api` | `X-API-KEY` | **form-data / urlencoded** (lectura y algunos GET devuelven JSON) | `application/x-www-form-urlencoded` |

Reglas por app:

- **Sonarr / Radarr (JSON):** todos los bodies se mandan como JSON. GET no lleva body
  (los params van en la query string: `?movieId=2`). POST/PUT llevan el objeto completo
  que devolvió el GET (se edita y se devuelve entero) — mandar el JSON a medias da
  `400 Validation failed`.
- **Bazarr (form-data):** `POST /api/system/settings` usa campos tipo
  `settings-general-<clave>` como form-urlencoded. **No** acepta JSON en ese endpoint
  (devuelve 405/204 sin persistir). Para el perfil de idioma se toca la DB SQLite
  directamente (`table_languages_profiles`). Los GET de lectura sí responden JSON.
- **Prowlarr (JSON):** ej. `POST /api/v1/search` con body
  `{"query":"...","type":"search","indexerIds":[1,2,3],"categories":[{"id":2000,"name":"Movies"}]}`.
  Nota: las `categories` se mandan como objetos `{id,name}`, no como números sueltos
  (un array de ints da `400`).

### Endpoints útiles

| App | Endpoint | Uso |
| --- | --- | --- |
| Sonarr/Radarr | `GET /api/v3/qualityprofile` | listar perfiles y ver `formatItems`/`cutoff` |
| Sonarr/Radarr | `PUT /api/v3/qualityprofile/{id}` | editar un perfil (JSON completo) |
| Sonarr/Radarr | `GET/POST /api/v3/customformat` | listar/crear custom formats |
| Radarr | `PUT /api/v3/movie/editor` | migrar varias pelis: `{"movieIds":[...],"qualityProfileId":N}` |
| Radarr | `GET /api/v3/release?movieId=N` | búsqueda interactiva de releases (tarda ~12 s) |
| Prowlarr | `GET /api/v1/indexer` | listar indexers y sus categorías/tags |
| Bazarr | `GET /api/system/settings` | leer settings (JSON) |
| Bazarr | `POST /api/system/settings` | guardar settings (**form-data**) |

## API keys: dónde están y cómo actualizarlas

Las claves se leen de los archivos de config **dentro del servidor** (por SSH),
nunca de este documento (que se versiona en git). Si una llamada a la API devuelve
`401 Unauthorized`, la clave rotó o cambió: volvé a leerla por SSH con estos comandos.

| App | Comando para obtener la API key | Ubicación |
| --- | --- | --- |
| Sonarr | `grep ApiKey ~/media-stack/config/sonarr/config.xml` | elemento `<ApiKey>` |
| Radarr | `grep ApiKey ~/media-stack/config/radarr/config.xml` | elemento `<ApiKey>` |
| Bazarr | `grep -A1 '^  apikey' ~/media-stack/config/bazarr/config/config.yaml` | sección `auth:` |

Ejemplo:

```bash
ssh jmro@192.168.1.34 'grep ApiKey ~/media-stack/config/sonarr/config.xml'
#   <ApiKey>2bb5...</ApiKey>
```

**Cómo se usan (headers HTTP):**

| App | Header | Base de la API |
| --- | --- | --- |
| Sonarr | `X-Api-Key: <clave>` | `http://192.168.1.34/sonarr/api/v3` |
| Radarr | `X-Api-Key: <clave>` | `http://192.168.1.34/radarr/api/v3` |
| Bazarr | `X-API-KEY: <clave>` | `http://192.168.1.34/bazarr/api` |

> ⚠️ Los archivos de config (`~/media-stack/config/**`) están en `.gitignore` del
> repo: las claves **no** deben volcarse a ningún `.md` commiteado. Si una clave no
> funciona, se actualiza leyendo de nuevo el archivo por SSH (no se edita a mano).

## Soluciones y troubleshooting

| Síntoma | Causa | Solución |
| --- | --- | --- |
| Baja el remux 4K gigante sin latino | Perfil sin Custom Formats; score 0 para todo | Aplicar el perfil `1080p Latino Preferido` al título |
| No baja nada en español (espera versión latina) | `minFormatScore` > 0 | Dejarlo en `0` para que descargue igual en inglés |
| Baja subs en castellano e inglés a la vez | `single_language: false` o perfil con varios idiomas | Activar `single_language: true` y ordenar el perfil con Latino primero |
| No baja ningún subtítulo | Defaults de Bazarr apagados | `serie_default_enabled` / `movie_default_enabled` en `true` |
| Los *arr ignoran el nuevo perfil | El título usa otro perfil | Migrar con `PUT /api/v3/movie/editor` (o re-asignar perfil en la UI) |
| La app muestra el viejo perfil | Caché / no reinició | `docker restart sonarr radarr` |
| Un release con latino no supera a uno sin | Los scores no están en el perfil | Verificar `formatItems` del perfil (Latino=1000, Castellano=500) |

## Notas

- Las API keys quedaron en los `config.xml` de cada app
  (`config/sonarr/config.xml`, `config/radarr/config.xml`) y en
  `config/bazarr/config/config.yaml` (sección `auth.apikey`).
- El stack vive en `~/media-stack` (Docker Compose) en el servidor `192.168.1.34`,
  accesible por SSH. Ver `README.md` y `docs/INDEXERS.md` para el resto.
- Los formatos y regex aplican igual para series (Sonarr) y películas (Radarr).
- **Ids de calidad para el cutoff difieren por app**: en Radarr `WEBDL-1080p` = `7`
  (directo); en Sonarr `WEBDL-1080p` vive dentro del grupo `WEB 1080p` = `1002`
  (si se quiere apuntar al ítem suelto, es el id `3`). Verificá con
  `GET /api/v3/qualityprofile/{id}` qué ids acepta el `cutoff` del perfil.

## Operación completa para IA u operador

### Mapa de servicios y puertos

| Servicio | URL interna | URL pública (proxy Caddy) |
| --- | --- | --- |
| Caddy | — | `http://192.168.1.34` (raíz, dashboard Homarr) |
| Homarr | — | `http://192.168.1.34/` |
| Jellyfin | `http://jellyfin:8096` | `http://192.168.1.34/jellyfin` |
| Sonarr | `http://sonarr:8989` | `http://192.168.1.34/sonarr` |
| Radarr | `http://radarr:7878` | `http://192.168.1.34/radarr` |
| Bazarr | `http://bazarr:6767` | `http://192.168.1.34/bazarr` |
| Prowlarr | `http://prowlarr:9696` | `http://192.168.1.34:9696` |
| qBittorrent | `http://qbittorrent:8080` | `http://192.168.1.34:8080` |
| FlareSolverr | `http://flaresolverr:8191` | `http://192.168.1.34:8191` |
| Jellyseerr | `http://jellyseerr:5055` | `http://192.168.1.34:5055` |
| Portainer | `http://portainer:9000` | `http://192.168.1.34:9000` |

### Comandos operativos (en el server, desde `~/media-stack`)

```bash
ssh jmro@192.168.1.34
cd ~/media-stack

docker compose ps                 # estado de todos los servicios
docker compose logs -f sonarr     # logs en vivo de una app
docker compose logs --tail 100 radarr
docker compose restart sonarr     # reiniciar una app
docker compose up -d              # levantar todo (idempotente)
docker compose pull && docker compose up -d   # actualizar imágenes
docker compose down               # frenar (conserva datos)
```

### Estructura de datos

```
data/
├── torrents/                     # staging (escribe qBittorrent)
│   ├── movies/  series/  music/
└── media/                        # biblioteca final (lee Jellyfin/Bazarr)
    ├── movies/  series/  music/
```

- qBittorrent **Default Save Path** = `/data/torrents` (los *arr crean subcarpetas).
- Sonarr Root Folder = `/data/series`; Radarr = `/data/movies`.
- Jellyfin bibliotecas = `/data/media/movies`, `/data/media/series`, `/data/media/music`.
- No romper la estructura o se pierden los hardlinks/moves atómicos (TRaSH).

### Flujo de trabajo recomendado para una IA

1. **Antes de tocar nada**: leer `README.md`, `docs/DATA_LAYOUT.md`,
   `docs/INDEXERS.md` y esta guía.
2. **Probar acceso a la API**: `curl -s -H "X-Api-Key: <KEY>" .../api/v3/system/status`
   Si da 401, releer la key por SSH (sección "API keys").
3. **Leer el estado antes de modificar**: siempre `GET` el recurso completo
   (perfil, custom format, movie) antes de un `PUT`. Los *arr exigen el JSON
   completo, no parcial.
4. **Modificar con cuidado**: los custom formats solo afectan selección de releases;
   los perfiles afectan qué se descarga. Verificá con un `GET` después de cada cambio.
5. **Verificar el efecto**: consultar `GET /api/v3/movie` (perfil correcto),
   `GET /api/v3/customformat`, y `GET /api/v3/history` para ver qué se grabó.
6. **No tocar la DB de Bazarr** salvo que sea estrictamente necesario (perfil de
   idiomas); después `docker restart bazarr`.

### Qué significa cada ajuste y cuándo usarlo

| Ajuste | Dónde | Efecto |
| --- | --- | --- |
| `Español Latino` score 1000 | Custom Format en Sonarr+Radarr | Pone arriba de todo los releases con `Latino`/`LAT` |
| `Español Castellano` score 500 | Custom Format en Sonarr+Radarr | Pone arriba los releases con `Español`/`Castellano`/`Spanish` |
| `upgradeAllowed: true` | Perfil de calidad | Reemplaza un release existente si aparece uno con mejor puntaje |
| `minFormatScore: 0` | Perfil de calidad | No bloquea: si no hay latino, baja en inglés (no se queda sin descargar) |
| cutoff = 7 (Radarr) / 1002 (Sonarr) | Perfil de calidad | No busca versiones por encima de 1080p WEB-DL |
| `single_language: true` | Bazarr | Baja subs solo del primer idioma del perfil (latino) |
| `serie/movie_default_enabled: true` | Bazarr | Asigna el perfil de idiomas a series/películas automáticamente |

### Checklist ante un problema

1. ¿El título usa el perfil correcto? → `GET /api/v3/movie/{id}` → `qualityProfileId`.
2. ¿El release existe con latino? → `GET /api/v3/release?movieId={id}` (tarda ~12 s).
3. ¿El custom format está en el perfil con su score? → `GET /api/v3/qualityprofile/{id}`.
4. ¿Bazarr ve la serie/película? → `GET /api/system/settings` y la DB de Bazarr.
5. ¿El índice tiene el contenido? → Prowlarr `GET /api/v1/indexer`.
6. ¿Están los indexers sincronizados a la app? → `GET /api/v3/indexer` en la app.

### Datos de acceso actuales (relevar por SSH, no están en git)

| Recurso | Comando para obtenerlo |
| --- | --- |
| SSH | `ssh jmro@192.168.1.34` (clave `id_ed25519` de la notebook) |
| Sonarr API key | `grep ApiKey ~/media-stack/config/sonarr/config.xml` |
| Radarr API key | `grep ApiKey ~/media-stack/config/radarr/config.xml` |
| Bazarr API key | `grep -A1 '^  apikey' ~/media-stack/config/bazarr/config/config.yaml` |
| Prowlarr API key | `grep ApiKey ~/media-stack/config/prowlarr/config.xml` |
