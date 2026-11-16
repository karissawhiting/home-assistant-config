# Cranberry Perch: Home Assistant Config

Home Assistant configuration for Cranberry Perch (Home Assistant OS running in VirtualBox on a Mac).

## Remote editing: VS Code + Claude Code

### 1. SSH access from the Mac

1. Install the **Advanced SSH & Web Terminal** add-on. The basic Terminal & SSH add-on blocks TCP forwarding, which VS Code Remote-SSH needs. Stop the basic add-on first, since both use the same port.
2. Generate a dedicated key on the Mac:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/ha_key -C "home-assistant"
   pbcopy < ~/.ssh/ha_key.pub
   ```
3. In the add-on config (Edit in YAML):
   - paste the key under `ssh.authorized_keys` (the line ends in `home-assistant`)
   - set `allow_tcp_forwarding: true`
   - under **Network**, set the SSH port to `22`
4. Add to `~/.ssh/config` on the Mac:
   ```
   Host homeassistant
     HostName XXXXXXX
     User root
     IdentityFile ~/.ssh/ha_key
   ```
5. Test from the Mac terminal: `ssh homeassistant`

### 2. Connect with VS Code

1. Install the **Remote - SSH** extension (Microsoft).
2. `Cmd+Shift+P` → **Remote-SSH: Connect to Host…** → `homeassistant` → Linux.
3. **File → Open Folder…** → `/homeassistant`

See [Applying edits](#applying-edits) for how to make changes take effect.

## Applying edits

Files in `/homeassistant` are the live config, so saving a file is enough for HA to see it. Each kind of change still needs a reload before it takes effect.

**1. Check the config first** (catches YAML errors before they break anything):
```bash
ha core check
```

**2. Reload what you changed:**

| You edited | How to apply it |
|---|---|
| `automations.yaml` | **Developer Tools → YAML → Automations** |
| `scripts.yaml` | **Developer Tools → YAML → Scripts** |
| `scenes.yaml` | **Developer Tools → YAML → Scenes** |
| `themes/` | **Developer Tools → YAML → Themes** |
| `dashboards/` (Cranberry Perch, YAML mode) | Open the dashboard → **⋮ → Refresh**, or hard-refresh the browser |
| `configuration.yaml`, `packages/`, new integrations | Full restart: `ha core restart` |

When in doubt, **Developer Tools → YAML → All YAML configuration** reloads everything that can be reloaded without a restart.

**From the terminal:** the add-on can call the HA API directly, which is handy right after editing in VS Code:
```bash
# reload one domain: replace "scene" with automation or script
curl -s -X POST -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
  http://supervisor/core/api/services/scene/reload

# reload all YAML that supports it
curl -s -X POST -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
  http://supervisor/core/api/services/homeassistant/reload_all
```
For themes, use `frontend/reload_themes` as the service path.

> **Edits made in the HA UI** (scenes, automations, scripts) are written straight to these files. They show up in `git status` and need a commit to reach GitHub.

### 3. Claude Code (inside the add-on, Alpine Linux)

```bash
apk add bash curl libgcc libstdc++ ripgrep
curl -fsSL https://claude.ai/install.sh | bash   # takes a few minutes, no progress shown
mkdir -p ~/.claude && echo '{"env":{"USE_BUILTIN_RIPGREP":"0"}}' > ~/.claude/settings.json
export PATH="$HOME/.local/bin:$PATH"
claude   # log in via the link in the Mac browser, paste the code back
```

### 4. Persistence across add-on restarts

The add-on container resets on restart, so Claude Code and its login live in `/data` (the add-on's persistent storage) and are symlinked back into place.

One-time move:
```bash
mkdir -p /data/claude-home
mv ~/.claude ~/.claude.json ~/.local /data/claude-home/
ln -s /data/claude-home/.claude ~/.claude
ln -s /data/claude-home/.claude.json ~/.claude.json
ln -s /data/claude-home/.local ~/.local
rehash
```

Add-on config, which restores packages and links on every start:
```yaml
packages:
  - bash
  - curl
  - libgcc
  - libstdc++
  - ripgrep
init_commands:
  - ln -sfn /data/claude-home/.claude /root/.claude
  - ln -sfn /data/claude-home/.local /root/.local
  - ln -sf /data/claude-home/.claude.json /root/.claude.json
  - echo 'export PATH="$HOME/.local/bin:$PATH"' >> /root/.zshrc
```

> **Note:** `/data/claude-home` holds the Claude login. Never copy it into this repo.

### Troubleshooting

- **`claude: command not found`**: run `rehash` and `export PATH="$HOME/.local/bin:$PATH"`.
- **`ssh` hangs on `homeassistant.local`**: use the IP address in `~/.ssh/config` instead.
- **Commands run in the wrong place**: check the prompt. `/root/...` paths mean you're on HA, not the Mac.
