# AmneziaWireGuard for iOS and macOS

This is a fork of [wireguard-apple](https://git.zx2c4.com/wireguard-apple) that replaces the WireGuard Go backend with [amnezia-wg](https://github.com/amnezia-vpn/amnezia-wg), adding traffic obfuscation parameters (`Jc`, `Jmin`, `Jmax`, `S1`, `S2`, `H1`–`H4`) to defeat deep packet inspection.

---

## Changes from upstream wireguard-apple

### macOS: Migration to System Extension

The original project used a legacy **App Extension** (`.appex`) for the macOS Network Extension, which is incompatible with Developer ID provisioning profiles on macOS 15+. This fork migrates the macOS VPN tunnel to a **System Extension** (`.systemextension`).

Key changes:

| File | Change |
|------|--------|
| `WireGuard.xcodeproj/project.pbxproj` | NE product type → `com.apple.product-type.system-extension`; `WRAPPER_EXTENSION = systemextension`; embed phase `dstSubfolderSpec = 16` (Contents/Library/SystemExtensions/) |
| `Sources/WireGuardNetworkExtension/main.swift` | New entry point calling `NEProvider.startSystemExtensionMode()` |
| `Sources/WireGuardNetworkExtension/Info.plist` | `CFBundlePackageType = SYSX`; added `NSSystemExtensionUsageDescription` |
| `Sources/WireGuardNetworkExtension/WireGuardNetworkExtension_macOS.entitlements` | `packet-tunnel-provider` → `packet-tunnel-provider-systemextension`; removed `com.apple.security.app-sandbox` (system extensions run as daemons) |
| `Sources/WireGuardApp/UI/macOS/WireGuard.entitlements` | Updated to `packet-tunnel-provider-systemextension`; App Group ID uses unprefixed form `group.<APP_ID>` |
| `Sources/WireGuardApp/UI/macOS/AppDelegate.swift` | Added `OSSystemExtensionRequestDelegate` conformance; calls `installSystemExtensionIfNeeded()` at launch |

### Build fixes for Xcode 26 / amnezia-wg

- **`Sources/WireGuardKitC/WireGuardKitC.h`**: Added `#include <sys/types.h>` for BSD types (`u_char`, `u_int32_t`) on newer macOS SDKs.
- **`Sources/WireGuardKitGo/Makefile`**: Fixed version header extraction regex to match `amnezia-vpn/amnezia-wg` instead of `golang.zx2c4.com/wireguard`.
- **`Sources/WireGuardApp/UI/macOS/ParseError+WireGuardAppError.swift`**: Added missing `interfaceHasInvalidCustomParam` switch case.
- **`.swiftlint.yml`**: Relaxed rules (`cyclomatic_complexity`, `type_body_length`, `identifier_name`, `line_length`) to allow AWG-specific naming conventions.

---

## Building for macOS (Developer ID distribution)

### Prerequisites

```bash
brew install swiftlint go
```

### 1. Apple Developer portal setup

In [developer.apple.com](https://developer.apple.com):

1. **Create two explicit App IDs** (macOS platform):
   - `com.yourteam.awg.macos` — enable **Network Extensions** and **System Extension** capabilities, and **App Groups**
   - `com.yourteam.awg.macos.network-extension` — enable **Network Extensions** (packet-tunnel-provider-systemextension)

2. **Create an App Group**: `group.com.yourteam.awg.macos` — assign it to both App IDs above.

3. **Create two Developer ID provisioning profiles**:
   - *AWG macOS* — for `com.yourteam.awg.macos`
   - *AWG macOS Network Extension* — for `com.yourteam.awg.macos.network-extension`

4. Download and double-click both profiles to install them.

### 2. Configure Developer.xcconfig

```bash
cp Sources/WireGuardApp/Config/Developer.xcconfig.template \
   Sources/WireGuardApp/Config/Developer.xcconfig
```

Edit the file:

```xcconfig
DEVELOPMENT_TEAM = YOUR_TEAM_ID
APP_ID_IOS       = com.yourteam.awg.ios
APP_ID_MACOS     = com.yourteam.awg.macos
ARCHS            = arm64 x86_64
```

> **Note:** `Developer.xcconfig` is tracked in this repo for convenience. Remove your Team ID before pushing to a public fork if preferred.

### 3. Build

```bash
xcodebuild -scheme WireGuardmacOS -configuration Release \
  -destination 'platform=macOS' \
  ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO \
  build
```

### 4. Post-build: embed system extension and re-sign

Xcode does not automatically embed the system extension via the CopyFiles build phase in all configurations. Run this script after a successful build (adjust `DERIVED` to your DerivedData path):

```bash
DERIVED=$(xcodebuild -scheme WireGuardmacOS -configuration Release \
  -showBuildSettings 2>/dev/null | grep ' BUILT_PRODUCTS_DIR' | awk '{print $3}')
APP="$DERIVED/WireGuard.app"
INTERMED="${DERIVED%/Build/Products/Release}/Build/Intermediates.noindex/WireGuard.build/Release"
CERT="Developer ID Application: Your Name (TEAMID)"
PROFILE_UUID="your-main-app-profile-uuid"

# Embed system extension
mkdir -p "$APP/Contents/Library/SystemExtensions"
cp -R "$DERIVED/WireGuardNetworkExtension.systemextension" \
      "$APP/Contents/Library/SystemExtensions/"

# Remove debug-only entitlement added by Xcode
NE_XCENT="$INTERMED/WireGuardNetworkExtensionmacOS.build/WireGuardNetworkExtension.systemextension.xcent"
MAIN_XCENT="$INTERMED/WireGuardmacOS.build/WireGuard.app.xcent"
/usr/libexec/PlistBuddy -c "Delete :com.apple.security.get-task-allow" "$NE_XCENT" 2>/dev/null
/usr/libexec/PlistBuddy -c "Delete :com.apple.security.get-task-allow" "$MAIN_XCENT" 2>/dev/null

# Re-sign system extension first, then the whole app
codesign --force --sign "$CERT" -o runtime --timestamp \
  --entitlements "$NE_XCENT" \
  "$APP/Contents/Library/SystemExtensions/WireGuardNetworkExtension.systemextension"

cp "$HOME/Library/MobileDevice/Provisioning Profiles/${PROFILE_UUID}.provisionprofile" \
   "$APP/Contents/embedded.provisionprofile"

codesign --force --deep --sign "$CERT" -o runtime --timestamp \
  --entitlements "$MAIN_XCENT" "$APP"
```

### 5. Notarize and staple

```bash
# Store credentials once (use an App-Specific Password from appleid.apple.com)
xcrun notarytool store-credentials "notarytool-profile" \
  --apple-id "you@example.com" \
  --team-id "YOUR_TEAM_ID"

# Create DMG and submit
hdiutil create -volname "AmneziaWireGuard" -srcfolder "$APP" \
  -ov -format UDZO /tmp/AWG.dmg

xcrun notarytool submit /tmp/AWG.dmg \
  --keychain-profile "notarytool-profile" --wait

# Staple the ticket
xcrun stapler staple "$APP"
```

### 6. First launch — allow system extension

On the first launch, macOS will show a notification that a system extension was blocked. Open **System Settings → Privacy & Security** and click **Allow** next to the WireGuard Network Extension. The extension is then active for all subsequent launches.

---

## Building for iOS

Open `WireGuard.xcodeproj` in Xcode and select the `WireGuardiOS` scheme. Standard iOS provisioning applies.

---

## WireGuardKit integration

1. Add the Swift package to your Xcode project:

   ```
   https://github.com/cfa532/awg-apple
   ```

2. `WireGuardKit` links against `wireguard-go-bridge` but cannot build it automatically. Add an External Build System target:
   - Product name: `WireGuardGoBridge<Platform>` (e.g. `WireGuardGoBridgemacOS`)
   - Build tool: `/usr/bin/make`
   - Directory: `${BUILD_DIR%Build/*}SourcePackages/checkouts/awg-apple/Sources/WireGuardKitGo`
   - Build Settings → `SDKROOT`: `macosx` or `iphoneos`

3. In your Network Extension target's Build Phases:
   - Add `WireGuardGoBridge<Platform>` to Dependencies
   - Add `WireGuardKit` to Link Binary With Libraries

4. In your main app target's Build Phases:
   - Add `WireGuardKit` to Link Binary With Libraries

---

## MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
