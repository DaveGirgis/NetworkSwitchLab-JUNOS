# Session 1 — Command Reference

## Mode Navigation

| Command | Description |
|---------|-------------|
| `cli` | Enter Junos CLI from BSD shell |
| `configure` | Enter configuration mode |
| `exit` | Exit current mode (config to operational, or operational to shell) |
| `quit` | Same as `exit` |

## Show Commands (Operational Mode)

| Command | Description |
|---------|-------------|
| `show version` | Junos version, model, uptime |
| `show system information` | Hostname, model, chassis serial |
| `show system uptime` | Boot time and current uptime |
| `show interfaces terse` | All interfaces — state and IP summary |
| `show interfaces ge-0/0/0` | Detailed stats for one interface |
| `show interfaces ge-0/0/0 detail` | Full counters and flags |
| `show route` | Routing table |
| `show configuration` | Full committed configuration |
| `show configuration \| display set` | Committed config in `set` format |

## Configuration Mode Commands

| Command | Description |
|---------|-------------|
| `set <path> <value>` | Stage a configuration change |
| `delete <path>` | Stage a configuration deletion |
| `edit <path>` | Navigate into a config hierarchy level |
| `up` | Go up one level in the hierarchy |
| `top` | Return to top of hierarchy |
| `show` | Show candidate config at current level |
| `show \| compare` | Diff: candidate vs committed (+ added, - removed) |
| `show \| display set` | Candidate config in `set` format |
| `commit` | Apply all staged changes |
| `commit confirmed <minutes>` | Apply changes with auto-rollback timer |
| `commit confirmed` then `commit` | Confirm a `commit confirmed` (disarms timer) |
| `rollback 0` | Discard all staged changes (back to committed) |
| `rollback 1` | Load previous committed config as candidate |
| `rollback ?` | List available rollback generations |

## Session 1 Wrap-Up

The Junos commit model is a fundamental shift from Cisco IOS, where every command takes effect immediately. In Junos:

1. You **build** changes in the candidate configuration
2. You **review** them with `show | compare`
3. You **apply** them with `commit`
4. You can always **revert** with `rollback`

This model makes Junos significantly safer to configure on production equipment, but it requires an extra mental step compared to IOS. Getting comfortable with this workflow in Session 1 pays dividends through every session that follows.
