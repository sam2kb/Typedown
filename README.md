<p align="center">
  <img alt="Typedown Logo" src="./logo.png" width="100px" />
  <h1 align="center">Typedown</h1>
</p>

<p align="center">
  <a href="https://github.com/flintt/Typedown/releases/latest"><img alt="Upstream release" src="https://img.shields.io/github/v/release/flintt/Typedown?label=upstream%20release"></a>
  <a href="https://github.com/flintt/Typedown/releases"><img alt="Upstream downloads" src="https://img.shields.io/github/downloads/flintt/Typedown/total?label=upstream%20downloads"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/github/license/sam2kb/Typedown?label=license"></a>
</p>

Typedown is a WYSIWYG Markdown editor for Windows, with a WinUI interface and the Muya editor from [MarkText](https://github.com/marktext/marktext).

This fork builds on [flintt/Typedown](https://github.com/flintt/Typedown), which extends the original [byxiaozhi/Typedown](https://github.com/byxiaozhi/Typedown) with document tabs, folder-wide search, session restore, HedgeDoc sharing, reading mode, custom themes, and local automation for scripts and AI assistants.

The focus here is WebView2 hardening: keeping browser security protections enabled, checking the source of native messages, and serving local and HTTP images through restricted image endpoints. The editor uses a dedicated HTTPS origin while retaining image copying, exports, and printing. See the [editor protocol](docs/editor-protocol.md) for the resource and message boundaries.

## Downloads

Published builds from the flintt project are available in its [latest release](https://github.com/flintt/Typedown/releases/latest), for x64 and ARM64:

| File | Description |
|---|---|
| `Typedown-windows-x64-*.exe` / `-arm64-*.exe` | Windows installer |
| `Typedown-portable-x64-*.zip` / `-arm64-*.zip` | Portable version; extract and run, without registry registration |
| `Typedown_*_x64.msix` / `_ARM64.msix` | MSIX package; first install the matching `Typedown-signing-certificate.cer` from the same release in Trusted Root Certification Authorities |

Flintt also publishes **Typeleaf** in the Microsoft Store. The Store handles installation and updates without a manual certificate import.

<a href="https://apps.microsoft.com/detail/9nxbvhm4k005?referrer=appbadge&mode=direct">
  <img src="https://get.microsoft.com/images/en-us%20dark.svg" width="200" alt="Get Typeleaf from Microsoft Store"/>
</a>

For Linux and macOS, see the [flintt/Typedown-Uno port](https://github.com/flintt/Typedown-Uno/releases/latest).

The [original Typedown app in the Microsoft Store](https://apps.microsoft.com/detail/9p8tcw4h2hb4) belongs to byxiaozhi's project and has its own feature set.

## Themes and automation

Create a theme by adding a CSS file to the themes folder; see [custom themes](docs/custom-theme.md). Flintt's [theme designer](https://flintt.github.io/Typedown/) supports English and Chinese and lets you design, import, and export themes in a browser.

Local automation lets scripts and AI assistants read and edit open documents, switch view modes and sidebars, and change supported settings. Enable **Allow local automation** in **Settings > General**; it is off by default. Enabling it trusts programs running as your Windows user.

The installation includes `typedownctl.exe` for command-line access. Run `typedownctl mcp` to start the MCP server for AI tools. See [local automation](docs/automation.md) and [MCP](docs/automation-mcp.md) for usage and security boundaries.

## Screenshots

| Visual editing | Source mode |
|:--:|:--:|
| <img src="docs/media/visual.png" alt="Visual editing"> | <img src="docs/media/source.png" alt="Source mode"> |
| **Outline** | **Dark theme** |
| <img src="docs/media/outline.png" alt="Outline sidebar"> | <img src="docs/media/dark.png" alt="Dark theme"> |

These screenshots come from the flintt project. Its [screenshot tool](Tools/Screenshots) uses local automation to switch documents, languages, modes, sidebars, and themes in a running app.

## Building from source

Developer documentation: [architecture](docs/architecture.md), [WebView message protocol](docs/editor-protocol.md), [localization](docs/localization.md), [packaging and releases](PACKAGING.md), and [Windows verification](docs/windows-verification.md). Some linked documentation is in Chinese.

### 1. Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) on Windows 10 (1903 or later) or Windows 11
- Git for Windows
- [Node.js](https://nodejs.org/) 20 with [yarn](https://yarnpkg.com/)
- Optional: Visual Studio 2026 (or another editor) for working on the C# code; [Inno Setup 6](https://jrsoftware.org/isinfo.php) for the installer

The app is WinUI 3 (Windows App SDK, carried with the app) on .NET 10, for x64 and ARM64.

### 2. Clone the repository

```ps
git clone https://github.com/sam2kb/Typedown.git
cd Typedown
```

### 3. Build the editor

```ps
cd Dev\Typedown.Editor
yarn install --frozen-lockfile
node scripts\copy-mermaid.js
yarn build
```
The editor is built into `Dev\Typedown\Resources\Statics`, where the app loads it from.

### 4. Build and run the app

```ps
cd ..\..
dotnet build Dev\Typedown\Typedown.csproj -c Debug_Local -p:Platform=x64
Dev\Typedown\bin\x64\Debug_Local\net10.0-windows10.0.26100.0\win-x64\Typedown.exe
```
Configurations:

- Debug: the editor is loaded from http://localhost:3000 (run `yarn start` in `Dev\Typedown.Editor` as well)
- Debug_Local: the editor is loaded from the built bundle (`Dev\Typedown\Resources\Statics`)
- Release: optimized release build

The installer, the portable folder and the MSIX: see [PACKAGING.md](PACKAGING.md) (`Tools\Installer\build-local.ps1` builds the installer the way CI does).

## Contributing and reporting problems

Open an [issue in this fork](https://github.com/sam2kb/Typedown/issues) with the version from **Settings > About**, steps to reproduce the problem, and relevant logs from `%LOCALAPPDATA%\Typedown\logs\`.

For proposed changes, describe the intended behavior in an issue before opening a [pull request](https://github.com/sam2kb/Typedown/pulls). Discussions about flintt's builds belong in [flintt/Typedown](https://github.com/flintt/Typedown/issues); discussions about the original app belong in [byxiaozhi/Typedown](https://github.com/byxiaozhi/Typedown/issues).

## Credits

Thanks to [flintt](https://github.com/flintt) for maintaining and developing [flintt/Typedown](https://github.com/flintt/Typedown), the base for this fork. Its editor features, Windows app, themes, and local automation are inherited from that work.

The original Typedown was created by [byxiaozhi](https://github.com/byxiaozhi/Typedown), and its Muya editor comes from [MarkText](https://github.com/marktext/marktext). The WebView virtual-host approach was informed by [awarson2233's Typedown changes](https://github.com/awarson2233/Typedown/commit/c251ca54fcac5e890e440937c6498225a920cd60).

## License

MIT, under the upstream [license](LICENSE). The Muya editor comes from [MarkText](https://github.com/marktext/marktext) and is also MIT-licensed.
