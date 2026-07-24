### Share ssh keychain for remote ssh session

```bash
claude() {
  if [ -n "$SSH_CONNECTION" ] && [ -z "$KEYCHAIN_UNLOCKED" ]; then
    security unlock-keychain ~/Library/Keychains/login.keychain-db
    export KEYCHAIN_UNLOCKED=true
  fi
  command claude "$@"
}
```
**3. Copy credentials to a file fallback (for automation/headless use)**

bash

```bash
# Run locally (not over SSH), on the Mac itself:
security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" -w > ~/.claude/.credentials.json
chmod 600 ~/.claude/.credentials.json
```

This lets Claude Code read credentials from that file when Keychain isn't reachable, useful for cron/pm2/CI-style headless sessions since Claude Code reads ~/.claude/.credentials.json as a fallback when Keychain is unavailable. Note this file will go stale after a token refresh, so you'd need to re-run it periodically. [GitHub](https://github.com/anthropics/claude-code/issues/44089)
#### Keep the iSH session

**Configure Keep-Alives**: Prevent server timeouts by adding a keep-alive argument directly to your command: `ssh -o ServerAliveInterval=60`