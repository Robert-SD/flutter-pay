# Swift Package Manager Support — Implementation Plan

## Overview

Add Swift Package Manager (SPM) support to `pay_ios` Flutter plugin. This is a **parallel support** to existing CocoaPods — both methods will work.

**Reference:** [Flutter SPM Documentation](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)

---

## Project Analysis

### Current Structure (pay_ios)

```
pay_ios/
├── ios/
│   ├── Assets/           # Empty
│   ├── Classes/
│   │   ├── ApplePayButtonView.swift
│   │   ├── PayPlugin.swift
│   │   ├── PaymentExtensions.swift
│   │   └── PaymentHandler.swift
│   └── pay_ios.podspec
├── pubspec.yaml
└── test/
```

### Key Findings

- ✅ **No Pigeon** — No code generation used
- ✅ **No external dependencies** — Only Flutter + PassKit
- ✅ **No resources** — Assets folder is empty
- ✅ **No PrivacyInfo.xcprivacy** — Not required
- ✅ **No Bundle.module usage** — No resource access in Swift code
- ⚠️ **Minimum iOS: 8.0** — Will be updated to 13.0 (SPM requirement)
- ⚠️ **Swift: 5.0** — Compatible with SPM

### Dependencies (podspec)

```ruby
s.dependency 'Flutter'
s.frameworks = 'PassKit'
```

For SPM, PassKit must be added as a package dependency instead.

---

## Implementation Steps

### Step 1: Create Branch

```bash
git checkout -b feature/swift-package-manager
```

### Step 2: Create SPM Directory Structure

```
pay_ios/ios/
├── pay_ios/                    # NEW: SPM package root
│   ├── Package.swift
│   └── Sources/
│       └── pay_ios/
│           ├── ApplePayButtonView.swift
│           ├── PayPlugin.swift
│           ├── PaymentExtensions.swift
│           ├── PaymentHandler.swift
│           └── (include/ pay_ios/ .gitkeep)
├── Assets/                     # Keep (empty, can delete later)
├── Classes/                    # Keep for CocoaPods compatibility
└── pay_ios.podspec            # Update paths
```

### Step 3: Create Package.swift

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "pay_ios",
    platforms: [
        .iOS("13.0")  // SPM requires iOS 13+
    ],
    products: [
        .library(name: "pay_ios", targets: ["pay_ios"])
    ],
    dependencies: [
        .package(name: "FlutterFramework", path: "../FlutterFramework")
    ],
    targets: [
        .target(
            name: "pay_ios",
            dependencies: [
                .product(name: "FlutterFramework", package: "FlutterFramework")
            ]
        )
    ]
)
```

**Note:** FlutterFramework dependency requires the consuming app to use Flutter ≥3.41. If app uses older Flutter, CocoaPods will be used instead.

### Step 4: Move Swift Files

Move `ios/Classes/*.swift` → `ios/pay_ios/Sources/pay_ios/`

### Step 5: Update Import Statements

All files use `import Flutter` — this remains compatible with FlutterFramework.

### Step 6: Update pay_ios.podspec

Update source paths to point to new locations for CocoaPods compatibility:

```ruby
s.source_files = 'pay_ios/Sources/pay_ios/**/*.swift'
s.resource_bundles = {'pay_ios_privacy' => ['pay_ios/Sources/pay_ios/PrivacyInfo.xcprivacy']}
```

### Step 7: Update .gitignore

Add:
```
.build/
.swiftpm/
```

### Step 8: Update Minimum iOS Version (pubspec.yaml)

Change iOS minimum from 8.0 to 13.0 in podspec.

---

## Verification Checklist

### CocoaPods (must still work)

- [ ] `flutter config --no-enable-swift-package-manager`
- [ ] `flutter pub get` in example app
- [ ] Build example app
- [ ] `pod lib lint ios/pay_ios.podspec --configuration=Debug --skip-tests --use-modular-headers --use-libraries`

### Swift Package Manager

- [ ] `flutter config --enable-swift-package-manager`
- [ ] `flutter pub get` in example app
- [ ] Open in Xcode — verify "Package Dependencies" appears
- [ ] Build example app

---

## Open Questions / Pending Decision

1. **FlutterFramework Dependency** — The SPM integration as documented requires Flutter 3.41+ in the consuming app. Two options:
   - **A:** Include FlutterFramework dependency (SPM only works with Flutter ≥3.41 apps)
   - **B:** Omit FlutterFramework (SPM structure works, but falls back to CocoaPods until app upgrades)

   Decision needed before proceeding.

---

## Files to Modify

| File | Action |
|------|--------|
| `pay_ios/ios/pay_ios/Package.swift` | Create |
| `pay_ios/ios/pay_ios/Sources/pay_ios/` | Create directory + move Swift files |
| `pay_ios/ios/pay_ios/Sources/pay_ios/include/pay_ios/.gitkeep` | Create |
| `pay_ios/ios/pay_ios.podspec` | Update paths, iOS minimum |
| `pay_ios/.gitignore` | Add .build/, .swiftpm/ |

---

## Files to Keep Unchanged (for CocoaPods compatibility)

- `pay_ios/ios/Classes/` — kept for CocoaPods
- `pay_ios/ios/Assets/` — kept (empty anyway)
