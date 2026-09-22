# Whip

**版本：** `v0.10.0`

一款内置插件系统的 Android SSH 客户端。

## 功能

- Android SSH 客户端：多主机、单主机多会话、tmux 会话
- 插件跑在所连接的目标机上（Linux / macOS / Windows），界面跑在手机上
- 每条连接只有一个长驻命令通道，插件页面即时响应，不再为每条命令重开一个 shell
- 插件既可从 tap（静态 `index.json` + 包）安装，也可以只放在自己的目标机上
- App 内即可完成插件管理：安装、更新、与目标机同步、删除
- 插件状态按目标机隔离，同一个插件跟随你当前连接的那台机器

## 下载

本仓库包含以下两部分：

### APK

下载最新版本：

📦 [whip-0.10.0.apk](https://github.com/SunFourteen/whip/releases/download/v0.10.0/whip-0.10.0.apk)

### 插件源（tap）

默认插件源地址：

```text
https://raw.githubusercontent.com/SunFourteen/whip/v0.10.0/index.json
```

在 Whip 里添加插件源：

1. 打开 **Plugins → Manager**；
2. 把上面的地址填进插件源输入框；
3. 点 **Refresh** 拉取可用插件；
4. 选中插件后点 **Install**。

## 开发自己的插件

你可以开发插件并自行托管。

### 1. 构建插件

插件清单格式、`SshBridge` API 与打包方式见 [PLUGIN_API.md](PLUGIN_API.md)。

### 2. 托管插件

把插件目录放到目标机上；每个插件是独立目录，内含 `manifest.json`：

```text
~/.whip/plugins/your-plugin/
├── manifest.json
└── ...
```

### 3. 分发多个插件

把它们打包成 tap（静态的 `index.json` + `packages/<id>-<version>.tar.gz`），托管到任意
能提供静态文件的地方，再在 **Plugins → Manager** 里填入该地址，即可安装与更新。

---

## 许可

许可信息见仓库。

English: [README.md](README.md)
