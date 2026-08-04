# Indexers, Prowlarr y FlareSolverr

> Este documento cubre la parte que se sacó del video de YouTube sobre el stack.
> YouTube no explica exactamente qué clip infringe sus normas, así que la idea
> es dejar todo el setup detallado acá, en texto, donde sí se puede describir
> paso a paso sin ambigüedad.

Este doc asume que ya tenés el stack levantado (ver `README.md` → Inicio rápido).
Si Prowlarr y FlareSolverr están corriendo, seguimos.

> Prowlarr reemplaza a Jackett: es del mismo ecosistema Servarr que Sonarr/Radarr,
> consume menos RAM y sincroniza los indexers a las apps automáticamente (no hay
> que pegar feeds Torznab a mano).

---

## Por qué hablamos en URLs internas

Todos los servicios del stack comparten la red Docker `proxy`. Eso significa
que **adentro de la red, los containers se ven por nombre**. Si Radarr quiere
hablarle a Prowlarr, no tiene que salir a Internet ni pasar por Caddy: hace
`http://prowlarr:9696` y listo.

Esa es la URL que vamos a usar en todos lados:

| Desde | A Prowlarr | A FlareSolverr |
| --- | --- | --- |
| Desde tu navegador | `http://<tu-servidor>:9696` | `http://<tu-servidor>:8191` |
| Desde un *arr (Sonarr/Radarr) | `http://prowlarr:9696` | `http://flaresolverr:8191` |
| Desde Prowlarr hacia FlareSolverr | — | `http://flaresolverr:8191/` |

Si usás la URL externa (`<tu-servidor>:9696`) desde adentro de los *arr,
funciona también pero pasás por la red pública innecesariamente. La URL
interna es más rápida y no depende de que el proxy esté vivo.

---

## 1. Agregar indexers en Prowlarr

1. Abrí Prowlarr en `http://<tu-servidor>:9696`.
2. Click en **Indexers → Add Indexer**.
3. Buscá el tracker que querés agregar (hay más de 600: 1337x, The Pirate Bay,
   YTS, etc.).
4. Click en él → se abre la pantalla de configuración:
   - Si el tracker pide login (usuario y contraseña), llená los campos. Si es
     público, muchos no piden nada.
   - **App Profile**: elegí `Standard` (habilita RSS y búsquedas automáticas).
5. Click **Test** y **Save**.

Probá que funcione: después de guardar, el estado del indexer tiene que estar
verde. Si está rojo, Prowlarr muestra el motivo (ej: "blocked by Cloudflare").

### Cómo se conectan los *arr

No hace falta copiar feeds ni API keys a mano: Prowlarr sincroniza cada
indexer a Sonarr/Radarr como una conexión Torznab apuntando a
`http://prowlarr:9696/<id>/`. Todo se maneja desde **Settings → Apps** en
Prowlarr (ver sección 3).

---

## 2. Configurar FlareSolverr en Prowlarr

Muchos trackers importantes (1337x, The Pirate Bay y un montón más) ponen un
**Cloudflare Challenge** delante. Sin algo que lo resuelva, Prowlarr no puede
bajar la página y parsear los resultados. Ahí entra FlareSolverr: un proxy que
usa un Chrome headless para resolver el challenge y devolverte el HTML limpio.

> **TL;DR**: si tu indexer favorito falla con un error tipo "blocked by
> Cloudflare Protection" o "403 Forbidden", este paso es lo que te falta.

En Prowlarr, FlareSolverr se configura como **Indexer Proxy** y se asocia a
los indexers mediante un **tag** (el proxy y el indexer tienen que compartir
el mismo tag, si no el proxy queda deshabilitado).

### Paso A — Crear el tag

1. En Prowlarr: **Settings → Tags → Add Tag**.
2. Nombre: `flaresolverr` (o el que prefieras). Anotá que se creó con un ID.

### Paso B — Crear el proxy FlareSolverr

1. En Prowlarr: **Settings → Indexer Proxies → Add → FlareSolverr**.
2. **Name**: `FlareSolverr`.
3. **Host**: `http://flaresolverr:8191/` (notá la barra al final).
4. **Tags**: agregá el tag `flaresolverr` (Paso A).
5. Click **Save**. El proxy queda activo solo para indexers con ese tag.

### Paso C — Asignar el tag a cada indexer

1. En Prowlarr: **Indexers → [tu indexer] → Edit**.
2. En **Tags**, agregá el tag `flaresolverr`.
3. Click **Test** y **Save**.

Solo los indexers que tengan el tag usan FlareSolverr; los demás van directo.

---

## 3. Conectar Sonarr y Radarr como Apps

En Prowlarr, **Settings → Apps → Add Application**:

| Campo | Sonarr | Radarr |
| --- | --- | --- |
| **Name** | `Sonarr` | `Radarr` |
| **Prowlarr URL** | `http://prowlarr:9696` | `http://prowlarr:9696` |
| **Base URL** | `http://sonarr:8989` | `http://radarr:7878` |
| **API Key** | La de Sonarr (`Settings → General` en Sonarr) | La de Radarr (`Settings → General` en Radarr) |
| **Sync Categories** | `5000, 5010, 5020, 5030, 5040, 5045, 5050, 5060, 5070, 5080, 5090` | `2000, 2010, 2020, 2030, 2040, 2045, 2050, 2060` |

Click **Test** (debe dar "Successfully connected") y **Save**.

> **Importante**: la app tiene que quedar **Enable = ON**. Si está apagada,
> Prowlarr no sincroniza ningún indexer. También verificá que el **Sync Level**
> esté en `Full Sync` para que los cambios en Prowlarr se propaguen a los *arr.

Cada indexer que agregues o edites en Prowlarr se sincroniza automáticamente a
Sonarr/Radarr (se agrega, actualiza o elimina la conexión Torznab
correspondiente en cada app). No hay que tocar nada en los *arr.

---

## Troubleshooting

- **"Test was successful" pero las búsquedas devuelven 0 resultados.** El
  indexer funciona pero no tiene feeds para lo que buscás. Probá con algo
  popular (un release conocido) para descartar.
- **Un indexer aparece rojo con "blocked by Cloudflare".** El proxy
  FlareSolverr no está aplicando. Verificá que (1) el proxy está creado en
  Settings → Indexer Proxies, (2) el proxy y el indexer comparten el mismo
  tag, y (3) el indexer tiene ese tag guardado. Sin tag coincidente el proxy
  queda deshabilitado.
- **Los *arr no reciben los indexers nuevos.** La app en Prowlarr tiene que
  estar **enabled** (`Settings → Apps`) y con Sync Level `Full Sync`. Después
  de activarla, Prowlarr sincroniza automáticamente; si no, editá y guardá la
  app de nuevo para forzar la sync.
- **Cloudflare challenge sigue sin resolverse.** Logs: `docker logs
  flaresolverr`. Si ves errores de Chrome headless, puede ser que el container
  no tenga suficiente RAM (FlareSolverr levanta un Chromium real, pide ~300-500
  MB por request). Sumá RAM al host o bajá la concurrencia de indexers.
- **Prowlarr no responde a `http://prowlarr:9696` desde el *arr.** Los dos
  containers tienen que estar en la misma red Docker. Acá lo están (red
  `proxy`), así que si falla es que algo custom lo rompió. Verificá con
  `docker inspect <container> | grep Networks`.
