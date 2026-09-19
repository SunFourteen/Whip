# Whip

**版本：** `v0.9.0`

一款内置插件系统的 Android SSH 客户端。

## 功能

- Android SSH 客户端
- 可扩展的插件系统
- 支持远端与本地插件源
- 在 App 内即可完成插件管理

## 下载

本仓库包含以下两部分：

### APK

下载最新版本：

📦 [whip-0.9.0.apk](https://github.com/SunFourteen/whip/releases/download/v0.9.0/whip-0.9.0.apk)

### 插件源（tap）

默认插件源地址：

```text
https://raw.githubusercontent.com/SunFourteen/whip/v0.9.0/index.json
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
