# Idioma Español Latino y control de tamaño/calidad

Guía de cómo se configuró el stack para que Sonarr/Radarr prefieran contenido con
**audio en español latino**, no bajen remuxes gigantes sin el idioma, y Bazarr agregue
subtítulos en español latino cuando el audio es inglés.

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
