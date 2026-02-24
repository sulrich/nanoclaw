---
name: add-things
description: Add Things 3 todo integration to NanoClaw. Gives agents five MCP tools to list, search, add, update, and complete todos via the things-cli binary. Triggers on "add things", "things integration", "things todo", "setup things", or "things cli".
---

# Add Things 3 Integration

This skill wires the local `things` CLI binary into the NanoClaw IPC bridge so that
agents running in containers can manage the user's Things 3 todos via five MCP tools:
`things_list`, `things_search`, `things_add`, `things_update`, and `things_delete`.

The integration is **read/write**:
- Read commands (`list`, `search`, `show`) query the Things SQLite database directly — no auth needed, Things 3 does not need to be running.
- Write commands (`add`, `update`, `delete`) use the Things URL scheme — Things 3 must be running. `update` additionally requires an auth token.

## Phase 1: Pre-flight

### Check if already applied

Read `container/agent-runner/src/ipc-mcp-stdio.ts`. If it contains `things_list` the code
changes are already in place — skip to Phase 3 (Configure Auth Token).

### Check Things CLI

Verify the binary exists and is executable:

```bash
/opt/homebrew/bin/things --version 2>&1
```

If not found, check alternate locations:

```bash
which things 2>&1
ls /usr/local/bin/things 2>/dev/null
```

If missing entirely, tell the user:

> Install things3-cli via Homebrew:
>
> ```bash
> brew install ossianhempel/tap/things3-cli
> ```
>
> Or download a release from https://github.com/ossianhempel/things3-cli and place it in `/opt/homebrew/bin/things`.

Once confirmed, ask if the binary is at a non-default path:

AskUserQuestion: Where is your `things` binary? (default: `/opt/homebrew/bin/things`)
- `/opt/homebrew/bin/things` — Homebrew on Apple Silicon (Recommended)
- `/usr/local/bin/things` — Homebrew on Intel Mac
- Custom path — I'll enter it

If they choose a custom path, collect it and use it in Phase 2.

## Phase 2: Apply Code Changes

Make all four changes below. If a file already has the change (detected by checking for key
strings), skip that step.

### 2a. Add `THINGS_CLI_PATH` to `src/config.ts`

Check if `THINGS_CLI_PATH` already exists:

```bash
grep -q "THINGS_CLI_PATH" src/config.ts && echo "already present" || echo "needs adding"
```

If missing, add after the `IPC_POLL_INTERVAL` line:

```typescript
export const THINGS_CLI_PATH =
  process.env.THINGS_CLI_PATH || '/opt/homebrew/bin/things';
```

Replace `/opt/homebrew/bin/things` with the user's actual path if different.

### 2b. Add host-side Things request processor to `src/ipc.ts`

Check:

```bash
grep -q "THINGS_CLI_PATH" src/ipc.ts && echo "already present" || echo "needs adding"
```

If missing, apply all three edits:

**Import additions** — add to the import block at the top:

```typescript
import { spawnSync } from 'child_process';
// (add to existing config import):
import { ..., THINGS_CLI_PATH } from './config.js';
// (add to existing local imports):
import { readEnvFile } from './env.js';
```

**Command allowlist** — add after existing module-level constants:

```typescript
const ALLOWED_THINGS_COMMANDS = new Set([
  'today', 'inbox', 'upcoming', 'anytime', 'someday', 'logbook',
  'logtoday', 'createdtoday', 'completed', 'canceled', 'trash',
  'repeating', 'deadlines', 'all', 'projects', 'areas', 'tags', 'tasks',
  'search', 'show', 'add', 'update', 'delete',
]);

function writeThingsResponse(
  dir: string,
  requestId: string,
  result: string,
  error?: string,
): void {
  fs.mkdirSync(dir, { recursive: true });
  const responseFile = path.join(dir, `${requestId}.json`);
  const tempPath = `${responseFile}.tmp`;
  fs.writeFileSync(tempPath, JSON.stringify({ result, error }));
  fs.renameSync(tempPath, responseFile);
}
```

**Processing block** — add inside the per-group loop in `processIpcFiles`, after the tasks
block and before `setTimeout(processIpcFiles, IPC_POLL_INTERVAL)`:

```typescript
// Process Things requests from this group's IPC directory
try {
  const thingsRequestsDir = path.join(ipcBaseDir, sourceGroup, 'things-requests');
  const thingsResponsesDir = path.join(ipcBaseDir, sourceGroup, 'things-responses');

  if (fs.existsSync(thingsRequestsDir)) {
    const requestFiles = fs
      .readdirSync(thingsRequestsDir)
      .filter((f) => f.endsWith('.json'));

    for (const file of requestFiles) {
      const filePath = path.join(thingsRequestsDir, file);
      let requestData: { requestId: string; command: string; cliArgs: string[] };

      try {
        requestData = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
        fs.unlinkSync(filePath);
      } catch (err) {
        logger.error({ file, sourceGroup, err }, 'Error reading Things request');
        continue;
      }

      if (
        !requestData.requestId ||
        !requestData.command ||
        !Array.isArray(requestData.cliArgs) ||
        !requestData.cliArgs.every((a) => typeof a === 'string')
      ) {
        logger.warn({ requestData }, 'Invalid Things request format, skipping');
        continue;
      }

      if (!ALLOWED_THINGS_COMMANDS.has(requestData.command)) {
        logger.warn({ command: requestData.command, sourceGroup }, 'Unknown Things command blocked');
        writeThingsResponse(thingsResponsesDir, requestData.requestId, '', `Unknown Things command: ${requestData.command}`);
        continue;
      }

      const thingsConfig = readEnvFile(['THINGS_AUTH_TOKEN']);
      const thingsEnv: NodeJS.ProcessEnv = {
        HOME: process.env.HOME,
        PATH: process.env.PATH,
        TMPDIR: process.env.TMPDIR,
        USER: process.env.USER,
        LOGNAME: process.env.LOGNAME,
        ...(thingsConfig.THINGS_AUTH_TOKEN
          ? { THINGS_AUTH_TOKEN: thingsConfig.THINGS_AUTH_TOKEN }
          : {}),
      };

      let result = '';
      let error: string | undefined;

      try {
        const proc = spawnSync(
          THINGS_CLI_PATH,
          [requestData.command, ...requestData.cliArgs],
          { encoding: 'utf-8', timeout: 8000, env: thingsEnv },
        );
        if (proc.error) {
          error = proc.error.message;
        } else if (proc.status !== 0) {
          error = (proc.stderr || `Exit code: ${proc.status}`).trim();
        } else {
          result = proc.stdout;
        }
      } catch (err) {
        error = err instanceof Error ? err.message : String(err);
      }

      writeThingsResponse(thingsResponsesDir, requestData.requestId, result, error);
      logger.info(
        { sourceGroup, command: requestData.command, hasError: !!error },
        'Things request processed',
      );
    }
  }
} catch (err) {
  logger.error({ err, sourceGroup }, 'Error processing Things requests');
}
```

### 2c. Add Things MCP tools to `container/agent-runner/src/ipc-mcp-stdio.ts`

Check:

```bash
grep -q "THINGS_REQUESTS_DIR" container/agent-runner/src/ipc-mcp-stdio.ts && echo "already present" || echo "needs adding"
```

If missing, add after the existing `TASKS_DIR` constant:

```typescript
const THINGS_REQUESTS_DIR = path.join(IPC_DIR, 'things-requests');
const THINGS_RESPONSES_DIR = path.join(IPC_DIR, 'things-responses');
```

Then add the `thingsRequest` helper and five MCP tool definitions before the `StdioServerTransport`
line. See the full implementation in `container/agent-runner/src/ipc-mcp-stdio.ts` — the five
tools are: `things_list`, `things_search`, `things_add`, `things_update`, `things_delete`.

The core `thingsRequest` helper:

```typescript
async function thingsRequest(
  command: string,
  cliArgs: string[],
): Promise<{ content: Array<{ type: 'text'; text: string }> }> {
  const requestId = `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;

  writeIpcFile(THINGS_REQUESTS_DIR, {
    requestId, command, cliArgs, groupFolder,
    timestamp: new Date().toISOString(),
  });

  const responseFile = path.join(THINGS_RESPONSES_DIR, `${requestId}.json`);
  const deadline = Date.now() + 10000;

  while (Date.now() < deadline) {
    if (fs.existsSync(responseFile)) {
      try {
        const response = JSON.parse(fs.readFileSync(responseFile, 'utf-8'));
        try { fs.unlinkSync(responseFile); } catch { /* ignore */ }
        if (response.error) {
          return { content: [{ type: 'text' as const, text: `Error: ${response.error}` }] };
        }
        return { content: [{ type: 'text' as const, text: response.result || '(no output)' }] };
      } catch { /* file mid-write, retry */ }
    }
    await new Promise((r) => setTimeout(r, 200));
  }

  return {
    content: [{ type: 'text' as const, text: 'Error: Things request timed out after 10 seconds' }],
  };
}
```

### 2d. Create agent skill documentation

Check if `container/skills/things/SKILL.md` exists:

```bash
ls container/skills/things/SKILL.md 2>/dev/null && echo "exists" || echo "missing"
```

If missing, create `container/skills/things/SKILL.md` documenting the five tools so agents
know how and when to use them. See the existing file for the template — it covers tool
signatures, usage examples, and workflow tips.

### 2e. Validate

```bash
npm run build
```

Build must be clean. If it fails, read the TypeScript errors and fix them before continuing.

## Phase 3: Configure Auth Token

`things update` (completing/rescheduling existing todos) requires a Things URL scheme token.
`things_list`, `things_search`, and `things_add` work without it.

### Check if already configured

```bash
grep -q "THINGS_AUTH_TOKEN" .env && echo "configured" || echo "not set"
```

### Set up the token

Tell the user:

> To enable `things_update` (completing, rescheduling, or modifying existing todos), you need
> a Things URL scheme auth token:
>
> 1. Open **Things 3**
> 2. Go to **Settings → General → Things URLs**
> 3. Enable **"Allow 'things' CLI to access Things"** (or copy the token shown there)
> 4. The token looks like a short alphanumeric string

Once they have the token, add it to `.env`:

```
THINGS_AUTH_TOKEN=<their-token>
```

If they want to skip this for now, `things_list`, `things_search`, and `things_add` will still
work — only `things_update` will fail with an auth error.

### Verify the token works

```bash
/opt/homebrew/bin/things update --id=nonexistent-id --completed 2>&1
```

An auth error like `authorization token required` means the token is missing or wrong.
An error like `todo not found` or similar means the token is valid and auth is working.

## Phase 4: Rebuild and Restart

```bash
./container/build.sh
npm run build
```

Then restart the service:

```bash
# macOS:
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
# Linux:
systemctl --user restart nanoclaw
```

## Phase 5: Test

### Quick read test (no auth needed)

Run this directly on the host to confirm the CLI and database access work:

```bash
/opt/homebrew/bin/things today --json 2>&1 | head -5
```

Expected: a JSON array of today's todos, or `[]` if today is empty.

If you see a permissions error like `operation not permitted` or the output is empty when
Things has todos, the process needs **Full Disk Access** — see Troubleshooting.

### Test via agent

Send a message to your main agent group:

> @Andy what's on my Things today?

The agent should call `mcp__nanoclaw__things_list` with `view="today"` and return the list.

To test write access (requires auth token):

> @Andy add a todo called "test from agent"

Then check Things inbox for the new item.

## Troubleshooting

### "operation not permitted" or empty results

Things 3 stores its database in the app sandbox. The process running NanoClaw needs
**Full Disk Access**:

- **macOS:** System Settings → Privacy & Security → Full Disk Access
- Add your **terminal app** (Terminal, iTerm2, Warp, etc.)
- If running as a launchd service, add the `node` binary itself:
  ```bash
  which node   # e.g., /opt/homebrew/bin/node
  ```
  Add that binary to Full Disk Access.
- Restart NanoClaw after granting access.

### "authorization token required" on things_update

The `THINGS_AUTH_TOKEN` is missing or wrong. Re-run phase 3 to set it up.
Confirm it's in `.env`:

```bash
grep THINGS_AUTH_TOKEN .env
```

### Things not found / binary path wrong

Check the configured path:

```bash
grep THINGS_CLI_PATH src/config.ts
```

Override at runtime without editing code by setting in `.env`:

```
THINGS_CLI_PATH=/your/actual/path/to/things
```

### Timeout errors from agent

The host-side IPC poll runs every 1 second. The MCP tool waits up to 10 seconds.
Timeouts usually mean the host process is not running or `ipc.ts` has an error.

Check logs:

```bash
tail -50 logs/nanoclaw.log | grep -i things
```

### Things 3 not running (for add/update/delete)

Write commands go through the Things URL scheme, which requires Things 3 to be open.
Read commands (`list`, `search`) work even when Things is closed.

If Things closes itself after launches, check System Settings → General → Login Items
and ensure Things 3 is in the "Open at Login" list.

## Key Files

| File | Purpose |
|------|---------|
| `src/config.ts` | `THINGS_CLI_PATH` constant |
| `src/ipc.ts` | Host-side request processor: runs CLI, writes responses |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP tools exposed to agents |
| `container/skills/things/SKILL.md` | Agent-facing documentation |
| `.env` | `THINGS_AUTH_TOKEN` for update operations |

## Removal

To remove the integration:

1. Remove `THINGS_CLI_PATH` from `src/config.ts`
2. Remove the Things request processing block from `src/ipc.ts` (the `things-requests` section)
3. Remove the `THINGS_REQUESTS_DIR`, `THINGS_RESPONSES_DIR` constants, `thingsRequest` helper, and the five `server.tool('things_*', ...)` calls from `container/agent-runner/src/ipc-mcp-stdio.ts`
4. Delete `container/skills/things/`
5. Remove `THINGS_AUTH_TOKEN` from `.env`
6. Rebuild: `npm run build && ./container/build.sh`
7. Restart the service
