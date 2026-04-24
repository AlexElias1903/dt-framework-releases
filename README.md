# DT Framework — Releases

Installers and auto-update metadata for the **DT Framework**, a Digital Twin platform for industrial IoT research built at PUCRS.

This repository only hosts binary release artifacts (`.dmg`, `.exe`, `.AppImage`, `.deb`, `latest-*.yml`, etc.). Its purpose is to make the published installers reachable to the Electron auto-updater so existing installations can detect and download new versions.

## Downloads

Grab the installer for your platform from the [latest release](../../releases/latest).

| Platform | File |
|---|---|
| macOS (Apple Silicon) | `dt-framework-<version>-mac-arm64.dmg` |
| Windows (x64) | `dt-framework-<version>-win-x64.exe` |
| Linux (x64) | `dt-framework-<version>-linux-x64.AppImage` / `.deb` |

## Auto-update

`electron-updater` polls `latest-mac.yml` / `latest.yml` / `latest-linux.yml` attached to each release and prompts the user when a newer version is available.
