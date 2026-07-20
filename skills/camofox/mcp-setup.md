# camofox MCP setup

Register the `camofox-browser` MCP server so Claude Code can call its tools.

## Option A — claude mcp add (recommended)

From anywhere:

```bash
claude mcp add camofox-browser \
  --env CAMOFOX_BASE_URL=http://localhost:9377 \
  -- node /Users/hackme/workspace/fork/camofox-browser/mcp/server.mjs
```

The `--env` pins the REST server URL. Omit it to use the default
`http://localhost:9377`.

For cookie import, also pass the server's API key:

```bash
claude mcp add camofox-browser \
  --env CAMOFOX_BASE_URL=http://localhost:9377 \
  --env CAMOFOX_API_KEY=<your-server-api-key> \
  -- node /path/to/camofox-browser/mcp/server.mjs
```

## Option B — edit ~/.claude.json manually

Add under `mcpServers`:

```json
{
  "mcpServers": {
    "camofox-browser": {
      "command": "node",
      "args": ["/Users/hackme/workspace/fork/camofox-browser/mcp/server.mjs"],
      "env": {
        "CAMOFOX_BASE_URL": "http://localhost:9377"
      }
    }
  }
}
```

## Verify

1. Start the REST server first (MCP only forwards calls):
   ```bash
   cd /Users/hackme/workspace/fork/camofox-browser
   npm install   # first run only — downloads Camoufox ~300MB
   npm start
   ```
   Wait for `server started, port 9377` + `browser pre-warmed`.
2. In Claude Code, run `/mcp`. You should see `camofox-browser: connected`
   with 11 tools.
3. Test: ask Claude to `create a camofox tab at https://example.com and
   snapshot it`.

## Environment variables

| Var | Default | Purpose |
|-----|---------|---------|
| `CAMOFOX_BASE_URL` | `http://localhost:9377` | REST server URL |
| `CAMOFOX_USER_ID` | `mcp-<random>` | Session isolation key (one MCP server = one session) |
| `CAMOFOX_SESSION_KEY` | `default` | Tab partition within the user |
| `CAMOFOX_API_KEY` | _(unset)_ | Required only for `camofox_import_cookies` |

## Without installing from source

Once published (or after `npm link`), the MCP entrypoint is the `camofox-mcp`
bin, so you can skip the repo path:

```bash
claude mcp add camofox-browser -- camofox-mcp
```
