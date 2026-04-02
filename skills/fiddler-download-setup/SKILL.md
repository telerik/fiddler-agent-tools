---
name: fiddler-download-setup
description: >
  Download, install, and launch Fiddler Everywhere from scratch, then configure MCP for
  AI agent tools. Use this skill when the user does not have Fiddler installed, says
  "download Fiddler", "install Fiddler Everywhere", "get started with Fiddler", "first
  time Fiddler setup", "set up Fiddler from scratch", or needs a complete onboarding
  from zero to a working Fiddler MCP connection.
---

# Fiddler Download & Setup

Guide the user through downloading, installing, and launching Fiddler Everywhere — then
automatically chain into MCP configuration so the agent can use Fiddler tools.

## Operating rules

1. This skill is shell-first. Fiddler is not installed yet, so no MCP tools are available.
2. Resolve the current version from the manifest before constructing any download URL.
   Never hardcode a version number.

---

## Phase 1 — Check if Fiddler is already installed

Before downloading, check whether Fiddler Everywhere is already installed.

### macOS
```bash
if [ -d "/Applications/Fiddler Everywhere.app" ]; then
  echo "INSTALLED"
else
  echo "NOT_INSTALLED"
fi
```

### Linux
```bash
if command -v fiddler-everywhere &>/dev/null || ls ~/Downloads/FiddlerEverywhere.AppImage &>/dev/null; then
  echo "INSTALLED"
else
  echo "NOT_INSTALLED"
fi
```

### Windows (PowerShell)
```powershell
$installed = Get-ItemProperty `
  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" `
  -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like "*Fiddler Everywhere*" }

if ($installed) { "INSTALLED" } else { "NOT_INSTALLED" }
```

If the output is `INSTALLED`, inform the user that Fiddler Everywhere is already
installed and stop further execution of the skill.
---

## Phase 2 — Detect OS and resolve version

Run both commands together. The manifest probe also confirms network access.

```bash
uname -s && uname -m
VERSION=$(curl -s "https://downloads.getfiddler.com/mac-arm64/latest-mac.yml" \
  | grep '^version:' | awk '{print $2}')
echo "Latest Fiddler Everywhere: $VERSION"
```

On Windows (PowerShell):
```powershell
$env:PROCESSOR_ARCHITECTURE   # AMD64 or ARM64
$manifest = Invoke-WebRequest "https://downloads.getfiddler.com/win/latest.yml" -UseBasicParsing
$text = [System.Text.Encoding]::UTF8.GetString($manifest.RawContentStream.ToArray())
$VERSION = ($text | Select-String '(?m)^version:\s*(.+)').Matches.Groups[1].Value.Trim()
Write-Host "Latest Fiddler Everywhere: $VERSION"
```

| `uname -s` | `uname -m` | Platform |
|-------------|------------|----------|
| `Darwin` | `arm64` | macOS Apple Silicon |
| `Darwin` | `x86_64` | macOS Intel |
| `Linux` | anything | Linux |
| — | — | Windows — use PowerShell path above |

---

## Phase 3 — Download

Use `$VERSION` resolved in Phase 2 to construct a direct, versioned URL.
The `.pkg` format is preferred on macOS.

### macOS

```bash
# Apple Silicon (arm64)
curl -L \
  "https://downloads.getfiddler.com/mac-arm64/Fiddler%20Everywhere%20${VERSION}.pkg" \
  -o ~/Downloads/FiddlerEverywhere.pkg

# Intel (x86_64)
curl -L \
  "https://downloads.getfiddler.com/mac/Fiddler%20Everywhere%20${VERSION}.pkg" \
  -o ~/Downloads/FiddlerEverywhere.pkg
```

### Linux

```bash
VERSION=$(curl -s "https://downloads.getfiddler.com/linux/latest-linux.yml" \
  | grep '^version:' | awk '{print $2}')
curl -L \
  "https://downloads.getfiddler.com/linux/fiddler-everywhere-${VERSION}.AppImage" \
  -o ~/Downloads/FiddlerEverywhere.AppImage
```

### Windows (PowerShell)

```powershell
Invoke-WebRequest `
  "https://downloads.getfiddler.com/win/Fiddler%20Everywhere%20$VERSION.exe" `
  -OutFile "$env:USERPROFILE\Downloads\FiddlerEverywhere.exe"
```

---

## Phase 4 — Install

### macOS — PKG (headless, no wizard)

Do **not** use `sudo installer` directly — `sudo` blocks on a password prompt inside
the agent's hidden shell where the user cannot type. Use `osascript` instead, which
raises the native macOS authentication dialog that the user can see on screen.

`do shell script with administrator privileges` runs as root and cannot access
`~/Downloads` due to macOS sandbox restrictions. Copy the package to `/tmp` first
(world-readable), then install from there.

Run both commands via bash tool calls:

```bash
cp ~/Downloads/FiddlerEverywhere.pkg /tmp/FiddlerEverywhere.pkg
```

Then announce before the next command:
> "A macOS password dialog will appear on your screen. Enter your login password there
> to authorize the installation, then I'll continue automatically."

```bash
osascript -e 'do shell script "installer -pkg /tmp/FiddlerEverywhere.pkg -target /" with administrator privileges'
```

`osascript` blocks until the user approves the dialog and the install completes. Once it
exits with code 0, clean up.

```bash
rm /tmp/FiddlerEverywhere.pkg
```

### macOS — DMG (if user prefers drag-and-drop)

```bash
hdiutil attach ~/Downloads/FiddlerEverywhere.dmg -nobrowse
cp -R "/Volumes/Fiddler Everywhere/Fiddler Everywhere.app" /Applications/
hdiutil detach "/Volumes/Fiddler Everywhere"
```

### Linux

```bash
chmod +x ~/Downloads/FiddlerEverywhere.AppImage
~/Downloads/FiddlerEverywhere.AppImage &
```

### Windows — silent install

```powershell
Start-Process "$env:USERPROFILE\Downloads\FiddlerEverywhere.exe" -ArgumentList "/S" -Wait
```

macOS launch:
```bash
open /Applications/Fiddler\ Everywhere.app
```

Windows launch:
```powershell
$candidates = @(
  "$env:LOCALAPPDATA\Programs\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files (x86)\Fiddler Everywhere\Fiddler Everywhere.exe"
)
$fiddlerExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
if ($fiddlerExe) {
  Start-Process $fiddlerExe
  Write-Host "Launched: $fiddlerExe"
} else {
  Write-Error "Could not find Fiddler Everywhere exe. Check installation completed successfully."
}
```
---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| macOS password dialog canceled / install aborted | Rerun the `osascript` command — the pkg in `/tmp` is still there |
| `hdiutil: attach failed` | File may not be fully downloaded — rerun the `curl` command |
| Windows EXE blocked by SmartScreen | Right-click → Run as Administrator |
| App won't open on macOS | Run `xattr -cr "/Applications/Fiddler Everywhere.app"` to clear quarantine |
