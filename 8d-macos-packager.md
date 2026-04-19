# macos-packager Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a new repo with build scripts that pull, build, and bundle all three component repos into a single macOS .app/.dmg installer with fixed install paths.

**Architecture:** Shell scripts that clone/build each component, assemble them into an Electron .app bundle, and package as a .dmg. The packager is build-time only — it produces the artifact but is not included in it.

**Tech Stack:** Bash, Electron Builder (via keyboards-mcp), macOS codesign/notarytool

**Spec:** `../keyboards-mcp/docs/superpowers/specs/2026-04-19-multi-repo-split-design.md`

**Prerequisite:** Plan 8a (keyboards-mcp cleanup) must be complete — the parent folder `~/test/sounds-and-recreation/` must exist.

---

### Task 1: Initialize repo with CI

**Files:**
- Create: `~/test/sounds-and-recreation/macos-packager/.github/workflows/ci.yml`
- Create: `~/test/sounds-and-recreation/macos-packager/.github/workflows/release.yml`
- Create: `~/test/sounds-and-recreation/macos-packager/.gitignore`

- [ ] **Step 1: Initialize git repo**

```bash
cd ~/test/sounds-and-recreation/macos-packager
git init
```

- [ ] **Step 2: Create .gitignore**

```gitignore
build/
dist/
*.dmg
*.pkg
.env
```

- [ ] **Step 3: Create CI workflow**

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install shellcheck
        run: sudo apt-get install -y shellcheck
      - name: Lint shell scripts
        run: shellcheck scripts/*.sh

  dry-run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify scripts are executable
        run: |
          for f in scripts/*.sh; do
            test -x "$f" || { echo "Not executable: $f"; exit 1; }
          done
      - name: Verify config files exist
        run: |
          test -f config/entitlements.plist
          test -f config/Info.plist
```

- [ ] **Step 4: Create release workflow (placeholder)**

Create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build all components
        run: ./scripts/build-all.sh
      - name: Package
        run: ./scripts/package.sh
      # TODO: Upload artifact, sign, notarize
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: initialize repo with CI and release workflows"
```

---

### Task 2: Build script

**Files:**
- Create: `scripts/build-all.sh`

- [ ] **Step 1: Create build-all.sh**

Create `scripts/build-all.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"
WORKSPACE_DIR="$(dirname "$ROOT_DIR")"
BUILD_DIR="$ROOT_DIR/build"

echo "=== Sound Recreation: Build All ==="
echo "Workspace: $WORKSPACE_DIR"
echo "Build dir: $BUILD_DIR"

# Clean build directory
rm -rf "$BUILD_DIR"
mkdir -p "$BUILD_DIR"

# --- keyboards-mcp ---
echo ""
echo "--- Building keyboards-mcp ---"
KEYBOARDS_DIR="$WORKSPACE_DIR/keyboards-mcp"
if [ ! -d "$KEYBOARDS_DIR" ]; then
  echo "ERROR: keyboards-mcp not found at $KEYBOARDS_DIR"
  exit 1
fi
cd "$KEYBOARDS_DIR"
npm ci
npm run build
cp -r dist "$BUILD_DIR/keyboards-mcp"
cp package.json "$BUILD_DIR/keyboards-mcp/"
cp -r node_modules "$BUILD_DIR/keyboards-mcp/"
echo "keyboards-mcp: OK"

# --- sound-recreation-agent ---
echo ""
echo "--- Building sound-recreation-agent ---"
AGENT_DIR="$WORKSPACE_DIR/sound-recreation-agent"
if [ ! -d "$AGENT_DIR" ]; then
  echo "ERROR: sound-recreation-agent not found at $AGENT_DIR"
  exit 1
fi
cd "$AGENT_DIR"
npm ci
npm run build
cp -r dist "$BUILD_DIR/sound-recreation-agent"
cp package.json "$BUILD_DIR/sound-recreation-agent/"
cp -r node_modules "$BUILD_DIR/sound-recreation-agent/"
cp -r prompts "$BUILD_DIR/sound-recreation-agent/"
echo "sound-recreation-agent: OK"

# --- audio-analysis-mcp ---
echo ""
echo "--- Building audio-analysis-mcp ---"
AUDIO_DIR="$WORKSPACE_DIR/audio-analysis-mcp"
if [ ! -d "$AUDIO_DIR" ]; then
  echo "ERROR: audio-analysis-mcp not found at $AUDIO_DIR"
  exit 1
fi
cd "$AUDIO_DIR"
uv sync
cp -r src "$BUILD_DIR/audio-analysis-mcp-src"
cp -r .venv "$BUILD_DIR/audio-analysis-mcp-venv"
echo "audio-analysis-mcp: OK"

echo ""
echo "=== Build complete: $BUILD_DIR ==="
ls -la "$BUILD_DIR"
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x scripts/build-all.sh
```

- [ ] **Step 3: Run shellcheck**

```bash
shellcheck scripts/build-all.sh
```

Expected: No errors.

- [ ] **Step 4: Commit**

```bash
git add scripts/build-all.sh
git commit -m "feat: add build-all script"
```

---

### Task 3: Package script

**Files:**
- Create: `scripts/package.sh`

- [ ] **Step 1: Create package.sh**

Create `scripts/package.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"
BUILD_DIR="$ROOT_DIR/build"
DIST_DIR="$ROOT_DIR/dist"
APP_NAME="Sound Recreation"
APP_BUNDLE="$DIST_DIR/$APP_NAME.app"

echo "=== Packaging: $APP_NAME ==="

if [ ! -d "$BUILD_DIR/keyboards-mcp" ]; then
  echo "ERROR: Build artifacts not found. Run build-all.sh first."
  exit 1
fi

# Clean dist
rm -rf "$DIST_DIR"
mkdir -p "$DIST_DIR"

# Create .app bundle structure
mkdir -p "$APP_BUNDLE/Contents/MacOS"
mkdir -p "$APP_BUNDLE/Contents/Resources"

# Copy Info.plist
cp "$ROOT_DIR/config/Info.plist" "$APP_BUNDLE/Contents/"

# Copy built components into Resources
cp -r "$BUILD_DIR/keyboards-mcp" "$APP_BUNDLE/Contents/Resources/"
cp -r "$BUILD_DIR/sound-recreation-agent" "$APP_BUNDLE/Contents/Resources/"
cp -r "$BUILD_DIR/audio-analysis-mcp-src" "$APP_BUNDLE/Contents/Resources/audio-analysis-mcp/"
cp -r "$BUILD_DIR/audio-analysis-mcp-venv" "$APP_BUNDLE/Contents/Resources/audio-analysis-mcp/.venv"

echo ""
echo "=== Package complete ==="
echo "App bundle: $APP_BUNDLE"
du -sh "$APP_BUNDLE"

echo ""
echo "NOTE: Electron binary, code signing, and DMG creation are not yet implemented."
echo "This script currently assembles the Resources bundle only."
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x scripts/package.sh
```

- [ ] **Step 3: Run shellcheck**

```bash
shellcheck scripts/package.sh
```

Expected: No errors.

- [ ] **Step 4: Commit**

```bash
git add scripts/package.sh
git commit -m "feat: add package script (assembles .app bundle)"
```

---

### Task 4: Config files

**Files:**
- Create: `config/Info.plist`
- Create: `config/entitlements.plist`

- [ ] **Step 1: Create Info.plist**

Create `config/Info.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>Sound Recreation</string>
    <key>CFBundleDisplayName</key>
    <string>Sound Recreation</string>
    <key>CFBundleIdentifier</key>
    <string>com.soundrecreation.app</string>
    <key>CFBundleVersion</key>
    <string>0.1.0</string>
    <key>CFBundleShortVersionString</key>
    <string>0.1.0</string>
    <key>CFBundleExecutable</key>
    <string>Sound Recreation</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>LSMinimumSystemVersion</key>
    <string>13.0</string>
    <key>NSHighResolutionCapable</key>
    <true/>
</dict>
</plist>
```

- [ ] **Step 2: Create entitlements.plist**

Create `config/entitlements.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Network access (agent HTTP server, Vercel AI Gateway, YouTube) -->
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <true/>

    <!-- Audio input/output (sounddevice for capture/render) -->
    <key>com.apple.security.device.audio-input</key>
    <true/>

    <!-- File access (backup files, audio cache) -->
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
</plist>
```

- [ ] **Step 3: Commit**

```bash
git add config/
git commit -m "feat: add Info.plist and entitlements"
```

---

### Task 5: Sign and notarize script (placeholder)

**Files:**
- Create: `scripts/sign-and-notarize.sh`

- [ ] **Step 1: Create sign-and-notarize.sh**

Create `scripts/sign-and-notarize.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"
DIST_DIR="$ROOT_DIR/dist"
APP_BUNDLE="$DIST_DIR/Sound Recreation.app"
ENTITLEMENTS="$ROOT_DIR/config/entitlements.plist"

echo "=== Code Signing & Notarization ==="

if [ ! -d "$APP_BUNDLE" ]; then
  echo "ERROR: App bundle not found. Run package.sh first."
  exit 1
fi

# Check for required environment
if [ -z "${DEVELOPER_ID_APPLICATION:-}" ]; then
  echo "ERROR: DEVELOPER_ID_APPLICATION not set."
  echo "Set to your Apple Developer ID certificate name, e.g.:"
  echo '  export DEVELOPER_ID_APPLICATION="Developer ID Application: Your Name (TEAM_ID)"'
  exit 1
fi

echo "Signing with: $DEVELOPER_ID_APPLICATION"

# Sign the app bundle
codesign --force --deep --sign "$DEVELOPER_ID_APPLICATION" \
  --entitlements "$ENTITLEMENTS" \
  --options runtime \
  "$APP_BUNDLE"

echo "Signed: $APP_BUNDLE"

# Create DMG
DMG_PATH="$DIST_DIR/SoundRecreation.dmg"
hdiutil create -volname "Sound Recreation" \
  -srcfolder "$DIST_DIR" \
  -ov -format UDZO \
  "$DMG_PATH"

echo "DMG created: $DMG_PATH"

# Notarize (requires Apple ID credentials)
if [ -n "${APPLE_ID:-}" ] && [ -n "${APPLE_TEAM_ID:-}" ]; then
  echo "Submitting for notarization..."
  xcrun notarytool submit "$DMG_PATH" \
    --apple-id "$APPLE_ID" \
    --team-id "$APPLE_TEAM_ID" \
    --keychain-profile "notarytool" \
    --wait

  xcrun stapler staple "$DMG_PATH"
  echo "Notarization complete."
else
  echo "Skipping notarization (APPLE_ID and APPLE_TEAM_ID not set)."
fi

echo ""
echo "=== Done ==="
echo "Output: $DMG_PATH"
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x scripts/sign-and-notarize.sh
```

- [ ] **Step 3: Run shellcheck**

```bash
shellcheck scripts/sign-and-notarize.sh
```

Expected: No errors.

- [ ] **Step 4: Commit**

```bash
git add scripts/sign-and-notarize.sh
git commit -m "feat: add sign-and-notarize script"
```

---

### Task 6: CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Create CLAUDE.md**

```markdown
# CLAUDE.md

## Build & Package

```bash
./scripts/build-all.sh           # Clone/pull + build all 3 component repos
./scripts/package.sh             # Assemble .app bundle
./scripts/sign-and-notarize.sh   # Sign, create DMG, notarize (requires certs)
```

## Linting

```bash
shellcheck scripts/*.sh          # Lint all shell scripts
```

## CI

- `ci.yml` — shellcheck + dry-run verification on every push/PR
- `release.yml` — full build + package, triggered by git tags (`v*`)

## Architecture

Build-time-only repo that produces a macOS `.app` bundle containing:

```
Sound Recreation.app/
  Contents/
    MacOS/Sound Recreation        # Electron binary (from keyboards-mcp mock runner)
    Resources/
      keyboards-mcp/              # Built Node MCP server
      sound-recreation-agent/     # Built Node agent
      audio-analysis-mcp/         # Python venv + source
```

All paths are relative to the app bundle — no runtime discovery needed.

### Workspace

Part of `~/test/sounds-and-recreation/`:
- `../keyboards-mcp/` — Keyboard MCP server + mock runner
- `../sound-recreation-agent/` — AI agent (TypeScript)
- `../audio-analysis-mcp/` — Audio analysis MCP server (Python)
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md"
```