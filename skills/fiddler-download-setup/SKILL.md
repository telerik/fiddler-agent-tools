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
6. Diagnose failures from actual output. If output is truncated, inspect its saved file and use exact paths printed by the script; do not recursively search temporary directories. Do not add `sudo`, change mountpoints, re-download, use verbose mode, or kill processes without evidence.
7. Never accept a license without the user's explicit informed consent. For the observed `hdiutil` EULA prompt, only after the user reviews and accepts may the agent send one `yes` response; never use unbounded `yes` or bypass the prompt.

---

## Phase 1 — Check if Fiddler is already installed

Before downloading, check whether Fiddler Everywhere is already installed.
If it is installed, also read the installed version and compare it against the latest
available release. Only proceed to Phase 2 if the app is not installed or an update
is available and the user wants to upgrade.

```bash
### macOS — detect and version-check
APP_PATH="$HOME/Applications/Fiddler Everywhere.app"
if [ ! -d "$APP_PATH" ]; then
  APP_PATH="/Applications/Fiddler Everywhere.app"
fi
if [ -d "$APP_PATH" ]; then
  INSTALLED_VERSION=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" \
    "$APP_PATH/Contents/Info.plist" 2>/dev/null)
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
MANIFEST_TEXT=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "$MANIFEST_URL") || {
  echo "Failed to fetch version manifest: $MANIFEST_URL" >&2
  exit 1
}
LATEST_VERSION=$(printf '%s\n' "$MANIFEST_TEXT" |
  awk '
    /^version:[[:space:]]*/ {
      sub(/^version:[[:space:]]*/, "")
      sub(/[[:space:]]+$/, "")
      print
      exit
    }
  ')
if [ -z "$LATEST_VERSION" ]; then
  echo "Version field missing from manifest: $MANIFEST_URL" >&2
  exit 1
fi
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
  INSTALLED_VERSION=${APPIMAGE##*/}
  INSTALLED_VERSION=${INSTALLED_VERSION#fiddler-everywhere-}
  INSTALLED_VERSION=${INSTALLED_VERSION%.AppImage}
  echo "INSTALLED: $INSTALLED_VERSION"
  MANIFEST_URL="https://downloads.getfiddler.com/linux/latest-linux.yml"
  MANIFEST_TEXT=$(curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 "$MANIFEST_URL") || {
    echo "Failed to fetch version manifest: $MANIFEST_URL" >&2
    exit 1
  }
  LATEST_VERSION=$(printf '%s\n' "$MANIFEST_TEXT" |
    awk '
      /^version:[[:space:]]*/ {
        sub(/^version:[[:space:]]*/, "")
        sub(/[[:space:]]+$/, "")
        print
        exit
      }
    ')
  if [ -z "$LATEST_VERSION" ]; then
    echo "Version field missing from manifest: $MANIFEST_URL" >&2
    exit 1
  fi
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
  $content = $manifest.Content
  if ($content -is [byte[]]) {
    $text = [System.Text.Encoding]::UTF8.GetString($content)
  } else {
    $text = [string]$content
  }
  $text = $text.TrimStart([char]0xFEFF)
  $versionLine = $null
  foreach ($line in $text.Split([char]10)) {
    $line = $line.Trim()
    if ($line.StartsWith("version:")) {
      $versionLine = $line
      break
    }
  }
  if ($null -eq $versionLine) { throw "Manifest version line not found" }
  $latestVersion = $versionLine.Substring(8).Trim()
  try { [void][version]::Parse($latestVersion) } catch { throw "Manifest version is malformed: $latestVersion" }
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

## Phase 2 — Select platform

Use the runtime OS to select the Phase 3 platform block. Each block resolves its release version and checksum from one manifest download; do not run a separate version or architecture probe.

---

## Phase 3 — Download, verify, and install

Each platform reads the release manifest and verifies the checksum for the exact
artifact filename before execution. macOS and Windows also check the vendor signature.
The macOS install, Linux launch, and Windows install run in the same command as the
download so each uses the artifact that was verified.

### macOS

Before running, tell the user:
- The execution environment may request `/Volumes` access to attach the DMG; this is separate from administrator access and EULA consent.
- If attach output contains `End User License Agreement` and mounting is canceled, inspect that entire section in `hdiutil-attach.log`, through the `Rev.` line. Cancellation may appear as `hdiutil: attach canceled`, `Error 112`, or `user canceled`. Summarize key terms and provide the log path; do not reproduce the EULA verbatim.
- If the user accepts, set `EULA_RETRY_DIR` to the retained path printed by the script and `EULA_ACCEPTED=1`, then rerun this block. It reuses the manifest and DMG and rechecks the checksum without downloading them again.
- If the user declines, stop and remove the retained staging directory.

The DMG is mounted read-only at the private temporary mountpoint.
Any attach failure retains the staging directory and log for diagnosis. Remove the
retained directory after diagnosis when it is no longer needed.

```bash
set -e
set -o pipefail

TMP_ROOT="${TMPDIR:-/tmp}"
TMP_ROOT="${TMP_ROOT%/}"
EULA_RETRY_DIR="${EULA_RETRY_DIR:-}"
EULA_ACCEPTED="${EULA_ACCEPTED:-0}"
PRESERVE_STAGING=0
if [ "$EULA_ACCEPTED" != "0" ] && [ "$EULA_ACCEPTED" != "1" ]; then
  echo "EULA_ACCEPTED must be 0 or 1" >&2
  exit 1
fi
if [ "$EULA_ACCEPTED" = "1" ] && [ -z "$EULA_RETRY_DIR" ]; then
  echo "EULA_ACCEPTED=1 requires the retained EULA_RETRY_DIR" >&2
  exit 1
fi
if [ -n "$EULA_RETRY_DIR" ]; then
  case "$EULA_RETRY_DIR" in
    "$TMP_ROOT"/fiddler-install.*) ;;
    *) echo "EULA_RETRY_DIR is not a Fiddler staging directory under the temp root" >&2; exit 1 ;;
  esac
  if [ ! -d "$EULA_RETRY_DIR" ] || [ -L "$EULA_RETRY_DIR" ] || \
    [ "$(stat -f '%u' "$EULA_RETRY_DIR" 2>/dev/null)" != "$(id -u)" ]; then
    echo "EULA_RETRY_DIR is missing, unsafe, or not owned by the current user" >&2
    exit 1
  fi
  STAGING_DIR="$EULA_RETRY_DIR"
else
  STAGING_DIR=$(mktemp -d "$TMP_ROOT/fiddler-install.XXXXXX") || exit 1
fi
chmod 700 "$STAGING_DIR"
MOUNT_DIR=""
INSTALL_DIR=""
cleanup() {
  if [ -n "$MOUNT_DIR" ]; then
    hdiutil detach "$MOUNT_DIR" >/dev/null 2>&1 || true
  fi
  if [ -n "$INSTALL_DIR" ]; then
    rm -rf "$INSTALL_DIR"
  fi
  if [ "$PRESERVE_STAGING" = "1" ]; then
    printf 'Retained installer staging directory: %s\n' "$STAGING_DIR"
  else
    rm -rf "$STAGING_DIR"
  fi
}
trap cleanup EXIT

ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  BASE_URL="https://agent-downloads.getfiddler.com/mac-arm64"
  MANIFEST_URL="https://downloads.getfiddler.com/mac-arm64/latest-mac.yml"
else
  BASE_URL="https://agent-downloads.getfiddler.com/mac"
  MANIFEST_URL="https://downloads.getfiddler.com/mac/latest-mac.yml"
fi
MANIFEST="$STAGING_DIR/latest-mac.yml"
if [ -z "$EULA_RETRY_DIR" ]; then
  curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 \
    "$MANIFEST_URL" -o "$MANIFEST"
elif [ ! -f "$MANIFEST" ] || [ -L "$MANIFEST" ]; then
  echo "Retained release manifest is missing or unsafe" >&2
  exit 1
fi
VERSION=$(awk -F': ' '$1 == "version" { print $2; exit }' "$MANIFEST")
if ! printf '%s\n' "$VERSION" | grep -Eq '^[0-9]+(\.[0-9]+)+$'; then
  echo "Manifest version is missing or malformed" >&2
  exit 1
fi
echo "Latest Fiddler Everywhere: $VERSION"
ARTIFACT_NAME="Fiddler Everywhere ${VERSION}.dmg"
EXPECTED_SHA512=$(awk -v artifact="$ARTIFACT_NAME" '
  /^[[:space:]]*-[[:space:]]+url:/ {
    url = $0
    sub(/^[[:space:]]*-[[:space:]]+url:[[:space:]]*/, "", url)
    sub(/[[:space:]]*$/, "", url)
    found = (url == artifact)
    next
  }
  found && /^[[:space:]]+sha512:[[:space:]]*/ {
    digest = $0
    sub(/^[[:space:]]+sha512:[[:space:]]*/, "", digest)
    sub(/[[:space:]]*$/, "", digest)
    print digest
    exit
  }
' "$MANIFEST")
if ! printf '%s\n' "$EXPECTED_SHA512" | grep -Eq '^[A-Za-z0-9+/]{86}==$'; then
  echo "Missing or malformed SHA-512 for $ARTIFACT_NAME" >&2
  exit 1
fi
printf '%s' "$EXPECTED_SHA512" | openssl base64 -d -A > "$STAGING_DIR/expected.sha512"
if [ "$(wc -c < "$STAGING_DIR/expected.sha512" | tr -d '[:space:]')" != "64" ] || \
  [ "$(openssl base64 -A -in "$STAGING_DIR/expected.sha512")" != "$EXPECTED_SHA512" ]; then
  echo "Malformed SHA-512 for $ARTIFACT_NAME" >&2
  exit 1
fi

DMG="$STAGING_DIR/$ARTIFACT_NAME"
if [ -z "$EULA_RETRY_DIR" ]; then
  curl --fail --silent --show-error --proto '=https' --proto-redir '=https' \
    --location --max-redirs 1 \
    "$BASE_URL/Fiddler%20Everywhere%20${VERSION}.dmg" -o "$DMG"
elif [ ! -f "$DMG" ] || [ -L "$DMG" ]; then
  echo "Retained DMG is missing or unsafe" >&2
  exit 1
fi
openssl dgst -sha512 -binary -out "$STAGING_DIR/actual.sha512" "$DMG"
ACTUAL_SHA512=$(openssl base64 -A -in "$STAGING_DIR/actual.sha512")
if [ "$ACTUAL_SHA512" != "$EXPECTED_SHA512" ]; then
  echo "DMG integrity verification failed" >&2
  exit 1
fi

verify_app() {
  local candidate_app="$1"
  local signature_details
  codesign --verify --deep --strict --verbose=2 "$candidate_app" || return 1
  signature_details=$(codesign --display --verbose=4 "$candidate_app" 2>&1) || return 1
  if ! printf '%s\n' "$signature_details" | grep -Fqx \
    'Authority=Developer ID Application: Telerik A D (CHSQ3M3P37)'; then
    echo "Unexpected macOS code-signing authority" >&2
    return 1
  fi
  if ! printf '%s\n' "$signature_details" | grep -Fqx 'TeamIdentifier=CHSQ3M3P37'; then
    echo "Unexpected macOS signing Team ID" >&2
    return 1
  fi
  spctl --assess --type execute --verbose=2 "$candidate_app" || return 1
}

MOUNT_DIR="$STAGING_DIR/mount"
if [ -L "$MOUNT_DIR" ]; then
  echo "DMG mountpoint must not be a symlink" >&2
  exit 1
fi
mkdir -p -m 700 "$MOUNT_DIR"
ATTACH_LOG="$STAGING_DIR/hdiutil-attach.log"
ATTACH_STATUS=0
if [ "$EULA_ACCEPTED" = "1" ]; then
  if printf 'yes\n' | hdiutil attach -readonly -nobrowse -noautoopen -mountpoint "$MOUNT_DIR" "$DMG" 2>&1 | tee "$ATTACH_LOG"; then
    ATTACH_STATUS=0
  else
    ATTACH_STATUS=$?
  fi
else
  if hdiutil attach -readonly -nobrowse -noautoopen -mountpoint "$MOUNT_DIR" "$DMG" 2>&1 | tee "$ATTACH_LOG"; then
    ATTACH_STATUS=0
  else
    ATTACH_STATUS=$?
  fi
fi
if [ "$ATTACH_STATUS" -ne 0 ]; then
  PRESERVE_STAGING=1
  if grep -Fq 'End User License Agreement' "$ATTACH_LOG" && \
    grep -Eiq 'Error 112|user canceled|attach canceled' "$ATTACH_LOG"; then
    echo "EULA interaction canceled. Full output: $ATTACH_LOG" >&2
    echo "Verified files retained for consent retry: $STAGING_DIR" >&2
  fi
  echo "DMG attach failed with exit code $ATTACH_STATUS; review $ATTACH_LOG. Staging is retained for diagnosis." >&2
  exit "$ATTACH_STATUS"
fi
APP_PATH="$MOUNT_DIR/Fiddler Everywhere.app"
if [ ! -d "$APP_PATH" ]; then
  echo "Fiddler Everywhere.app was not found in the mounted DMG" >&2
  exit 1
fi
verify_app "$APP_PATH" || exit 1

APPLICATIONS_DIR="$HOME/Applications"
mkdir -p "$APPLICATIONS_DIR"
INSTALL_DIR=$(mktemp -d "$APPLICATIONS_DIR/.fiddler-install.XXXXXX")
chmod 700 "$INSTALL_DIR"
INSTALL_APP="$INSTALL_DIR/Fiddler Everywhere.app"
ditto --rsrc --extattr --acl --qtn "$APP_PATH" "$INSTALL_APP"
verify_app "$INSTALL_APP" || exit 1

DESTINATION="$APPLICATIONS_DIR/Fiddler Everywhere.app"
BACKUP="$INSTALL_DIR/Fiddler Everywhere.previous.app"
if [ -e "$DESTINATION" ] || [ -L "$DESTINATION" ]; then
  mv "$DESTINATION" "$BACKUP"
fi
if ! mv "$INSTALL_APP" "$DESTINATION"; then
  if [ -e "$BACKUP" ] || [ -L "$BACKUP" ]; then
    if ! mv "$BACKUP" "$DESTINATION"; then
      echo "Install failed; previous app remains at $BACKUP" >&2
      INSTALL_DIR=""
      exit 1
    fi
  fi
  echo "Could not move the verified app into $APPLICATIONS_DIR" >&2
  exit 1
fi
rm -rf "$INSTALL_DIR"
INSTALL_DIR=""
```

### Linux

```bash
set -e

mkdir -p "$HOME/Downloads"
STAGING_DIR=$(mktemp -d "$HOME/Downloads/.fiddler-setup.XXXXXX") || exit 1
MANIFEST="$STAGING_DIR/manifest"
ACTUAL_DIGEST="$STAGING_DIR/actual-digest"
EXPECTED_DIGEST="$STAGING_DIR/expected-digest"
trap 'rm -rf "$STAGING_DIR"' EXIT
curl --fail --silent --show-error --proto '=https' --proto-redir '=https' --max-redirs 0 \
  "https://downloads.getfiddler.com/linux/latest-linux.yml" -o "$MANIFEST"
VERSION=$(awk -F': ' '$1 == "version" {print $2; exit}' "$MANIFEST")
if ! printf '%s\n' "$VERSION" | grep -Eq '^[0-9]+(\.[0-9]+)+$'; then
  echo "Manifest version is missing or malformed" >&2
  exit 1
fi
echo "Latest Fiddler Everywhere: $VERSION"
ARTIFACT_NAME="fiddler-everywhere-${VERSION}.AppImage"
EXPECTED_SHA512=$(awk -v artifact="$ARTIFACT_NAME" '
  /^[[:space:]]*-[[:space:]]+url:/ {
    url = $0
    sub(/^[[:space:]]*-[[:space:]]+url:[[:space:]]*/, "", url)
    sub(/[[:space:]]*$/, "", url)
    found = (url == artifact)
    next
  }
  found && /^[[:space:]]+sha512:[[:space:]]*/ {
    digest = $0
    sub(/^[[:space:]]+sha512:[[:space:]]*/, "", digest)
    sub(/[[:space:]]*$/, "", digest)
    print digest
    exit
  }
' "$MANIFEST")
if ! printf '%s\n' "$EXPECTED_SHA512" | grep -Eq '^[A-Za-z0-9+/]{86}==$'; then
  echo "Missing or malformed SHA-512 for $ARTIFACT_NAME" >&2
  exit 1
fi
printf '%s' "$EXPECTED_SHA512" | openssl base64 -d -A > "$EXPECTED_DIGEST"
if [ "$(wc -c < "$EXPECTED_DIGEST" | tr -d '[:space:]')" != "64" ] || \
  [ "$(openssl base64 -A -in "$EXPECTED_DIGEST")" != "$EXPECTED_SHA512" ]; then
  echo "Malformed SHA-512 for $ARTIFACT_NAME" >&2
  exit 1
fi

APPIMAGE="$HOME/Downloads/$ARTIFACT_NAME"
STAGED_APPIMAGE="$STAGING_DIR/$ARTIFACT_NAME"
curl --fail --silent --show-error --proto '=https' --proto-redir '=https' \
  --location --max-redirs 1 \
  "https://agent-downloads.getfiddler.com/linux/$ARTIFACT_NAME" -o "$STAGED_APPIMAGE"
openssl dgst -sha512 -binary -out "$ACTUAL_DIGEST" "$STAGED_APPIMAGE"
ACTUAL_SHA512=$(openssl base64 -A -in "$ACTUAL_DIGEST")
if [ "$ACTUAL_SHA512" != "$EXPECTED_SHA512" ]; then
  echo "AppImage integrity verification failed" >&2
  exit 1
fi
chmod +x "$STAGED_APPIMAGE"
mv -f "$STAGED_APPIMAGE" "$APPIMAGE"
"$APPIMAGE" &
```

### Windows (PowerShell)

The tracking endpoint requires a `User-Agent`; the PowerShell command below sets it on
the download client.

Create a new per-user temporary folder, verify its ACL, and keep download, verification,
installation, and cleanup in one PowerShell command:

```powershell
  $stagingDir = Join-Path ([System.IO.Path]::GetTempPath()) ("FiddlerSetup-" + [guid]::NewGuid().ToString("N"))
  New-Item -ItemType Directory -Path $stagingDir -ErrorAction Stop | Out-Null
  try {
    $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
    & icacls.exe $stagingDir /inheritance:r /grant:r "${currentUser}:(OI)(CI)F" | Out-Null
    if ($LASTEXITCODE -ne 0) {
      throw "Could not restrict access to the installer staging folder"
    }

    $manifest = Invoke-WebRequest "https://downloads.getfiddler.com/win/latest.yml" `
      -MaximumRedirection 0 -UseBasicParsing
    $content = $manifest.Content
    if ($content -is [byte[]]) {
      $text = [System.Text.Encoding]::UTF8.GetString($content)
    } else {
      $text = [string]$content
    }
    $text = $text.TrimStart([char]0xFEFF)
    $versionLine = $null
    foreach ($line in $text.Split([char]10)) {
      $line = $line.Trim()
      if ($line.StartsWith("version:")) {
        $versionLine = $line
        break
      }
    }
    if ($null -eq $versionLine) {
      $manifestPreview = ($text -split '\r?\n' | Select-Object -First 3) -join ' | '
      if ($manifestPreview.Length -gt 240) {
        $manifestPreview = $manifestPreview.Substring(0, 240)
      }
      throw "Manifest version line not found. Response begins: $manifestPreview"
    }
    $VERSION = $versionLine.Substring(8).Trim()
    try { [void][version]::Parse($VERSION) } catch { throw "Manifest version is malformed: $VERSION" }
    Write-Host "Latest Fiddler Everywhere: $VERSION"
    $installerName = "Fiddler Everywhere $VERSION.exe"
    $installerNamePattern = [regex]::Escape($installerName)
    $checksumMatch = [regex]::Match(
      $text,
      "(?ms)^[ \t]*-[ \t]+url:[ \t]+${installerNamePattern}[ \t]*\r?\n(?:(?!^[ \t]*-[ \t]+url:).)*?^[ \t]+sha512:[ \t]*([^\r\n]+)"
    )
    if (-not $checksumMatch.Success) {
      throw "Installer checksum not found for $installerName"
    }
    $expectedSha512 = $checksumMatch.Groups[1].Value.Trim()
    if ($expectedSha512 -notmatch '^[A-Za-z0-9+/]{86}==$') {
      throw "Installer SHA-512 is missing or malformed"
    }
    try {
      $expectedDigestBytes = [Convert]::FromBase64String($expectedSha512)
    } catch {
      throw "Installer SHA-512 is malformed"
    }
    if ($expectedDigestBytes.Length -ne 64 -or [Convert]::ToBase64String($expectedDigestBytes) -cne $expectedSha512) {
      throw "Installer SHA-512 is malformed"
    }

    $installerPath = Join-Path $stagingDir "FiddlerEverywhere.exe"
    $downloadUri = [Uri]"https://agent-downloads.getfiddler.com/win/Fiddler%20Everywhere%20$VERSION.exe"
    Add-Type -AssemblyName System.Net.Http -ErrorAction Stop
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
      [System.IO.File]::WriteAllBytes($installerPath, $response.Content.ReadAsByteArrayAsync().GetAwaiter().GetResult())
    } finally {
      $client.Dispose()
      $handler.Dispose()
    }

    $sha512 = [System.Security.Cryptography.SHA512]::Create()
    try {
      $stream = [System.IO.File]::OpenRead($installerPath)
      try {
        $actualSha512 = [Convert]::ToBase64String($sha512.ComputeHash($stream))
      } finally {
        $stream.Dispose()
      }
    } finally {
      $sha512.Dispose()
    }
    if ($actualSha512 -cne $expectedSha512) {
      throw "Installer integrity verification failed"
    }

    $signature = Get-AuthenticodeSignature -FilePath $installerPath
    if ($signature.Status -ne "Valid" -or $null -eq $signature.SignerCertificate) {
      throw "Installer Authenticode signature verification failed: $($signature.Status)"
    }
    $publisher = $signature.SignerCertificate.GetNameInfo(
      [System.Security.Cryptography.X509Certificates.X509NameType]::SimpleName,
      $false
    )
    if ($publisher -ne "Progress Software Corporation") {
      throw "Unexpected installer publisher: $publisher"
    }

    $installerProcess = Start-Process -FilePath $installerPath -ArgumentList "/S" -Wait -PassThru -ErrorAction Stop
    if ($installerProcess.ExitCode -ne 0) {
      throw "Fiddler Everywhere installer failed with exit code $($installerProcess.ExitCode)"
    }
    $installedRelease = Get-ItemProperty `
      "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
      "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
      "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" `
      -ErrorAction SilentlyContinue |
      Where-Object { $_.DisplayName -like "*Fiddler Everywhere*" -and $_.DisplayVersion -eq $VERSION }
    if (-not $installedRelease) {
      throw "Fiddler Everywhere $VERSION was not found in the installed application registry after installation"
    }
  } finally {
    Remove-Item -LiteralPath $stagingDir -Recurse -Force -ErrorAction Stop
  }
```
## Phase 4 — Launch Fiddler

The macOS app is installed per-user and the Linux AppImage is started in Phase 3. The
Windows installer also runs in Phase 3, where its exit code and installed version are
checked before continuing; use these commands to launch the installed app.

macOS:
```bash
if [ -d "$HOME/Applications/Fiddler Everywhere.app" ]; then
  open "$HOME/Applications/Fiddler Everywhere.app" && sleep 15
else
  open -a "Fiddler Everywhere" && sleep 15
fi
```

Linux:
The verified AppImage was started after its checksum passed in Phase 3.

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
if (-not $fiddlerExe) {
  throw "Installed Fiddler Everywhere executable was not found"
}
$cmdLine = '"' + $fiddlerExe + '"'
$launchResult = Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = $cmdLine }
if ($launchResult.ReturnValue -ne 0) {
  throw "Fiddler Everywhere process creation failed with code $($launchResult.ReturnValue)"
}
Start-Sleep 15
$fiddlerProcesses = Get-Process -Name "Fiddler*" -ErrorAction SilentlyContinue |
  Where-Object { $_.Path -eq $fiddlerExe }
if (-not $fiddlerProcesses) {
  throw "Fiddler Everywhere was launched but no process for $fiddlerExe was found"
}
$processIds = @($fiddlerProcesses.Id) -join ", "
Write-Host "Fiddler Everywhere is running (PID(s) $processIds)"
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
| macOS DMG checksum, signature, or Gatekeeper check fails | Stop; do not install. Investigate the failed check before retrying. |
| hdiutil output contains an EULA and attach is canceled | Review the saved EULA log with the user. On acceptance, set `EULA_RETRY_DIR` and `EULA_ACCEPTED=1` and rerun; the script rechecks the retained DMG. On decline, remove the retained staging directory. |
| The execution environment requests `/Volumes` access | This is a separate permission for mounting, not EULA consent. Explain the request and proceed only with the user's choice. |
| `hdiutil attach` fails without an EULA section and cancellation marker | Inspect the retained `hdiutil-attach.log`, report the exact error, and stop for diagnosis. Remove retained staging when no longer needed; do not add `sudo`, change mountpoints, re-download, or use verbose mode without supporting evidence. |
| Windows installer exits with an error or the expected version is not registered | Report the failure and stop; do not launch an older installation or report upgrade success. |
| Windows EXE blocked by SmartScreen | Right-click → Run as Administrator |
