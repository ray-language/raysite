# raysite

Generador de sitios estáticos (Hugo, en muy pequeño), escrito en [raylang](https://github.com/ray-language/raylang). Es el primer consumidor real del camino **Markdown → HTML** de `std/markdown` (raycode ya estrujaba el AST, pero lo pintaba en ANSI; `to_html` y su modelo de seguridad no tenían app): frontmatter TOML, layouts `std/template`, índice por fecha, RSS, y `serve` con rebuild al guardar.

```text
$ raysite new "Señales y ruido"
content/senales-y-ruido.md

$ raysite build
built 12 page(s) into ./public

$ raysite serve --port 8080     # sirve public/ y reconstruye al cambiar
```

Estructura del sitio: `site.toml` ([site] title/base_url/author) ·
`content/*.md` con frontmatter `+++ title/date/draft +++` ·
`templates/page.html` (opcional, `std/template`: `{{ title }}`,
`{{& content }}`, `{% if %}`) · `public/` generado.

## Qué ejercita

- **`markdown.to_html`** con su modelo de seguridad de serie: el HTML embebido
  se ESCAPA (un `<script>` en un post sale como texto) y las URLs peligrosas
  se neutralizan — verificado en tests. Tablas GFM, cercas, énfasis, enlaces.
- **`std/template`** fuera del servidor: layout compilable con autoescape y
  `{{& }}` para el cuerpo ya-HTML.
- `std/toml` (frontmatter + site.toml), `parse_iso8601` para ordenar por
  fecha, slugify con transliteración castellana, RSS a mano.
- `serve`: `static_response` del webserver + rebuild por EVENTOS de kernel
  (`fs.watch`, raylang M115.4) con debounce — una ráfaga de guardados es un
  solo rebuild; degrada a sondeo de mtimes si el watch no se puede armar.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| build: markdown+frontmatter → HTML con layout, drafts excluidos | ✅ |
| Índice ordenado por fecha desc + RSS (últimos 20) | ✅ |
| Layout propio (`templates/page.html`) o el embebido | ✅ |
| serve con rebuild en vivo (verificado en caliente) | ✅ |
| new: scaffold con slug y fecha | ✅ |
| Binario nativo | ✅ |
| Tests (slugify, frontmatter, build completo con seguridad HTML) | ✅ 3 |
| Subdirectorios en content/, taxonomías, assets copiados | 📋 v2 |

## Hallazgos de dogfood

Anotados en `raylang/IDEAS.md` §71:

1. **`markdown.to_html` cumple su promesa de seguridad** (positivo): el HTML
   embebido sale escapado sin sanitizador externo — el modelo "seguro por
   diseño" aguanta su primera app real.
2. **[RESUELTO — raylang M115.4]** Quinta app sondeando mtimes: `fs.watch` existe y `serve` lo usa.
3. **Sin normalización Unicode**: el slugify translitera a mano las vocales
   acentuadas del castellano y descarta el resto — `NFD`/`NFKD` no existen
   (predicho por el catálogo).
4. Sin globbing (`content/**/*.md` se recorre a mano — patrón walk repetido
   de raysync).

## Desarrollo

```sh
ray test
ray build --native src/main.ray -o raysite --release
```

Estructura: `src/main.ray` (CLI + serve) · `content.ray` (frontmatter, slug,
orden) · `build.ray` (render + índice + RSS).
