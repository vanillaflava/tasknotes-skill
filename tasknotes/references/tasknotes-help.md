# TaskNotes - Setup and Configuration Guide

Setup instructions, full API reference, and troubleshooting for all TaskNotes access methods. The tasknotes skill reads this file when connection or setup problems arise and uses it to guide users through fixes.

Written for TaskNotes plugin v5 (5.0.0 and later; checked against 5.0.5). For a vault still on 4.x, see "TaskNotes 4.x notes" near the end.

**Plugin docs:** https://tasknotes.dev/
**HTTP API docs:** https://tasknotes.dev/HTTP_API/
**MCP / integrations:** https://tasknotes.dev/features/integrations/

---

## Access methods overview

| Method | Requires | Body content | Best for |
|---|---|---|---|
| MCP server | Obsidian running + toggles on + API token | Yes (fixed in v4.8.0) | Primary - full task ops including body |
| HTTP API | Obsidian running + API toggle on + API token | Yes (fixed in v4.8.0) | Fallback or scripting |
| Filesystem | Any filesystem tool | Full file including body | Obsidian closed; schema diagnostic |
| `mtn` CLI | Node.js + npm | Full file including body | Headless; scripting |

---

## Enabling the MCP server and HTTP API

Both run inside the TaskNotes Obsidian plugin (desktop only). They are only available while Obsidian is running.

**In Obsidian:**
1. Settings → TaskNotes → Advanced → HTTP API
2. Turn on **Enable HTTP API**
3. Turn on **Enable MCP server** (requires the HTTP API)
4. Note the **API port** (default: 8080)
5. Copy the value of **API authentication token** on that same page - every request needs it
6. Restart Obsidian

**Authentication is required (plugin v4.13.0+).** Every HTTP API and MCP request must carry `Authorization: Bearer <token>`. An empty token field no longer means "no authentication": the plugin generates a token instead, and rejects every request that does not present it. A setup that previously connected without a token stops working on upgrade until the token reaches the client config.

**Getting the token.** Settings → TaskNotes → Advanced → HTTP API → **API authentication token**. The plugin's own instruction for generating one: enable HTTP API, leave that field empty, restart Obsidian, then copy the generated value into your client's bearer authentication settings. It is a TaskNotes token, not your AI provider's API key. Update every client whenever it changes.

Read it from the settings pane. Since v5 the token is not in `data.json` at all: it is kept in Obsidian Secret Storage on each device, so every device that runs the HTTP API has its own token, and syncing or committing `data.json` does not share it. Clearing the field does not turn authentication off; requests are rejected until a token is configured or generated at the next start. The skill never reads or writes plugin files to find it.

**Verify:**

```bash
curl -H "Authorization: Bearer <token>" http://127.0.0.1:8080/api/health
```

Expect `{"status":"ok","vault":"...","version":"..."}`. Opening that URL in a browser returns `401 {"success":false,"error":"Authentication required"}` because a browser sends no token - that 401 means the server is running correctly, not that it is broken. No endpoint is exempt, `/api/health` included.

---

## Connecting an agent to the MCP server

The MCP server is exposed at `http://127.0.0.1:{port}/mcp` over streamable HTTP. How you connect depends on your agent platform, but every platform must send `Authorization: Bearer <token>` (plugin v4.13.0+) - a client that cannot set a header cannot connect.

**Use `127.0.0.1`, not `localhost`.** The listener binds to the IPv4 loopback only, and nothing answers on `::1`. On many systems `localhost` resolves to `::1` first, so a client that does not fall back to IPv4 fails with a connection error that reads as "the server is not running". Node-based clients such as `mcp-remote` fall back automatically only from Node 18.18 / 20.0 onwards, where Happy Eyeballs is the default. Addressing the interface directly removes the question.

### Claude Desktop

Edit `claude_desktop_config.json`:

- **Windows:** `C:\Users\{username}\AppData\Roaming\Claude\claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "tasknotes": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:8080/mcp",
        "--header", "Authorization: Bearer <token>"
      ]
    }
  }
}
```

**On Windows, use this form instead.** Claude Desktop does not escape spaces inside `args`, so the header above arrives mangled and the connection fails. Pass the value through the environment, with no space around the colon:

```json
{
  "mcpServers": {
    "tasknotes": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:8080/mcp",
        "--header", "Authorization:${AUTH_HEADER}"
      ],
      "env": { "AUTH_HEADER": "Bearer <token>" }
    }
  }
}
```

The environment form also keeps the token out of the process argument list, where other local users could read it. For stricter handling, `mcp-remote` accepts `--header-file <path>` and reads the header from that file instead, keeping the token out of the client config as well; the trade-off is one more file to place and protect.

Quit and restart Claude Desktop fully after editing - the config is read only at startup.

### Other agent platforms

MCP setup varies by platform. The transport is streamable HTTP; point your agent at `http://127.0.0.1:{port}/mcp` and give it the bearer token.

- **Claude Code:** https://docs.claude.ai/en/docs/claude-code/mcp
- **Cursor:** https://docs.cursor.com/advanced/model-context-protocol
- **Windsurf / Codex CLI / other agents:** check your platform's MCP or tool-server documentation
- **Generic MCP spec:** https://modelcontextprotocol.io/

All platforms that support MCP can connect to the same URL, and all of them must send the bearer token. The `mcp-remote` npm package (used above for Claude Desktop) is a convenience wrapper for clients that cannot reach a remote MCP server directly; a client with native remote support sets the header in its own config instead. Cursor and Codex CLI share the Windows argument-escaping bug described above, so use the environment-variable form on Windows there too.

---

## Finding the configured port

If port 8080 is in use, change it in Obsidian: Settings → TaskNotes → Advanced → HTTP API → **API port**.

Update the port in your agent config to match.

---

## Network binding and security

The HTTP API and MCP server bind to loopback only (`127.0.0.1`) - they are not reachable from other machines, and there is no remote-bind option. Since v4.9.0, browser CORS is also restricted to loopback origins. For remote access, run your own local tunnel/forwarder. Since v4.13.0 authentication is mandatory on both listeners, so loopback binding is no longer the only thing between a local script and your vault; see the endpoint reference below.

---

## Default folder locations and filesystem scope

**Where TaskNotes creates its folders by default:** The plugin creates `TaskNotes/Tasks/` and `TaskNotes/Views/` directly under the Obsidian vault root - not inside any subfolder.

Settings → TaskNotes → Task files → Task storage → **Default tasks folder** controls where new task files are written. The plugin's default value is `TaskNotes/Tasks`.

**Auto-create task folder:** if your plugin version offers an option to create the task folder automatically on startup (4.x had one under General; look under Task files in v5), note what it does: When enabled, Obsidian will recreate the folder at vault root even if you have moved it elsewhere. **If you move your TaskNotes folder inside a narrower filesystem scope, disable this toggle** - otherwise the plugin will recreate an empty folder at vault root and task creation may silently route to the wrong location.

Reference: https://tasknotes.dev/settings/general/ (the v5 "Task files" page)

### If your agent's filesystem scope is narrower than the vault root

Many agent setups scope the filesystem tool to a subdirectory rather than the full vault (e.g., an `Agent Access/` folder rather than the full `Obsidian/` vault root). In this case, the default `TaskNotes/` folder at vault root is outside the agent's reach and the filesystem fallback will not work.

**To bring TaskNotes into scope:**

1. In Obsidian, create a `TaskNotes/` folder inside your scoped directory (e.g., `Agent Access/TaskNotes/`)
2. Move your existing `TaskNotes/Tasks/` and `TaskNotes/Views/` contents into it
3. Update Settings → TaskNotes → Task files → Task storage → Default tasks folder to the new path (e.g., `Agent Access/TaskNotes/Tasks`)
4. **Disable auto-create task folder**, if your version has it, so the plugin does not recreate the folder at vault root
5. Update `tasks_folder` in `tasknotes-config.md` to match (see section below)

**Archive folder:** If archive moves are enabled (Settings → TaskNotes → Task files → Task storage), the plugin moves archived tasks to a subfolder. The default is typically a subfolder under the task folder (e.g., `TaskNotes/Tasks/Archive`). Verify this destination is also inside your filesystem scope, or change it to a path that is. The archive path must be updated in both Obsidian settings and in `tasknotes-config.md`.

### tasknotes-config.md implications

The skill reads `tasknotes-config.md` to find the `tasks_folder` path for filesystem operations. This value must match the actual path configured in Obsidian - they are not linked automatically.

**After moving your TaskNotes folder:**

1. Open `tasknotes-config.md` in your vault
2. Update `tasks_folder` to reflect the new location (use a path relative to the config file's own directory)
3. If you have an archive subfolder configured, also update any archive-related paths in the config
4. Confirm that the path resolves correctly: `tasks_folder` joined with the config file's directory should produce the full path to your tasks folder

**Quick check:** ask your agent to run a schema diagnostic (Workflow 9) after moving folders - it will either find your tasks or tell you the folder is unreachable, confirming whether the paths are aligned.

---

## Task creation defaults (the stale-scheduled gotcha)

The plugin fills omitted fields on new tasks from its task-creation defaults, and it **ships with Default scheduled date set to Today**. Every task created without an explicit scheduled date - through the GUI, natural-language input, note conversion, and the MCP/HTTP create path - is therefore scheduled for its creation day, and reads as stale (a past `scheduled` date) from the next day on.

If you do not want tasks auto-scheduled, change it yourself in the plugin - the skill cannot: **Settings → TaskNotes → Task creation → Defaults → Default scheduled date → None**. The options are None, Today, Tomorrow, and Next Week, and the same control exists for Default due date (which ships as None). On the same page, **Use default projects only when no project is specified** (v5, off by default) stops the default projects being added to a task that already names a project. With Default scheduled date set to None, a new task stays unscheduled unless you give it a date, so `scheduled` becomes a deliberate "I plan to work on this that day" signal rather than noise. Treat `due` as the deadline (a past due is overdue) and `scheduled` as the planned work-day (a past scheduled date has only slipped).

Reference: https://tasknotes.dev/settings/defaults/

---

## Complete HTTP API endpoint reference

Base URL: `http://127.0.0.1:{port}/api`

**Authentication: required (plugin v4.13.0+).** Send `Authorization: Bearer {token}` on every request, `/api/health` included. The token is at Settings → TaskNotes → Advanced → HTTP API → **API authentication token**; if that field is empty when the server starts, the plugin generates one, saves it to that device's Secret Storage, and shows a notice asking you to update your clients. Requests without it return `401 {"success":false,"error":"Authentication required"}`.

Responses use `{"success": true, "data": ...}` or `{"success": false, "error": "..."}`. Common errors: `400` invalid request, `401` missing or wrong token, `404` missing task or resource.

Full docs: https://tasknotes.dev/HTTP_API/

### System

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/health` | Server status and vault info |
| GET | `/api/docs` | OpenAPI spec (JSON) |
| GET | `/api/docs/ui` | Interactive API explorer |
| POST | `/api/nlp/parse` | Parse natural language into task fields (no creation) |
| POST | `/api/nlp/create` | Parse and create task from natural language |

### Tasks

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/tasks` | List tasks (pagination only - no filtering; use query endpoint) |
| POST | `/api/tasks` | Create a task |
| GET | `/api/tasks/:id` | Get one task (frontmatter, plus body in `details`) by vault-relative path |
| PUT | `/api/tasks/:id` | Update task with partial payload |
| DELETE | `/api/tasks/:id` | Permanently delete the task file (the skill asks the user first) |
| POST | `/api/tasks/:id/toggle-status` | Cycle status through configured workflow |
| POST | `/api/tasks/:id/archive` | Toggle archived state (a toggle: check `archived` first) |
| POST | `/api/tasks/:id/complete-instance` | Complete one instance of a recurring task |
| POST | `/api/tasks/:id/materialize-occurrence` | Create or return the occurrence note for a date (idempotent) |
| POST | `/api/tasks/query` | Advanced filtering with AND/OR logic, sorting, grouping |
| GET | `/api/filter-options` | Available statuses, priorities, tags, contexts, projects |
| GET | `/api/stats` | Task counts: total, completed, active, overdue, archived, withTimeTracking |
| POST | `/api/bases/default-files/update` | Overwrite the default `.base` files from current settings (replaces manual edits - ask the user first) |

**`:id` format:** URL-encoded vault-relative path. Example: `Agent%20Access%2FTaskNotes%2FTasks%2Fmy-task.md`. This is the path, not the v5 frontmatter `id` UUID.

**`complete-instance` body:** optional `date` (`YYYY-MM-DD` recommended; an ISO timestamp is also accepted), legacy alias `instanceDate`. Conflicting aliases or a malformed date return `400` without completing anything. If the parent uses occurrence notes, the matching note is completed (and, with **Create next after completion**, created first if missing).

**`materialize-occurrence` body:** `date` (required, e.g. `2026-06-01`).

**Partial updates:** `PUT /api/tasks/:id` accepts a partial payload - only the fields you send change. Tag handling in partial updates is correct as of v4.9.1 (earlier versions could rewrite native tags with `#` prefixes or duplicate the task tag). Since v4.12.4, sending `{"contexts": []}` or `{"blockedBy": []}` clears those fields; earlier versions ignored an empty array, so there was no way to clear them over HTTP.

**Query body format:**
```json
{
  "type": "group",
  "id": "root",
  "conjunction": "and",
  "children": [
    { "type": "condition", "id": "c1", "property": "status", "operator": "is", "value": "open" },
    { "type": "condition", "id": "c2", "property": "projects", "operator": "contains", "value": "[[My Project]]" }
  ],
  "sortKey": "due",
  "sortDirection": "asc"
}
```

**MCP note:** the `tasknotes_query_tasks` MCP tool takes these fields **as top-level arguments** (`conjunction`, `children`, optional `sortKey` / `sortDirection` / `groupKey`). Nesting the group under another key fails with a `conjunction`/`children` validation error; the extra top-level `type` and `id` keys are tolerated (checked on 5.0.5). Invalid properties or operators return an error result, and a valid query with no matches returns `{"all":[]}`. See SKILL.md "Path: MCP".

**Valid filter operators:** `is`, `is-not`, `contains`, `does-not-contain`, `is-before`, `is-after`, `is-on-or-before`, `is-on-or-after`, `is-empty`, `is-not-empty`, `is-checked`, `is-not-checked`, `is-greater-than`, `is-less-than`, `is-greater-than-or-equal`, `is-less-than-or-equal`. The `value` field is omitted for the empty/checked operators over HTTP; the MCP tool rejects a condition with no `value` key, so pass `"value": true` there (checked on 5.0.5 for `is-checked` / `is-not-checked`). User-defined fields use `property: "user:<fieldId>"`; `user:<frontmatter-key>`, `user.<fieldId>` and `user.<frontmatter-key>` are also accepted, bare keys are not. `GET /api/filter-options` lists each field's id and aliases.

**Create task body fields:** `title` (required), `details`, `status`, `priority`, `due`, `scheduled`, `tags`, `contexts`, `projects`, `recurrence`, `recurrence_anchor`, `timeEstimate`, `reminders`, `blockedBy` (array of `{uid, reltype, gap?}`)

### Time tracking

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/tasks/:id/time/start` | Start a time tracking session |
| POST | `/api/tasks/:id/time/start-with-description` | Start session with description |
| POST | `/api/tasks/:id/time/stop` | Stop active session |
| GET | `/api/tasks/:id/time` | Time summary and entries for a task |
| GET | `/api/time/active` | All currently running sessions |
| GET | `/api/time/summary` | Aggregate time summary |

Since v4.13.2 the plugin also exposes start/stop time tracking for the current task note as Command Palette commands, usable with hotkeys or button plugins. That is a path for the user inside Obsidian, not an agent surface.

### Pomodoro

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/pomodoro/start` | Start a pomodoro (body: `{ "taskId": "path", "duration": 25 }`) |
| POST | `/api/pomodoro/stop` | Stop and reset |
| POST | `/api/pomodoro/pause` | Pause |
| POST | `/api/pomodoro/resume` | Resume |
| GET | `/api/pomodoro/status` | Current state, time remaining, stats |
| GET | `/api/pomodoro/sessions` | Session history |
| GET | `/api/pomodoro/stats` | Aggregate stats |

### Calendars

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/calendars` | Provider connectivity overview |
| GET | `/api/calendars/google` | Google Calendar connection details |
| GET | `/api/calendars/microsoft` | Microsoft/Outlook connection details |
| GET | `/api/calendars/subscriptions` | ICS subscriptions |
| GET | `/api/calendars/events` | Merged event list from all providers |

### Webhooks

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/webhooks` | Register a webhook |
| GET | `/api/webhooks` | List webhooks |
| DELETE | `/api/webhooks/:id` | Remove a webhook |
| GET | `/api/webhooks/deliveries` | Delivery history |

Webhook docs: https://tasknotes.dev/webhooks/

---

## TaskNotes v5 - what changed for agents

TaskNotes 5.0.0 is the stable line (5.0.5 at the time of writing). What matters to an agent:

- **Settings regrouped.** Settings use Obsidian's native pages: Task files, Properties, Task creation, Appearance & interaction, Time & reminders, Calendars & integrations, TaskNotes App, Advanced. Obsidian's settings search finds individual options. The HTTP API and MCP toggles, port and token are under **Advanced → HTTP API**; task folder, archive folder and task identification under **Task files**; default dates and projects under **Task creation → Defaults**; the project subtasks (Relationships) widget under **Appearance & interaction → UI elements**.
- **Token in Secret Storage, per device.** On first v5 launch the token moves from `data.json` into Obsidian Secret Storage on that device. A synced device that starts v5 after the token has already left `data.json` generates its own token and shows a notice; the clients on that device need the new token. Clearing the token does not allow unauthenticated access.
- **Stable task IDs.** New and converted tasks get a UUID in frontmatter `id` (key remappable under Properties → Metadata & integration properties → Task ID). Existing tasks without one stay valid and are not rewritten. HTTP and MCP still address tasks by vault path.
- **New optional frontmatter.** `attachments` (a list of file links) and, with type definitions on, `assignees` (links to person notes).
- **mdbase type definitions.** Settings → TaskNotes → TaskNotes App → **Turn on type definitions**: on by default for new installs, unchanged (normally off) for vaults upgraded from 4.x. When on, the plugin maintains `mdbase.yaml`, a hidden `_mdbase/` folder (or existing `_types/`, `_contracts/`, `_settings/`), and backups under `.tasknotes/migrations/`. Agents never edit these. The command **TaskNotes: Check collection** then lists tasks with a missing creation date or an unconfigured status and offers backed-up repairs.
- **Recurring tasks.** Since 5.0.2 the status control of a recurring task is labelled **Series status** and sets the whole series; per-occurrence completion goes through complete-instance. A series whose rule has `COUNT` or `UNTIL` completes itself after its last occurrence.
- **Removed:** the Statistics and Pomodoro-statistics dashboards. The Pomodoro timer, time tracking, stored history and the HTTP/MCP routes remain.
- **Working with Obsidian closed.** The HTTP API settings page points to mdbase Connect and an "mdbase MCP server" for that case. The skill does not route to it; there is no primary documentation for it yet.

Migration guide: https://tasknotes.dev/migration-v4-to-v5/

---

## TaskNotes 4.x notes

The skill defaults to v5. On a vault still on 4.x (4.13.0 and later require the API token):

- **Settings paths.** HTTP API, MCP toggle, port and token are under Settings → TaskNotes → **Integrations** → HTTP API. Task folder and archive folder are under **General**. Default scheduled date is under **Task Properties**. The Relationships widget is under **Appearance → UI Elements**.
- **Token storage.** 4.x keeps the token in the plugin's `data.json`, shared across synced devices. Still read it from the settings pane, never from the file.
- **No `id`, `attachments` or `assignees`** properties, no **Check collection** command, and no **Series status** label (setting a recurring task's status still closes the series).
- **Downgrading from v5 without restoring a backup:** 4.x finds no token in `data.json` and generates a new one, so every client needs updating. If an mdbase upgrade ran, restore `mdbase.yaml` and the task type from `.tasknotes/migrations/`.
- Do not run 4.x and 5.x against the same synced vault; they store some settings differently.

---

## Troubleshooting

### MCP tools not loading

1. Confirm Obsidian is running
2. Confirm **Enable HTTP API** and **Enable MCP server** are both on in Settings → TaskNotes → Advanced → HTTP API
3. Confirm the agent config carries `Authorization: Bearer <token>` and that it matches Settings → TaskNotes → Advanced → HTTP API → **API authentication token** on this device (each device has its own token since v5)
4. Confirm port matches between plugin settings and agent config
5. Check `curl -H "Authorization: Bearer <token>" http://127.0.0.1:8080/api/health`. `{"status":"ok",...}` means the server is up and the agent connection is at fault; `401` means the token is missing or wrong; no response at all means the server is not running
6. Restart both Obsidian and the agent

### `RegistrationRejectedError` / Dynamic Client Registration rejected (HTTP 404)

The signature, as it appears in an MCP client log:

```
Received error (status unknown): Dynamic Client Registration rejected (HTTP 404): {"success":false,"error":"Not found"}
Fatal error: RegistrationRejectedError: Dynamic Client Registration rejected (HTTP 404)
```

**This is a missing or wrong API token, not an OAuth problem.** TaskNotes answers an unauthenticated request with `401`. Clients built on `mcp-remote` read any `401` as "this server wants OAuth", begin OAuth discovery, and ask the server to register a client. TaskNotes has no such endpoint, so it answers `404` and the client exits with the error above. Anything in the log about OAuth callback ports, sign-in coordination, or one instance taking the sign-in over from another is downstream noise from the same cause.

Fix: add the bearer token to the client config (see "Connecting an agent to the MCP server") and restart the client. This appears on upgrade to 4.13.0+ for any setup that previously connected without a token.

### Connection refused on port 8080

- Another application may be using port 8080
- Change the port in plugin settings (Settings → TaskNotes → Advanced → HTTP API → API port)
- Update the port in your agent config and restart

### Windows Firewall prompt

When Obsidian first starts the server, Windows may show a firewall prompt. Allow access on private networks. If you declined previously, go to Windows Firewall settings and add an exception for Obsidian.

### Toggles reset after update

Plugin updates can reset settings. After any TaskNotes update, re-enable both toggles and restart Obsidian. Upgrading to 4.13.0 or later also introduces the token requirement: read Settings → TaskNotes → Advanced → HTTP API → **API authentication token** and make sure the client config carries it. Changing that token later invalidates every client until each one is updated.

### Token stopped working after the v5 upgrade (synced vaults)

In v5 each device keeps its own token in Obsidian Secret Storage. The first device to start v5 moves the old token out of `data.json`; any other synced device that starts v5 afterwards finds none, generates a new token and shows a notice. Clients on that device still sending the old token get `401` (with `mcp-remote`, the `RegistrationRejectedError` above). Copy the token shown under Settings → TaskNotes → Advanced → HTTP API on **that** device into its clients and restart them. Downgrading to 4.x also generates a new token.

### `mcp-remote` errors

- Confirm the health endpoint answers an authenticated `curl` first (see above)
- Confirm `npx` is available in the environment (requires Node.js)
- Reproduce in a terminal with the header attached. Without it you only re-trigger the 401 described above:

  ```bash
  AUTH_HEADER="Bearer <token>" npx -y mcp-remote http://127.0.0.1:8080/mcp --header 'Authorization:${AUTH_HEADER}'
  ```

  A healthy start logs `Using custom headers: Authorization`, then `Connected to remote server` and `Proxy established successfully`.

### Task body content not returned (pre-v4.8.0 only)

If you are on TaskNotes v4.7.x or earlier, MCP and HTTP API GET operations only return frontmatter fields - body content requires a direct filesystem file read. Upgrade to v4.8.0 or later to resolve this. The fix was tracked in GitHub issue #1858 and shipped in v4.8.0.

---

## Filesystem access (Obsidian closed)

When Obsidian is not running, all API access is unavailable. A missing or wrong API token fails the same way from the agent's point of view, so rule that out before concluding Obsidian is closed. Task files are standard `.md` files with YAML frontmatter - any filesystem tool can read and write them directly.

Default task folder location: `{vault_root}/TaskNotes/Tasks/`. If this folder has been moved into a narrower filesystem scope, see the **Default folder locations and filesystem scope** section above for where to find it and what to check in `tasknotes-config.md`.

---

## mdbase-tasknotes CLI (`mtn`)

A standalone Node.js CLI that works without Obsidian running. Reads the vault directly. Supports full body content - titles, notes, checklists.

```
npm install -g mdbase-tasknotes
mtn config --set collectionPath=/path/to/your/vault
```

Key commands:
```
mtn list                              # Open tasks sorted by due date
mtn list --status in-progress         # Filter by status
mtn list --tag work                   # Filter by tag
mtn show "Task title"                 # Full task including body content
mtn search quarterly                  # Full-text search across titles and body
mtn create "Task title @context"      # Natural language creation
mtn update "Task title" --status in-progress
mtn update "Task title" --priority urgent --due 2026-06-01
mtn complete "Task title"
mtn timer start "Task title"
mtn timer stop
mtn projects list --stats
```

Full docs: https://tasknotes.dev/mdbase-tasknotes-cli/
