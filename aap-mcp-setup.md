# AAP MCP Server — Claude Code Integration

## What was discovered

- The AAP MCP server runs at a separate OpenShift route: `https://aap-mcp-jacek.apps.ocp.redhat.lab`
- It exposes tools via HTTP streaming transport — each tool group has its own `/mcp` endpoint
- URL pattern: `https://aap-mcp-jacek.apps.ocp.redhat.lab/<tool_group>/mcp`
- Available tool groups:
  - `/job_management/mcp`
  - `/inventory_management/mcp`
  - `/system_monitoring/mcp`
  - `/user_management/mcp`
  - `/security_compliance/mcp`
  - `/platform_configuration/mcp`
- Authentication: Bearer token required via `Authorization` header

## What was configured

Created `.mcp.json` in this directory with one server entry per tool group, all using `${AAP_MCP_TOKEN}` for the Bearer token (see [Hiding the Bearer token](#hiding-the-bearer-token)):

```json
{
  "mcpServers": {
    "aap-mcp-job-mgmt": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/job_management/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    },
    "aap-mcp-inventory-mgmt": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/inventory_management/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    },
    "aap-mcp-system-monitor": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/system_monitoring/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    },
    "aap-mcp-user-mgmt": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/user_management/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    },
    "aap-mcp-security": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/security_compliance/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    },
    "aap-mcp-platform-config": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/platform_configuration/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    }
  }
}
```

## How to activate

1. **Restart Claude Code** from this directory (`/home/jskorzyn/ansible/demos/ansible-mcp`). It reads `.mcp.json` on startup and will connect to AAP automatically.
2. When prompted, approve the `aap` MCP server.
3. The AAP tools are now available in conversation.

## Example usage

- "List all job templates in AAP"
- "Run job template X with inventory Y"
- "Show me the inventory hosts"
- "Check system monitoring status"

## Hiding the Bearer token

Storing the token in plain text in `.mcp.json` is a security risk, especially if the file is committed to git. Use an environment variable instead — Claude Code supports `${ENV_VAR}` substitution in `.mcp.json` headers.

**Step 1** — Replace the token value in `.mcp.json` with an env var reference:

```json
"headers": {
  "Authorization": "Bearer ${AAP_MCP_TOKEN}"
}
```

**Step 2** — Export the token in your shell profile so it is set on every login:

```bash
echo 'export AAP_MCP_TOKEN="<your-token>"' >> ~/.bashrc
source ~/.bashrc
```

**Step 3** — Keep `.mcp.json` out of version control:

```bash
echo '.mcp.json' >> .gitignore
```

**Step 4** — Restart Claude Code so it picks up the env var, then verify the MCP servers connect successfully.

> If you had the token committed previously, rotate it in AAP (Administration → Tokens) to invalidate the exposed value.

## Optional tweaks

**Use only a specific tool group** — add just that server entry to `.mcp.json`, e.g. for job management only:
```json
{
  "mcpServers": {
    "aap-mcp-job-mgmt": {
      "type": "http",
      "url": "https://aap-mcp-jacek.apps.ocp.redhat.lab/job_management/mcp",
      "headers": { "Authorization": "Bearer ${AAP_MCP_TOKEN}" }
    }
  }
}
```
