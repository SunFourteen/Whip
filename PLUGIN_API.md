# Whip Plugin API

A plugin is a plain HTML/CSS/JS page running in a WebView. It talks to the SSH core
through the injected global `SshBridge`. **Unless stated otherwise, every method returns
a JSON string that the plugin parses itself.**

## Plugin package

```text
your-plugin/
├── manifest.json
├── index.html          # the page the plugin list opens
└── ...                 # plus everything listed in manifest.files
```

`manifest.json`:

```json
{
  "id": "your-plugin",
  "name": "Your Plugin",
  "version": "0.1.0",
  "description": "one line, shown in the plugin list",
  "entry": "index.html",
  "depends": { "agent-base": "0.9.0" },
  "setup": {
    "command": "bash install.sh",
    "when": "version",
    "writes": ["~/.your-plugin"]
  },
  "files": ["index.html", "app.js", "app.css"]
}
```

| Field | Meaning |
| --- | --- |
| `id` | `[a-zA-Z0-9][a-zA-Z0-9_-]{0,63}`, also the directory name on the host |
| `name` / `description` | shown in the plugin list |
| `version` | dotted version (`0.9.0`); the app compares it to decide on updates |
| `entry` | page opened from the plugin list |
| `files[]` | the package contract: exactly what ships — a listed file that is missing fails packaging |
| `depends` | optional; `pluginId: minimum version`, installed first |
| `setup` | optional host-side runtime install, see “Host-side setup” below |

Install a plugin by putting its directory on the target host:

```text
~/.whip/plugins/your-plugin/
```

To distribute several plugins, package them as a **tap** — a static `index.json` plus
`packages/<id>-<version>.tar.gz`, built with `tools/pack.mjs` in the Whip repository. The
app fetches the index, verifies each package sha256, unpacks it into
`~/.whip/plugins/<id>/` and caches a copy locally for rendering.

## Overview

| Capability | Methods |
| --- | --- |
| Command execution | `run(cmd, opts?)` |
| Interactive shells | `listShells` `shellWrite` `shellTail` `openTerminalShell` `openShellWith` |
| Data feeds | `createFeed` `feedVersion` `feedRead` `removeFeed` |
| State / connection | `getState` `getStateVersion` `isConnected` `getConnection` `getTheme` |
| Persistence | `prefsGet` `prefsSet` |
| Plugin management | `syncRemotePlugins` `getSyncStatus` `getPluginDiff` `getRemotePlugins` `getLocalPluginUrl` `pullPlugin` `removeLocalPlugin` `removeRemotePlugin` |
| Host-side setup | `pluginSetup` |
| Store / tap | `tapRefresh` `tapStatus` `tapPlugins` `tapInstall` |
| Navigation | `navigateTo` `openPluginMarket` |
| Wearable VM (inner pages only) | `vmInfo` `vmListProfiles` `vmGetProfile` `vmDeleteProfile` `vmSaveAndConnect` `vmConnectProfile` `vmOpState` `vmNewShell` `vmCloseShell` `vmDisconnect` `vmResizeShell` `vmListThemes` `vmSetTheme` `vmSetBackHint` `vmSetShellBarRect` |

## 1. Command execution

```js
const r = JSON.parse(SshBridge.run("git -C ~/project status -sb", "{}"));
// { ok, exitCode, stdout, stderr, timedOut, durationMs, error? }
```

- `opts`: `{ timeoutMs?: number (default 8000), cwd?: string }`
- stdout is capped at 256KB, stderr at 64KB.
- Judge success by `ok` / `exitCode` / `timedOut`; never match on output text.

## 2. Interactive shells

```js
const shells = JSON.parse(SshBridge.listShells());
// [{ id, number, name, status, cols, rows, tty }]

SshBridge.shellWrite(id, "ls -la\r");
const t = JSON.parse(SshBridge.shellTail(id));
// { text, version, closed }

// Jump to Terminal and focus a shell (empty id / "latest" = current active shell)
SshBridge.openTerminalShell(id);

// Open a new shell and run `cd <cwd> && <command>` once ready (agent session restore)
SshBridge.openShellWith(cwd, command);
```

`tty` is the remote `pts/N` (the pane tty under tmux) and is used to match an agent to
its shell.

## 3. Feeds

```js
// file-lines: tail a remote file by line / json-line
SshBridge.createFeed("agent", JSON.stringify({
  type: "file-lines", path: "/root/.whip/monitor/state/agent.ndjson",
  framing: "json-line"
}));

// shell output observation
SshBridge.createFeed("sh1", JSON.stringify({
  type: "shell", shellId: "<id>", framing: "chunk"
}));

// periodic execution
SshBridge.createFeed("sys", JSON.stringify({
  type: "exec-loop", cmd: "df -P -B1 -T", intervalMs: 2000
}));
```

Read with `feedVersion(id)` then `feedRead(id, fromVersion)`, which returns
`{records, nextVersion, ended, dropped}`. Limits: 300 records / 64KB per feed; `json-line`
delivers one JSON object per line; a feed is cleaned up with its page.

## 4. State and persistence

```js
const s = JSON.parse(SshBridge.getState("connection"));
// { connected, host, port, user, shellCount }
```

- `prefsGet(key)` / `prefsSet(key, value)` read and write the app's native
  SharedPreferences (shared across pages and WebViews, survives restarts). Keep
  persistent state such as archives here, not in `file://` localStorage.

## 5. Plugin management and marketplace

```js
SshBridge.syncRemotePlugins();                 // auto: update cached plugins only
SshBridge.getSyncStatus();                     // {state,message,...}

const diff = JSON.parse(SshBridge.getPluginDiff());
// { remoteIds:[...], pullable:[...], localOnly:[...], pendingRemote:[...] }
// pendingRemote = deleted locally, removed from the host on the next sync

SshBridge.pullPlugin(id);                      // pull one remote plugin
SshBridge.removeLocalPlugin(id);               // delete the local cache; the host copy
                                               // goes away on the next sync (tombstone)
SshBridge.removeRemotePlugin(id);              // delete the target directory
SshBridge.openPluginMarket();                  // open the native management page

// Read-only queries used by the plugin list page
const remotes = JSON.parse(SshBridge.getRemotePlugins());
// [{ id, name, version, description, entry, localUrl, ... }], base-style deps excluded
SshBridge.getLocalPluginUrl(id);               // local file:// URL, "" when not cached
```

Source model: plugins live on the target host (`~/.whip/plugins/<id>/`) and are pulled
into a local cache that the WebView renders from. Auto sync only updates plugins that
are already cached; adding and removing happen on the marketplace/management page.
`getPluginDiff()` reflects the last sync (empty after a restart), while
`getRemotePlugins()` reflects the current local registry.

### Tap (the app as package manager)

A tap is a static plugin repository — `index.json` plus
`packages/<id>-<version>.tar.gz`, built by `tools/pack.mjs`. `tapInstall` downloads the
package, verifies its sha256, uploads it to the connected host, unpacks it into
`~/.whip/plugins/<id>/` and caches it locally for rendering; dependencies declared in
the index are installed first. Public and private taps differ only by token.

```js
SshBridge.tapRefresh("https://example.com/tap/index.json", ""); // token for private taps
SshBridge.tapRefresh("", "");          // reuse the remembered source

const st = JSON.parse(SshBridge.tapStatus());
// { state: "idle"|"refreshing"|"downloading"|"uploading"|"installing"|"done"|"error",
//   message, current, total, url, count }

const entries = JSON.parse(SshBridge.tapPlugins());
// [{ id, name, version, description, kind, entry, sha256, size, depends }]

SshBridge.tapInstall("agent-monitor");  // requires a connected host
```

### Host-side setup

A plugin may declare host-side setup in its manifest (`setup.command`, run in the plugin
directory on the target host; exit code 0 means success, `setup.when` decides whether it
runs once per version or on every check). The installer runs it after pull/sync, so the
host runtime is installed by the installer instead of by the plugin's own JavaScript.
`pluginSetup` lets a plugin ask for its own setup and render the outcome:

```js
const st = JSON.parse(SshBridge.pluginSetup("agent-base"));
// { ok, state: "none"|"ok"|"error", version, message }
```

## 6. Wearable VM extensions

Available only inside the Wearable tab's small-screen VM (`vmInfo().vm === true`). They
share the host's `SshManager`, `ProfileStorage` and `ThemeManager`: only the connection
and static assets are shared, and navigation never leaves the VM.

```js
const info = JSON.parse(SshBridge.vmInfo());
// { vm, connected, host, user, shellCount }

// Nodes (passwords excluded; the edit form uses vmGetProfile / vmSaveAndConnect)
SshBridge.vmListProfiles();                    // [{id,name,host,port,username,useTmux,tmuxSession,useVpn,useTunnel}]
SshBridge.vmGetProfile(id);                    // full node config (prototype trust model)
SshBridge.vmSaveAndConnect(profileJson);       // insert/update and connect; poll vmOpState()
SshBridge.vmConnectProfile(id);                // connect using a saved node
SshBridge.vmDeleteProfile(id);
SshBridge.vmOpState();                         // {state:"idle"|"connecting"|"ok"|"error", message?, shellId?, host?}

// Shell lifecycle on the shared connection
SshBridge.vmNewShell(cwd, cmd);                // {ok, id, starting:true}; readiness via vmOpState()
SshBridge.vmCloseShell(id);
SshBridge.vmResizeShell(id, cols, rows);       // sync the remote PTY after an xterm fit
SshBridge.vmDisconnect();                      // drop the shared SSH connection

// Theme and host plumbing
SshBridge.vmListThemes();                      // [{id,name}]
SshBridge.vmSetTheme(id);                      // switch and refresh host chrome + VM

// Fire-and-forget host signals
SshBridge.vmSetBackHint(hint);                 // Back-key semantics for the host
SshBridge.vmSetShellBarRect(l, t, r, b);       // shell tab bar rect, avoids swipe conflicts
```

Differences from the host API: inside the VM, `openTerminalShell` / `openShellWith` only
switch the VM's own Terminal page, and `openPluginMarket` is a no-op because management is
built into the Wearable pages.

## 7. Errors

Failures return `{"ok":false,"error":"<code>"}`. Common codes:
`disconnected`, `shell-not-found`, `feed-not-found`, `feed-manager-unavailable`,
`invalid-spec`, `missing-path`, `missing-cmd`, `unsupported-feed-type`, `io`, `timeout`,
`pull-failed`, `remove-failed`, `plugin-manager-unavailable`, `profile-not-found`,
`missing-id`, `invalid-profile`, `missing-host`, `theme-not-found`.

## Not exposed

- Raw PTY byte streams / full scrollback;
- SSH internals (keys, session objects);
- Arbitrary local files or phone resources beyond the whitelist above.
