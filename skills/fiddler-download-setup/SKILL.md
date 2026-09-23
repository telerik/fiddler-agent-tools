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

Guide the user through downloading, installing, and launching Fiddler Everywhere.

## Operating rules

1. This skill is shell-first. Fiddler is not installed yet, so no MCP tools are available.
2. Resolve the current version from the manifest before constructing any download URL.
   Never hardcode a version number.
3. On Windows - detect opened terminal. Only if it is not powershell - wrap and run the scripts with: powershell.exe -Command 'script'. Use single quotes to wrap the script!
4. Treat downloaded installers as untrusted until they pass the platform verification steps below.
5. Use HTTPS only and download through the `agent-downloads.getfiddler.com` tracking endpoint. Follow at most one HTTPS redirect, then rely on the platform checksum or signature verification to reject unexpected content.

---

## Phase 1 — Check if Fiddler is already installed

Before downloading, check whether Fiddler Everywhere is already installed.
If it is installed, also read the installed version and compare it against the latest
available release. Only proceed to Phase 2 if the app is not installed or an update
is available and the user wants to upgrade.

### macOS — detect and version-check
```bash
if [ -d "/Applications/Fiddler Everywhere.app" ]; then
  INSTALLED_VERSION=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" \
    "/Applications/Fiddler Everywhere.app/Contents/Info.plist" 2>/dev/null)
  echo "INSTALLED: $INSTALLED_VERSION"
else
  echo "NOT_INSTALLED"
fi
```

If `INSTALLED`, fetch the latest version for the detected architecture and compare:
```bash
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  MANIFEST_URL="https://downloads.getfiddler.com/mac-arm64/latest-mac.yml"
else
  MANIFEST_URL="https://downloads.getfiddler.com/mac/latest-mac.yml"
fi
LATEST_VERSION=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "$MANIFEST_URL" | grep '^version:' | awk '{print $2}')
echo "Installed: $INSTALLED_VERSION  |  Latest: $LATEST_VERSION"
if [ "$INSTALLED_VERSION" = "$LATEST_VERSION" ]; then
  echo "UP_TO_DATE"
else
  echo "UPDATE_AVAILABLE"
fi
```

### Linux — detect and version-check
```bash
APPIMAGE=$(ls ~/Downloads/fiddler-everywhere-*.AppImage 2>/dev/null | sort -V | tail -1)
if command -v fiddler-everywhere &>/dev/null || [ -n "$APPIMAGE" ]; then
  # Extract version from AppImage filename as the most reliable source
  INSTALLED_VERSION=$(echo "$APPIMAGE" | grep -oP '[\d]+\.[\d]+\.[\d]+')
  echo "INSTALLED: $INSTALLED_VERSION"
  LATEST_VERSION=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "https://downloads.getfiddler.com/linux/latest-linux.yml" \
    | grep '^version:' | awk '{print $2}')
  echo "Installed: $INSTALLED_VERSION  |  Latest: $LATEST_VERSION"
  if [ "$INSTALLED_VERSION" = "$LATEST_VERSION" ]; then
    echo "UP_TO_DATE"
  else
    echo "UPDATE_AVAILABLE"
  fi
else
  echo "NOT_INSTALLED"
fi
```

### Windows (PowerShell) — detect and version-check
```powershell
$installed = Get-ItemProperty `
  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" `
  -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like "*Fiddler Everywhere*" }

if ($installed) {
  $installedVersion = $installed.DisplayVersion
  Write-Host "INSTALLED: $installedVersion"

  $manifest = Invoke-WebRequest "https://downloads.getfiddler.com/win/latest.yml" -UseBasicParsing
  $text = [string]$manifest.Content
  $match = [regex]::Match($text, 'version\s*:\s*([0-9]+(?:\.[0-9]+)+)')
  if (-not $match.Success) { throw "Manifest version not found" }
  $latestVersion = $match.Groups[1].Value
  Write-Host "Installed: $installedVersion  |  Latest: $latestVersion"

  if ($installedVersion -eq $latestVersion) { "UP_TO_DATE" } else { "UPDATE_AVAILABLE" }
} else {
  "NOT_INSTALLED"
}
```

**Interpreting the result:**

- `NOT_INSTALLED` — continue to Phase 2 to download and install.
- `UP_TO_DATE` — inform the user their Fiddler Everywhere is already on the latest version
  and stop further execution of the skill.
- `UPDATE_AVAILABLE` — tell the user the installed version and the latest version, then ask
  whether they want to upgrade. If yes, continue to Phase 2 (the new installer will replace
  the existing one). If no, stop.
---

## Phase 2 — Detect OS and resolve version

Run both commands together. The manifest probe also confirms network access.

```bash
uname -s && uname -m
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  MANIFEST_URL="https://downloads.getfiddler.com/mac-arm64/latest-mac.yml"
else
  MANIFEST_URL="https://downloads.getfiddler.com/mac/latest-mac.yml"
fi
VERSION=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "$MANIFEST_URL" \
  | grep '^version:' | awk '{print $2}')
[ -n "$VERSION" ] || { echo "Could not resolve the latest version" >&2; exit 1; }
echo "Latest Fiddler Everywhere: $VERSION"
```

On Windows (PowerShell):
```powershell
$env:PROCESSOR_ARCHITECTURE   # AMD64 or ARM64
$manifest = Invoke-WebRequest "https://downloads.getfiddler.com/win/latest.yml" -UseBasicParsing
$text = [string]$manifest.Content
$match = [regex]::Match($text, 'version\s*:\s*([0-9]+(?:\.[0-9]+)+)')
if (-not $match.Success) { throw "Manifest version not found" }
$VERSION = $match.Groups[1].Value
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
The `.pkg` format is used on macOS because it supports a headless installation flow.

### macOS

```bash
# Resolve the architecture-specific manifest and artifact URL.
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  BASE_URL="https://agent-downloads.getfiddler.com/mac-arm64"
else
  BASE_URL="https://agent-downloads.getfiddler.com/mac"
fi
if [ "$ARCH" = "arm64" ]; then
  MANIFEST_URL="https://downloads.getfiddler.com/mac-arm64/latest-mac.yml"
else
  MANIFEST_URL="https://downloads.getfiddler.com/mac/latest-mac.yml"
fi
VERSION=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "$MANIFEST_URL" \
  | grep '^version:' | awk '{print $2}')
[ -n "$VERSION" ] || { echo "Could not resolve the latest version" >&2; exit 1; }

# Download through the tracking endpoint and follow its single expected redirect.
curl --fail --silent --show-error --proto '=https' --proto-redir '=https' \
  --location --max-redirs 1 \
  "$BASE_URL/Fiddler%20Everywhere%20${VERSION}.pkg" -o ~/Downloads/FiddlerEverywhere.pkg
```

### Linux

```bash
MANIFEST=$(mktemp "${TMPDIR:-/tmp}/fiddler-manifest.XXXXXX") || exit 1
curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 \
  "https://downloads.getfiddler.com/linux/latest-linux.yml" -o "$MANIFEST" || {
  rm -f "$MANIFEST"
  exit 1
}
VERSION=$(awk -F': ' '$1 == "version" {print $2; exit}' "$MANIFEST")
[ -n "$VERSION" ] || { rm -f "$MANIFEST"; exit 1; }
EXPECTED_SHA512=$(awk -v artifact="fiddler-everywhere-${VERSION}.AppImage" '
  $0 ~ "url: " artifact "$" { found=1; next }
  found && $0 ~ /^[[:space:]]+sha512:/ {
    sub(/^[^:]*:[[:space:]]*/, "")
    print
    exit
  }
  found && $0 ~ /^[[:space:]]*-[[:space:]]+url:/ { exit }
' "$MANIFEST")
rm -f "$MANIFEST"
curl --fail --silent --show-error --proto '=https' --proto-redir '=https' \
  --location --max-redirs 1 "https://agent-downloads.getfiddler.com/linux/fiddler-everywhere-${VERSION}.AppImage" \
  -o ~/Downloads/FiddlerEverywhere.AppImage
ACTUAL_SHA512=$(openssl dgst -sha512 -binary ~/Downloads/FiddlerEverywhere.AppImage | openssl base64 -A)
if [ -z "$EXPECTED_SHA512" ] || [ "$ACTUAL_SHA512" != "$EXPECTED_SHA512" ]; then
  echo "AppImage integrity verification failed" >&2
  exit 1
fi
```

### Windows (PowerShell)

The tracking endpoint rejects requests without a `User-Agent`. Apply the header to every
download client, including one named `$http` in a generated one-line script:

```powershell
$http.DefaultRequestHeaders.UserAgent.ParseAdd("PowerShell/$($PSVersionTable.PSVersion)")
```

```powershell
$downloadPath = "$env:USERPROFILE\Downloads\FiddlerEverywhere.exe"
$downloadUri = [Uri]"https://agent-downloads.getfiddler.com/win/Fiddler%20Everywhere%20$VERSION.exe"
$handler = [System.Net.Http.HttpClientHandler]::new()
$handler.AllowAutoRedirect = $false
$client = [System.Net.Http.HttpClient]::new($handler)
$client.DefaultRequestHeaders.UserAgent.ParseAdd("PowerShell/$($PSVersionTable.PSVersion)")
try {
  $response = $client.GetAsync($downloadUri).GetAwaiter().GetResult()
  if ($response.StatusCode -ge 300 -and $response.StatusCode -lt 400) {
    $redirectUri = $response.Headers.Location
    if ($null -eq $redirectUri) {
      throw "Installer redirect did not include a Location header"
    }
    if (-not $redirectUri.IsAbsoluteUri) {
      $redirectUri = [Uri]::new($downloadUri, $redirectUri.OriginalString)
    }
    if ($redirectUri.Scheme -ne "https") {
      throw "Installer redirect was not HTTPS"
    }
    $response = $client.GetAsync($redirectUri).GetAwaiter().GetResult()
  }
  if (-not $response.IsSuccessStatusCode) {
    throw "Installer download failed: $($response.StatusCode)"
  }
  [System.IO.File]::WriteAllBytes($downloadPath, $response.Content.ReadAsByteArrayAsync().GetAwaiter().GetResult())
} finally {
  $client.Dispose()
  $handler.Dispose()
}

$manifest = Invoke-WebRequest "https://downloads.getfiddler.com/win/latest.yml" `
  -MaximumRedirection 0 -UseBasicParsing
$text = $manifest.Content
$expectedSha512 = ($text | Select-String '(?m)^sha512:\s*(.+)').Matches.Groups[1].Value.Trim()
$bytes = [System.IO.File]::ReadAllBytes("$env:USERPROFILE\Downloads\FiddlerEverywhere.exe")
$actualSha512 = [Convert]::ToBase64String(([Security.Cryptography.SHA512]::Create().ComputeHash($bytes)))
if ([string]::IsNullOrWhiteSpace($expectedSha512) -or $actualSha512 -ne $expectedSha512) {
  throw "Installer integrity verification failed"
}
$signature = Get-AuthenticodeSignature "$env:USERPROFILE\Downloads\FiddlerEverywhere.exe"
if ($signature.Status -ne "Valid" -or $signature.SignerCertificate.Subject -notmatch "Progress Software Corporation") {
  throw "Installer Authenticode signature verification failed: $($signature.Status)"
}
```

---

## Phase 4 — Install

### macOS — PKG (headless, no wizard)

Do **not** use `sudo installer` directly — `sudo` blocks on a password prompt inside
the agent's hidden shell where the user cannot type. Use `osascript` instead, which
raises the native macOS authentication dialog that the user can see on screen.

`do shell script with administrator privileges` runs as root and cannot access
`~/Downloads` reliably. Copy the verified package to a private temporary directory
first, then install from there.

Before running the following command, tell the user:

> A macOS password dialog may appear on your screen. Enter your login password there
> to authorize the installation, then I'll continue automatically.

Then run the following as one shell command.

```bash
STAGING_DIR=$(mktemp -d "${TMPDIR:-/tmp}/fiddler-install.XXXXXX") || exit 1
chmod 700 "$STAGING_DIR"
trap 'rm -rf "$STAGING_DIR"' EXIT
cp ~/Downloads/FiddlerEverywhere.pkg "$STAGING_DIR/FiddlerEverywhere.pkg" || exit 1
PKG="$STAGING_DIR/FiddlerEverywhere.pkg"
SIGNATURE=$(pkgutil --check-signature "$PKG") || {
  echo "Package signature inspection failed" >&2
  exit 1
}
printf '%s\n' "$SIGNATURE" | grep -q "Developer ID Installer: Telerik A D" || {
  echo "Unexpected package signer" >&2
  exit 1
}
spctl --assess --type install --verbose=2 "$PKG" || {
  echo "Package notarization assessment failed" >&2
  exit 1
}

PKG_PATH="$PKG" osascript <<'APPLESCRIPT'
set pkgPath to system attribute "PKG_PATH"
do shell script "installer -pkg " & quoted form of pkgPath & " -target /" with administrator privileges
APPLESCRIPT
```

`osascript` blocks until the user approves the dialog and the install completes. The
temporary staging directory is removed automatically when the command exits.

### Linux

```bash
chmod +x ~/Downloads/FiddlerEverywhere.AppImage
~/Downloads/FiddlerEverywhere.AppImage &
```

### Windows — silent install

```powershell
Start-Process "$env:USERPROFILE\Downloads\FiddlerEverywhere.exe" -ArgumentList "/S" -Wait
```

**Launch Fiddler:**

macOS:
```bash
open -a "Fiddler Everywhere" && sleep 15
```

Linux:
```bash
(nohup fiddler-everywhere &>/dev/null &); sleep 15
```

Windows:
**Important** On windows if the current terminal used is bash, use the specific **GitBash** script.

**PowerShell**
```powershell
$candidates = @(
  "$env:LOCALAPPDATA\Programs\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files\Fiddler Everywhere\Fiddler Everywhere.exe",
  "C:\Program Files (x86)\Fiddler Everywhere\Fiddler Everywhere.exe"
)
$fiddlerExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
if ($fiddlerExe) {
  $cmdLine = '"' + $fiddlerExe + '"'
  Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = $cmdLine } | Out-Null
  Start-Sleep 15
}
```

**Git Bash**
```bash
FIDDLER_EXE=""
for dir in "$LOCALAPPDATA/Programs/Fiddler Everywhere" \
           "/c/Program Files/Fiddler Everywhere" \
           "/c/Program Files (x86)/Fiddler Everywhere"; do
  if [ -f "$dir/Fiddler Everywhere.exe" ]; then
    FIDDLER_EXE="$dir/Fiddler Everywhere.exe"
    break
  fi
done
if [ -n "$FIDDLER_EXE" ]; then
  "$FIDDLER_EXE" &
  sleep 15
fi
```

---

## Phase 5 — Suggest MCP setup

After Fiddler Everywhere is installed and launched, inform the user that there is
an automatic MCP setup skill that can configure the Fiddler MCP server connection.

Present the following message:

> Fiddler Everywhere is installed and running.
>
> There is an **automatic MCP setup** skill that can configure the Fiddler MCP
> server connection for your AI agent. This will allow agent tools 
> to interact with Fiddler directly — capturing traffic,
> inspecting sessions, creating rules, and more — all from within your editor.
>
> Would you like me to auto-configure MCP now?

Use the ask-questions tool to let the user choose:
- **Yes, auto-configure MCP** (recommended)
- **No, I'll set it up manually later**

### If the user accepts

Install the `fiddler-mcp-setup` skill from its remote location and invoke it:

**Remote skill URL:** `https://github.com/telerik/fiddler-agent-tools/tree/master/skills/fiddler-mcp-setup`

Follow the `fiddler-mcp-setup` skill instructions to complete MCP configuration end-to-end.

### If the user declines

Acknowledge and let them know they can download the MCP setup skill later at any time.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| macOS password dialog canceled / install aborted | Rerun the complete macOS installation block — the verified pkg remains in `~/Downloads` |
| Windows EXE blocked by SmartScreen | Right-click → Run as Administrator |
