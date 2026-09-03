# Recuperar la última sesión de OpenCode Desktop

Guía para que cualquier arnés (Claude Code, otro agente, etc.) reconstruya el
contexto de la última sesión de trabajo de **OpenCode Desktop** y pueda
continuar donde se quedó.

---

## 1. Dónde guarda OpenCode Desktop las sesiones

OpenCode Desktop **ya no usa un archivo JSON por sesión**. Todo vive en una
única base de datos SQLite:

```
~/.local/share/opencode/opencode.db          # base principal (puede pesar cientos de MB)
~/.local/share/opencode/opencode.db-wal      # write-ahead log (cambios recientes aún no fusionados)
~/.local/share/opencode/opencode.db-shm      # índice compartido del WAL
```

Otras rutas de OpenCode (contexto, normalmente no hacen falta):

```
~/.local/share/opencode/storage/session_diff/ses_*.json   # snapshots de diffs por sesión
~/.local/share/opencode/snapshot/                          # snapshots de árboles de archivos
~/.local/share/opencode/tool-output/                       # salidas grandes de herramientas
~/.config/opencode/opencode.json                           # configuración del usuario
~/.cache/opencode/models.json                              # catálogo de modelos
```

### Esquema relevante de `opencode.db`

| Tabla      | Contenido                                                                    |
|------------|-----------------------------------------------------------------------------|
| `project`  | Un registro por carpeta raíz abierta (`worktree`).                          |
| `session`  | Una fila por conversación: `id`, `title`, `directory`, `agent`, `model`, `time_created`, `time_updated`, `time_archived`, contadores de tokens/coste. |
| `message`  | Una fila por mensaje. `data` = JSON con `role` (`user`/`assistant`), `time.created`, `agent`, `model`. |
| `part`     | Los trozos de cada mensaje. `data` = JSON con `type`: `text`, `reasoning`, `tool` (con `state.input` / `state.output`), `step-start`, `file`. |
| `todo`     | Lista de tareas de la sesión: `content`, `status` (`pending`/`in_progress`/`completed`), `priority`, `position`. |

> El texto real de la conversación está en `message` + `part`.
> Las tablas `session_message` y `session_input` existen en el esquema pero
> suelen estar vacías: **no las uses**.

---

## 2. Con qué leerla

El sistema normalmente **no trae el binario `sqlite3`**. Usa Python 3, que sí
incluye el módulo `sqlite3` de serie:

```bash
python3 -c "import sqlite3; print(sqlite3.sqlite_version)"
```

**Siempre abrir en solo-lectura** para no corromper la sesión en curso:

```python
con = sqlite3.connect("file:opencode.db?mode=ro", uri=True)
```

> ⚠️ Gotcha de `sqlite3` en Python: no reutilices el mismo cursor para una
> consulta anidada mientras iteras otra — la iteración externa se corta. Crea
> un cursor nuevo por consulta (los helpers de abajo ya lo hacen).

---

## 3. Localizar la sesión

### 3a. Proyectos y sesiones más recientes

```bash
cd ~/.local/share/opencode
python3 - <<'EOF'
import sqlite3, datetime
con = sqlite3.connect("file:opencode.db?mode=ro", uri=True)
def q(sql, a=()):
    c = con.cursor(); c.execute(sql, a)
    cols = [d[0] for d in c.description]
    return [dict(zip(cols, r)) for r in c.fetchall()]
def ts(v):
    if not v: return ""
    v = int(v)
    if v > 1e12: v /= 1000
    return datetime.datetime.fromtimestamp(v).strftime("%Y-%m-%d %H:%M")

print("=== PROYECTOS (por actividad) ===")
for p in q("SELECT id, worktree, time_updated FROM project ORDER BY time_updated DESC LIMIT 20"):
    print(f"{ts(p['time_updated'])}  {p['worktree']}")

print("\n=== 20 SESIONES MÁS RECIENTES ===")
for s in q("""SELECT id, title, directory, agent, model, time_updated, time_archived
              FROM session ORDER BY time_updated DESC LIMIT 20"""):
    print(f"{ts(s['time_updated'])}  {s['id']}  arch={bool(s['time_archived'])}")
    print(f"      {s['title']!r}")
    print(f"      dir={s['directory']}  agent={s['agent']}")
EOF
```

### 3b. Buscar por palabra clave (nombre del proyecto, tema, etc.)

```bash
cd ~/.local/share/opencode
python3 - <<'EOF'
import sqlite3, datetime
con = sqlite3.connect("file:opencode.db?mode=ro", uri=True)
def q(sql, a=()):
    c = con.cursor(); c.execute(sql, a)
    cols = [d[0] for d in c.description]
    return [dict(zip(cols, r)) for r in c.fetchall()]
def ts(v):
    v = int(v);  v = v/1000 if v > 1e12 else v
    return datetime.datetime.fromtimestamp(v).strftime("%Y-%m-%d %H:%M")

TERMINO = "app-demo"   # <-- cámbialo
for s in q("""SELECT id, title, directory, time_updated FROM session
              WHERE lower(title) LIKE ? OR lower(directory) LIKE ? OR lower(slug) LIKE ?
              ORDER BY time_updated DESC""",
           (f"%{TERMINO.lower()}%",)*3):
    print(f"{ts(s['time_updated'])}  {s['id']}  {s['title']!r}  ({s['directory']})")

# Si el título no menciona el término, busca en el texto de los mensajes:
print("--- por contenido del primer mensaje ---")
for s in q("""SELECT DISTINCT s.id, s.title, s.time_updated
              FROM session s JOIN part p ON p.session_id = s.id
              WHERE p.data LIKE ?
              ORDER BY s.time_updated DESC LIMIT 20""", (f"%{TERMINO}%",)):
    print(f"{ts(s['time_updated'])}  {s['id']}  {s['title']!r}")
EOF
```

> La sesión "última" suele ser la de `time_updated` más alto **cuyo
> `directory` coincide con el proyecto**. Ten en cuenta que `directory` puede
> ser la carpeta padre del monorepo, no el subrepo concreto.

---

## 4. Exportar la transcripción completa a Markdown

Sustituye `SID` por el `id` encontrado. Genera un `.md` legible con todos los
mensajes, razonamientos, y llamadas a herramientas (input/output recortados).

```bash
mkdir -p ~/Dev/Resources/opencode/dumps
cd ~/.local/share/opencode
python3 - <<'EOF'
import sqlite3, json, datetime, os
from collections import defaultdict

SID  = "ses_XXXXXXXXXXXXXXXXXXXXXXXXXX"          # <-- cámbialo
DEST = os.path.expanduser(f"~/Dev/Resources/opencode/dumps/{SID}.md")

con = sqlite3.connect("file:opencode.db?mode=ro", uri=True)
def q(sql, a=()):
    c = con.cursor(); c.execute(sql, a)
    cols = [d[0] for d in c.description]
    return [dict(zip(cols, r)) for r in c.fetchall()]

s = q("SELECT * FROM session WHERE id=?", (SID,))[0]
out = [
    f"# {s['title']}  ({SID})",
    f"dir: {s['directory']}",
    f"agent: {s['agent']}   model: {s['model']}",
    f"creada:  {datetime.datetime.fromtimestamp(s['time_created']/1000)}",
    f"última:  {datetime.datetime.fromtimestamp(s['time_updated']/1000)}",
    "",
]

todos = q("SELECT content, status, priority FROM todo WHERE session_id=? ORDER BY position", (SID,))
if todos:
    out.append("## Todos")
    for t in todos:
        out.append(f"- [{t['status']}] ({t['priority']}) {t['content']}")
    out.append("")

msgs  = {m['id']: json.loads(m['data'])
         for m in q("SELECT id, data FROM message WHERE session_id=?", (SID,))}
parts = q("SELECT message_id, data FROM part WHERE session_id=? ORDER BY time_created", (SID,))
bymsg = defaultdict(list)
for p in parts:
    bymsg[p['message_id']].append(json.loads(p['data']))

order = sorted(msgs.items(), key=lambda kv: kv[1].get('time', {}).get('created', 0))
for i, (mid, md) in enumerate(order):
    out.append(f"\n{'='*70}\n## {md.get('role', '?').upper()} #{i}\n")
    for d in bymsg.get(mid, []):
        t = d.get('type')
        if t == 'text':
            out.append(d.get('text', '').strip())
        elif t == 'reasoning':
            out.append("<reasoning>\n" + d.get('text', '').strip() + "\n</reasoning>")
        elif t == 'tool':
            st = d.get('state', {})
            out.append(f"[TOOL {d.get('tool')}] {st.get('status')}\n"
                       f"  in:  {json.dumps(st.get('input', {}))[:3000]}\n"
                       f"  out: {str(st.get('output', ''))[:3000]}")
        elif t == 'file':
            out.append(f"[FILE {d.get('filename', '')}]")

open(DEST, "w").write("\n".join(out))
print("escrito:", DEST, "-", len(order), "mensajes")
EOF
```

### Variante rápida: solo prompts del usuario + respuesta final

Útil para captar la intención sin leer 300 mensajes:

```bash
cd ~/.local/share/opencode
python3 - <<'EOF'
import sqlite3, json
from collections import defaultdict
SID = "ses_XXXXXXXXXXXXXXXXXXXXXXXXXX"   # <-- cámbialo
con = sqlite3.connect("file:opencode.db?mode=ro", uri=True)
def q(sql, a=()):
    c = con.cursor(); c.execute(sql, a)
    cols = [d[0] for d in c.description]
    return [dict(zip(cols, r)) for r in c.fetchall()]
msgs  = {m['id']: json.loads(m['data'])
         for m in q("SELECT id, data FROM message WHERE session_id=?", (SID,))}
parts = q("SELECT message_id, data FROM part WHERE session_id=? ORDER BY time_created", (SID,))
bymsg = defaultdict(list)
for p in parts: bymsg[p['message_id']].append(json.loads(p['data']))
order = sorted(msgs.items(), key=lambda kv: kv[1].get('time', {}).get('created', 0))
for i, (mid, md) in enumerate(order):
    txts = [d['text'] for d in bymsg[mid] if d.get('type') == 'text' and d.get('text', '').strip()]
    if not txts: continue
    if md.get('role') == 'user':
        print(f"\n### USER #{i}\n" + "\n".join(txts))
    elif md.get('role') == 'assistant' and i >= len(order) - 4:
        print(f"\n### ASSISTANT #{i} (final)\n" + "\n".join(txts))
EOF
```

---

## 5. Ver dónde quedó el trabajo

1. **Todos de la sesión** (los imprime el script del punto 4). El que esté en
   `in_progress` es el punto exacto de corte.
2. **Último mensaje del usuario**: si la sesión termina en un mensaje `user`
   sin respuesta `assistant`, la sesión fue interrumpida — ese mensaje es la
   instrucción pendiente.
3. **Árbol de trabajo del repo**: los cambios sin commitear son lo que la
   sesión dejó a medias.

   ```bash
   git -C <ruta-del-subrepo> status -s
   git -C <ruta-del-subrepo> diff
   git -C <ruta-del-subrepo> log --oneline -10
   ```

4. **Planes / doctrina** que la sesión cite en sus mensajes (busca `PLAN.md`,
   `MASTER_PLAN`, `AGENTS.md`, `HARNESS` en la transcripción). Si un
   `docs/PLAN.md` fue borrado, se recupera con:

   ```bash
   git -C <repo> log --oneline --all --diff-filter=D -- docs/PLAN.md
   git -C <repo> show <commit>^:docs/PLAN.md
   ```

---

## 6. Continuar

Con la transcripción (`dumps/<SID>.md`), los todos pendientes, el último
mensaje del usuario y el `git status` del repo, ya hay contexto suficiente
para retomar sin volver a preguntar. Aplica las reglas del proyecto
(`AGENTS.md`, `CONSTRUCTION_HARNESS.md`, skills) igual que la sesión original.
