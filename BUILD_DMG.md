# Building DMG Files for Warthog

This guide explains how to build macOS DMG installer files for the Warthog gRPC GUI client.

## Overview

Warthog uses a two-step build process:
1. **Bundle** - Creates `.app` bundles for both Intel (x86-64) and Apple Silicon (arm64) architectures
2. **Package** - Creates DMG installer files from the `.app` bundles

## Prerequisites

Before building, ensure you have the following tools installed:

### 1. Go (Required)
- Version 1.23.0 or later
- Check: `go version`

### 2. astilectron-bundler (Required)
Install the Electron bundler for Go:
```bash
go install github.com/asticode/go-astilectron-bundler/astilectron-bundler@latest
```

Ensure `$GOPATH/bin` is in your PATH:
```bash
export PATH=$PATH:$(go env GOPATH)/bin
```

To make this permanent, add it to your `~/.zshrc` or `~/.bash_profile`:
```bash
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.zshrc
source ~/.zshrc
```

### 3. create-dmg (Required)
Install via Homebrew:
```bash
brew install create-dmg
```

## Build Process

### Step 1: Bundle the Application

Navigate to the `bin` directory and run the bundler script:

```bash
cd /Users/edward/projects/forked/warthog/bin
./bundler-darwin.sh
```

This script will:
- Download Electron (v13.6.9) and Astilectron (v0.56.0) if not cached
- Copy resources to the deployment directory
- Build binaries for both architectures:
  - `darwin-amd64` (Intel Macs)
  - `darwin-arm64` (Apple Silicon Macs)
- Create `.app` bundles in `bin/distr/darwin-amd64/` and `bin/distr/darwin-arm64/`
- Embed version information from `bin/version`

**Build time:** Approximately 3-5 minutes (first build may take longer due to downloads)

**Output:**
- `bin/distr/darwin-amd64/Warthog.app`
- `bin/distr/darwin-arm64/Warthog.app`

### Step 2: Create DMG Installers

Navigate to the installer directory and run the DMG creation script:

```bash
cd bin/installer/darwin
chmod +x make-dmg.sh  # Only needed once
./make-dmg.sh
```

This script will:
- Create two DMG files with custom branding:
  - Volume name and icon
  - Window size and position
  - Application icon placement
  - Link to Applications folder

**Build time:** Approximately 1-2 minutes

**Output:**
- `Warthog-0.5.7-darwin-x86-64.dmg` (~90 MB) - For Intel Macs
- `Warthog-0.5.7-darwin-arm64.dmg` (~94 MB) - For Apple Silicon Macs

## Complete Build Command

To run both steps sequentially:

```bash
cd /Users/edward/projects/forked/warthog/bin && \
./bundler-darwin.sh && \
cd installer/darwin && \
./make-dmg.sh
```

## Build Artifacts

After a successful build, you'll find:

```
bin/
├── distr/
│   ├── darwin-amd64/
│   │   └── Warthog.app          # Intel Mac application
│   └── darwin-arm64/
│       └── Warthog.app          # Apple Silicon application
└── installer/
    └── darwin/
        ├── Warthog-0.5.7-darwin-x86-64.dmg   # Intel Mac installer
        └── Warthog-0.5.7-darwin-arm64.dmg    # Apple Silicon installer
```

## Changing the Version

To build with a different version number:

1. Edit `bin/version`:
   ```bash
   echo "0.6.0" > bin/version
   ```

2. Run the build process as normal

The version will be embedded in:
- The application binary
- The DMG file names
- The application's "About" dialog

## Troubleshooting

### Error: `astilectron-bundler: command not found`

**Solution:**
```bash
# Install the bundler
go install github.com/asticode/go-astilectron-bundler/astilectron-bundler@latest

# Add Go bin to PATH
export PATH=$PATH:$(go env GOPATH)/bin
```

### Error: `create-dmg: command not found`

**Solution:**
```bash
brew install create-dmg
```

### Error: `permission denied: ./bundler-darwin.sh`

**Solution:**
```bash
chmod +x bin/bundler-darwin.sh
chmod +x bin/installer/darwin/make-dmg.sh
```

### Compilation Error: `too many arguments in call to uc.workspaceRepo.Get`

This was a bug in the forked repository. The fix has been applied to:
- `business/usecase/workspace.export.go` (lines 18 and 60)

If you encounter this, ensure the `Get()` method is called without arguments:
```go
// Incorrect
w, err := uc.workspaceRepo.Get(nil)

// Correct
w, err := uc.workspaceRepo.Get()
```

### Build Cache Issues

If you encounter strange build errors, try cleaning the cache:

```bash
# Clean Go build cache
go clean -cache -modcache

# Remove bundler cache
rm -rf bin/tmp/cache
rm -rf bin/tmp/bind

# Rebuild
cd bin && ./bundler-darwin.sh
```

## Build Configuration

### Bundler Configuration
The bundler is configured via `bin/bundler-darwin.json`:
- Electron version: 13.6.9
- Astilectron version: 0.56.0
- Output path: `../../bin/distr`
- Working directory: `../../bin/tmp`
- Icons: `resources/icons/app.icns`

### DMG Configuration
The DMG creation is configured in `bin/installer/darwin/make-dmg.sh`:
- Volume name: "Warthog Intel Installer" / "Warthog M1 Installer"
- Window size: 800x400
- Icon size: 100px
- Application icon position: (200, 190)
- Applications folder link position: (600, 185)

## Development Notes

### CGO Requirement
The build requires CGO to be enabled (`CGO_ENABLED=1`) because:
- SQLite support via `mattn/go-sqlite3`
- Native macOS integrations

### Architecture-Specific Builds
The bundler creates separate binaries for each architecture:
- `bind_darwin_amd64.go` - Generated for Intel builds
- `bind_darwin_arm64.go` - Generated for Apple Silicon builds

These files are automatically created and cleaned up during the build process.

### Resources
Application resources are copied from `resources/` to the deployment directory during build:
- Icons (app.icns, tray24.png)
- UI templates
- Proto definitions

## Continuous Integration

For automated builds, you can use this script:

```bash
#!/bin/bash
set -e

# Ensure tools are installed
if ! command -v astilectron-bundler &> /dev/null; then
    echo "Installing astilectron-bundler..."
    go install github.com/asticode/go-astilectron-bundler/astilectron-bundler@latest
fi

if ! command -v create-dmg &> /dev/null; then
    echo "Installing create-dmg..."
    brew install create-dmg
fi

# Add Go bin to PATH
export PATH=$PATH:$(go env GOPATH)/bin

# Build
cd bin
./bundler-darwin.sh

# Create DMG
cd installer/darwin
./make-dmg.sh

echo "Build complete!"
echo "DMG files created in bin/installer/darwin/"
ls -lh *.dmg
```

## Distribution

The generated DMG files can be:
- Uploaded to GitHub Releases
- Distributed via direct download
- Notarized for Gatekeeper (requires Apple Developer account)

### Notarization (Optional)
For distribution outside the App Store, you should notarize the app:

```bash
# Sign the app
codesign --deep --force --verify --verbose --sign "Developer ID Application: Your Name" Warthog.app

# Create DMG (as normal)
# Then notarize
xcrun notarytool submit Warthog-0.5.7-darwin-arm64.dmg --apple-id your@email.com --team-id TEAMID --wait

# Staple the notarization
xcrun stapler staple Warthog-0.5.7-darwin-arm64.dmg
```

## Additional Resources

- [Astilectron Documentation](https://github.com/asticode/go-astilectron)
- [create-dmg Documentation](https://github.com/create-dmg/create-dmg)
- [Warthog GitHub Repository](https://github.com/forest33/warthog)
