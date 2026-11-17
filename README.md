# MyGitea

Self-hosted Gitea at https://gitea.knowslog.com, served by Docker and exposed through the existing Cloudflare tunnel `orca`.

## Layout

- `docker-compose.yml`: Gitea (SQLite) bound to `127.0.0.1:3030` only. Data lives in the Docker volume `mygitea_gitea-data`.
- Tunnel ingress (outside this repo, `~/.cloudflared/config.yml`):

  ```yaml
  - hostname: gitea.knowslog.com
    service: http://127.0.0.1:3030
  ```

  DNS: `cloudflared tunnel route dns orca gitea.knowslog.com`. Tunnel runs as `systemctl --user ... cloudflared-orca.service`.

## Operate

```bash
docker compose up -d                 # start / apply changes
docker compose logs -f gitea
docker compose pull && docker compose up -d   # upgrade after bumping the image tag
systemctl --user restart cloudflared-orca.service   # after editing tunnel config
```

Backup (stop the container so the SQLite file is consistent):

```bash
docker compose stop gitea
docker run --rm -v mygitea_gitea-data:/data -v "$PWD":/backup alpine tar czf /backup/gitea-$(date +%F).tgz -C /data .
docker compose start gitea
```

## MCP (Claude Code)

Uses the official [gitea-mcp](https://gitea.com/gitea/gitea-mcp) server via Docker.

1. Create an access token: Gitea → Settings → Applications → Generate New Token, or as admin:

   ```bash
   docker exec -u git gitea gitea admin user generate-access-token \
     --username <user> --token-name claude-code-mcp --raw \
     --scopes write:repository,write:issue,write:user,write:organization,write:notification,read:misc,read:package
   ```

2. Register it with Claude Code (user scope; restart Claude Code afterwards):

   ```bash
   claude mcp add --transport stdio --scope user gitea \
     --env GITEA_ACCESS_TOKEN=<token> \
     --env GITEA_HOST=https://gitea.knowslog.com \
     -- docker run -i --rm -e GITEA_ACCESS_TOKEN -e GITEA_HOST docker.gitea.com/gitea-mcp-server:1.13.0
   ```

   Do not append `-t stdio`: the image sets the binary as `CMD`, so extra args replace it and the container fails to start. stdio is already the default.

3. Check: `claude mcp list` shows `gitea: ... ✔ Connected`. In a new session, ask Claude e.g. "gitea에서 내 정보 보여줘" (`get_me` tool).

- The token is stored in plain text in the Claude config (`.claude.json`, or `$CLAUDE_CONFIG_DIR/.claude.json`). Revoke it under Settings → Applications when no longer needed.
- Upgrade: bump the image tag, then `claude mcp remove --scope user gitea` and re-add.

## Notes

- Web installer is locked and self-registration is disabled. Create users as admin (Site Administration, or `docker exec -u git gitea gitea admin user create ...`).
- SSH is disabled: the tunnel only carries HTTP. Clone and push over HTTPS with a password or access token.
