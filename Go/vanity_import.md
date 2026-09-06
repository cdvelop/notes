# Go Vanity Import Paths

Cómo lograr que tus paquetes Go se importen desde tu propio dominio
(`example.org/db`) en vez de desde el repositorio donde vive el código
(`github.com/acme/db`), sin dejar de alojar el código en GitHub, GitLab, etc.

A lo largo del documento se usan nombres genéricos:

| Rol                        | Ejemplo                    |
|----------------------------|----------------------------|
| Dominio propio (identidad) | `example.org`              |
| Host real del código       | `github.com/acme`          |
| Paquetes                   | `db`, `ui`, `config`       |

---

## 1. Qué es

Un **vanity import path** (o *custom import path* / *vanity URL*) es un mecanismo
**oficial de Go** que permite desacoplar:

- **El import path** — el identificador público del paquete (`example.org/db`).
- **La ubicación física** — dónde está realmente el código (`github.com/acme/db`).

El desarrollador escribe:

```go
import "example.org/db"
```

en vez de:

```go
import "github.com/acme/db"
```

pero el código sigue viviendo en el repositorio de siempre.

---

## 2. Cómo funciona

En el `go.mod` del repositorio declaras el module path con tu dominio:

```go
module example.org/db
```

Cuando alguien ejecuta `go get example.org/db`, la toolchain de Go hace:

```text
go get example.org/db
        │
        ▼
GET https://example.org/db?go-get=1
        │
        ▼
      HTML con <meta name="go-import" ...>
        │
        ▼
github.com/acme/db   (repositorio real)
        │
        ▼
      clone / fetch
```

La pieza central es una etiqueta `<meta>` en el HTML que devuelve tu dominio:

```html
<meta
    name="go-import"
    content="example.org/db git https://github.com/acme/db">
```

Formato del atributo `content`:

```text
<import-prefix>  <vcs>  <repo-url>
example.org/db   git    https://github.com/acme/db
```

- `<vcs>` normalmente es `git` (también `hg`, `svn`, `bzr`, o `mod` para un proxy).
- Go valida que el `go-import` coincida con el prefijo del path solicitado; si
  pides un subpaquete (`example.org/db/driver`), Go consulta el prefijo y
  comprueba que el `go-import` sea coherente. Es un control de seguridad.

> No es un redirect HTTP. Go **no** sigue un `301` hacia GitHub: lee el `<meta>`
> y va directo al repo. Por eso el mecanismo funciona aunque la web "normal" de
> `example.org` sea un sitio completamente distinto.

Analogía: es un **DNS para código**. `example.org` resuelve a un servidor;
`example.org/db` "resuelve" a `github.com/acme/db` para la toolchain de Go.

---

## 3. Qué tiene que servir tu dominio

Para cada paquete, una petición con `?go-get=1` debe devolver el `<meta>`
correspondiente. Ejemplo de respuesta para `GET /db?go-get=1`:

```html
<!doctype html>
<html>
<head>
<meta name="go-import" content="example.org/db git https://github.com/acme/db">
<meta name="go-source" content="example.org/db
      https://github.com/acme/db
      https://github.com/acme/db/tree/main{/dir}
      https://github.com/acme/db/blob/main{/dir}/{file}#L{line}">
</head>
<body>Redirecting to <a href="https://pkg.go.dev/example.org/db">docs</a>.</body>
</html>
```

- `go-import` es **obligatorio** para que funcione.
- `go-source` es **opcional**; ayuda a `pkg.go.dev` a enlazar al código fuente.

No necesitas un servidor grande. Opciones habituales:

- **Hosting estático** (Cloudflare Pages, GitHub Pages, Netlify): un archivo
  HTML por paquete.
- **Función serverless / Worker**: genera la respuesta dinámicamente.
- **Servicio propio mínimo**.

Implementación mínima en Go:

```go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

const domain = "example.org"

// Repos alojados bajo la misma organización: basta con derivar el nombre.
func handler(w http.ResponseWriter, r *http.Request) {
	pkg := strings.Trim(r.URL.Path, "/")
	if pkg == "" || r.URL.Query().Get("go-get") != "1" {
		http.Redirect(w, r, "https://pkg.go.dev/"+domain, http.StatusFound)
		return
	}
	// Solo el primer segmento identifica el módulo raíz.
	mod := strings.SplitN(pkg, "/", 2)[0]

	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	fmt.Fprintf(w,
		`<meta name="go-import" content="%s/%s git https://github.com/acme/%s">`,
		domain, mod, mod,
	)
}
```

Si tus repos no siguen un patrón regular, usa un mapa explícito:

```go
var repos = map[string]string{
	"db":     "github.com/acme/db",
	"ui":     "github.com/acme/frontend-ui", // nombre distinto
	"config": "gitlab.com/acme/config",      // otro host
}
```

---

## 4. `go.mod` es lo que manda

El `module` declarado en `go.mod` **es** el module path, y se convierte en el
prefijo de importación de todos los paquetes del módulo.

```go
// go.mod del repositorio github.com/acme/db
module example.org/db
```

**No** `module github.com/acme/db`.

| Concepto             | Valor                     |
|----------------------|---------------------------|
| Ubicación física     | `github.com/acme/db`      |
| Identidad pública    | `example.org/db`          |

Para Go, `github.com/acme/db` y `example.org/db` son **dos módulos distintos**
aunque el código sea idéntico byte a byte. De ahí que convenga adoptar el vanity
path **pronto**, antes de tener muchos consumidores externos.

---

## 5. Versionado

No cambia nada respecto a lo habitual:

```bash
git tag v1.0.0
git tag v1.2.0
go get example.org/db@v1.2.0
```

Go obtiene la versión desde el repo (o el proxy) y la asocia al module path.

Versiones mayores (`v2+`) siguen la regla de siempre — sufijo en el path y en
`go.mod`:

```go
module example.org/db/v2
```

```go
import "example.org/db/v2"
```

---

## 6. `pkg.go.dev` y GOPROXY

- La documentación aparece en `pkg.go.dev/example.org/db` en lugar de
  `pkg.go.dev/github.com/acme/db`. El module path es el identificador que usa
  todo el ecosistema (proxy, checksum DB, `pkg.go.dev`).
- Por defecto `go get` pasa por `proxy.golang.org`. El proxy resuelve el vanity
  path una vez y **cachea** el resultado, lo que reduce la dependencia de que tu
  dominio esté siempre disponible (pero no la elimina para paquetes nuevos o
  versiones nuevas).
- Variante avanzada: el `go-import` puede apuntar a un **module proxy** propio en
  vez de a un repositorio:

  ```html
  <meta name="go-import" content="example.org/db mod https://proxy.example.org">
  ```

  Con esto Go habla el protocolo GOPROXY con tu servidor y ni siquiera necesita
  saber dónde está el VCS. Útil si quieres controlar distribución, caché o
  proveniencia — pero es un paso opcional y posterior.

---

## 7. Motivación: separar marca de infraestructura

Sin vanity path, la identidad pública de tu paquete queda atada a:

```text
host (github.com)  +  organización (acme)  +  nombre del repo (db)
```

Si en el futuro cambias de host, mueves o renombras el repo, cambias de
organización, transfieres o vendes el proyecto, o decides autoalojarte, **todos
los imports de todos los usuarios se rompen**.

Con `example.org/db`, el identificador público es tuyo y estable:

```text
Hoy:      example.org/db  →  github.com/acme/db
Mañana:   example.org/db  →  gitlab.com/acme/db
Futuro:   example.org/db  →  git.example.org/db
```

El usuario nunca toca su código. GitHub (o quien sea) pasa a ser
**infraestructura**, no **identidad**.

```text
                 example.org           ← marca / namespace
                     │
        ┌────────────┼────────────┐
        │            │            │
       /db          /ui        /config
        │            │            │
        ▼            ▼            ▼
  github.com/    github.com/  github.com/  ← infraestructura
  acme/db        acme/ui      acme/config
```

Además, el dominio se vuelve el hogar natural del proyecto: docs, API y paquetes
bajo el mismo namespace (`example.org/docs`, `example.org/db`, ...).

---

## 8. Pros y contras

### Pros

1. **Marca / profesionalidad** — `example.org/db` comunica un ecosistema
   propio; `github.com/acme/db` comunica "un repo más".
2. **Independencia del host** — puedes migrar de GitHub a GitLab, a un servidor
   propio, etc., sin romper imports.
3. **Control del namespace** — decides tú la organización lógica
   (`example.org/db`, `/ui`, `/http`, ...).
4. **Documentación coherente** — el dominio es la puerta de entrada al proyecto.
5. **Estabilidad ante cambios organizativos** — renombrar el repo, mover la org
   o transferir el proyecto no afecta a los consumidores.

### Contras

1. **Dependencia del dominio** — el dominio se convierte en **infraestructura
   crítica**. Si lo pierdes (caduca, se transfiere, cae el DNS), los builds y
   descargas nuevas fallan. Mitigable en parte por el caché de `proxy.golang.org`,
   pero no del todo.
2. **Hay que mantener un servicio** — aunque sea un HTML estático, alguien tiene
   que asegurarse de que siga en pie.
3. **Migración inicial** — cambiar `module github.com/acme/db` →
   `module example.org/db` y actualizar imports en todo el código.
4. **Es un cambio de módulo, no un alias** — para Go son módulos distintos, así
   que el ecosistema (proxy, sumas de verificación) tratará el cambio como una
   ruta nueva. Conviene hacerlo temprano.

---

## 9. Checklist de adopción

1. Registrar el dominio y asegurar su renovación a largo plazo (auto-renew,
   contacto de propietario claro).
2. Servir, para cada módulo, `https://example.org/<mod>?go-get=1` con el
   `<meta name="go-import">` correcto (HTML estático o función serverless).
3. En cada repo, cambiar `go.mod` a `module example.org/<mod>` (y `/v2`, `/v3`
   donde corresponda).
4. Actualizar todos los imports internos al nuevo path.
5. Etiquetar una versión (`git tag vX.Y.Z`) y verificar con
   `go get example.org/<mod>@vX.Y.Z` desde un módulo limpio.
6. Comprobar que `pkg.go.dev/example.org/<mod>` indexa correctamente.
7. (Opcional) Añadir `<meta name="go-source">` para enlaces al código fuente.
8. Dejar el repositorio como "solo código"; enviar issues, docs y web al dominio.

---

## Referencias

- `go` command — custom import paths: <https://pkg.go.dev/cmd/go#hdr-Remote_import_paths>
- `go.mod` file reference: <https://go.dev/doc/modules/gomod-ref>
- Managing module source: <https://go.dev/doc/modules/managing-source>
- Go Modules Reference: <https://go.dev/ref/mod>
