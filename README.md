# Whip

**Version:** `v0.10.0`

An Android SSH client with a built-in plugin system.

## Features

- SSH client for Android: several hosts, several shells per host, tmux sessions
- Plugins run on the connected host — Linux, macOS or Windows — with their UI on the phone
- One long-lived command channel per connection, so plugin pages answer immediately instead
  of paying a shell start-up per command
- Plugins install from a tap (a static `index.json` plus packages) or from a host you keep
  yourself
- In-app plugin manager: install, update, sync with the host, remove
- Plugin state is kept per host, so the same plugin follows the machine you are connected to

## Downloads

This repository contains the following components:

### APK

Download the latest release:

📦 [whip-0.10.0.apk](https://github.com/SunFourteen/whip/releases/download/v0.10.0/whip-0.10.0.apk)

### Plugin Registry

The default plugin registry is available at:

```text
https://raw.githubusercontent.com/SunFourteen/whip/v0.10.0/index.json
```

To add the registry to Whip:

1. Open **Plugins → Manager**.
2. Copy the registry URL above into the plugin source field.
3. Click **Refresh** to load the available plugins.
4. Select a plugin and click **Install**.

## Creating Your Own Plugins

You can create and host your own plugins for Whip.

### 1. Build a Plugin

Refer to [PLUGIN_API.md](PLUGIN_API.md) for the plugin manifest, the `SshBridge` API and
how plugins are packaged.

### 2. Host Your Plugin

Put the plugin directory on the target host; each plugin is its own directory with a
`manifest.json`:

```text
~/.whip/plugins/your-plugin/
├── manifest.json
└── ...
```

### 3. Share Several Plugins

Package them as a tap — a static `index.json` plus `packages/<id>-<version>.tar.gz` —
and host it anywhere that serves static files. Paste that URL in
**Plugins → Manager** to install and update from it.

---

## License

See the repository for license information.
