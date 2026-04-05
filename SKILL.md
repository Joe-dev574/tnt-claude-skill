# TNT Complete Skill — iOS Best Practices for PulseAnvil

## Project Context
PulseAnvil is a fitness workout app for iOS + Apple Watch built with SwiftUI, SwiftData, CloudKit (premium), Sign in with Apple, and StoreKit 2. All data is local-first with optional end-to-end encrypted iCloud sync for premium subscribers. No external servers.

---

## 1. SwiftData + CloudKit Rules

### The #1 Rule — All Attributes Must Be Optional
CloudKit-backed SwiftData stores require **every** `@Model` attribute to be optional or have a CoreData-recognized default. Swift-level defaults like `= []` or `= ""` are **NOT** recognized by CoreData/CloudKit as schema-level defaults.

```swift
// WRONG — will crash on CloudKit-backed container init
var progressSelfies: [ProgressSelfie] = []
var name: String = ""

// CORRECT — truly optional
var progressSelfies: [ProgressSelfie]?
var name: String?

// At read sites, nil-coalesce:
let selfies = metrics.progressSelfies ?? []
```

**Why this matters**: Schema violations cause `ModelContainer` init to fail — including the in-memory fallback. The app fatally crashes with no recovery path.

### Schema Evolution
- Only additive changes: new optional properties, new entities
- Cannot remove, rename, or make properties non-optional
- Schema promotion (Development → Production) is one-way via CloudKit Dashboard
- Use `VersionedSchema` and `SchemaMigrationPlan` with `.lightweight` stages

### Relationship Rules
- Use `@Relationship(deleteRule: .cascade)` for owned children (Workout → Exercise, History → SplitTime)
- Use `@Relationship(deleteRule: .nullify)` for references (Category → Workout)
- Use `@Attribute(.externalStorage)` for large binary data (profile images)
- Use `@Transient` for non-persisted properties (Logger instances)

---

## 2. Container Initialization

### Never Block the Main Thread
`ModelContainer` init with CloudKit can take 25-30 seconds on slow networks. Never use `static let` for the container — `dispatch_once` will block whatever thread accesses it first.

```swift
// WRONG — blocks main thread
static let container = buildContainer()

// CORRECT — async init with reader/writer pattern
nonisolated(unsafe) private static var _container: ModelContainer?
private static let containerAccessQueue = DispatchQueue(
    label: "com.tnt.PulseAnvil.containerAccess",
    attributes: .concurrent
)

// Read: concurrent
public static var container: ModelContainer {
    containerAccessQueue.sync { _container ?? buildContainerSync() }
}

// Write: barrier
@MainActor
public static func initializeContainer() async {
    try? await Task.sleep(for: .milliseconds(150)) // let SwiftUI render first
    await withCheckedContinuation { continuation in
        DispatchQueue.global(qos: .userInitiated).async {
            let built = buildContainer()
            containerAccessQueue.sync(flags: .barrier) { _container = built }
            continuation.resume()
        }
    }
}
```

### Container Configuration by Tier
- **Premium**: `.private("iCloud.com.tnt.PulseAnvil")` — slow on bad networks
- **Free**: `.cloudKitDatabase(.none)` — always fast
- **watchOS**: Always CloudKit (Watch is premium-only, container inits before WCSession delivers flag)
- **Fallback**: In-memory store if primary fails (data won't persist, warn user)

---

## 3. Launch Screen & Splash

### System Launch Screen
- Defined in `Info.plist` → `UILaunchScreen` → `UIColorName: "SplashBackground"`
- References an asset catalog color by base name (not group path)
- iOS aggressively caches the launch screen — must **delete the app** to see changes

### Asset Catalog Colors
- Colors inside group folders (e.g., `BackgroundColors/SplashBackground.colorset`) are referenced by base name only: `Color("SplashBackground")`
- `Color("AssetName")` silently renders as **transparent/clear** if the asset doesn't exist — no crash, no warning
- Always verify colorset values in both light and dark appearances

### Splash → App Transition
- Splash screen view should use `Color("SplashBackground")` to match the system launch screen
- Container initialization runs in a `.task` on the splash screen
- 150ms sleep before heavy work lets SwiftUI render the first frame

---

## 4. Account Deletion (App Store Guideline 5.1.1(v))

Apps that support account creation **must** offer account deletion. For PulseAnvil (no external servers), this means:

1. Delete all SwiftData records: `context.delete(model: Type.self)` for all 7 model types
2. Clear App Group shared defaults (appleUserID, isPremium)
3. Clear UserDefaults preferences (theme, units, appearance, seeded-data flag)
4. Notify Watch to sign out via WCSession
5. Set `currentUser = nil` — returns to sign-in screen
6. CloudKit records auto-delete when SwiftData syncs local deletions

### UI Pattern
- Place in Settings under Account section
- Red destructive button with trash icon
- Two-step confirmation alert explaining what gets deleted and that it's irreversible

---

## 5. SwiftUI Patterns

### State & Environment
```swift
@State private var isEditing = false
@Environment(AuthenticationManager.self) private var authManager
@Environment(\.modelContext) private var context
@Environment(\.dismiss) private var dismiss
```

### Avoid Combine — Use async/await
Prefer Swift concurrency over Combine publishers. Use `async let` for parallel work, `Task { }` for fire-and-forget, `withCheckedContinuation` to bridge callbacks.

### Liquid Glass (iOS 26+)
```swift
.liquidGlass()                              // plain glass, capsule
.liquidGlass(cornerRadius: 16)              // plain glass, rounded rect
.liquidGlassInteractive(cornerRadius: 16)   // interactive press effect
.liquidGlass(tint: .blue, cornerRadius: 16) // tinted glass
```

### Settings Row Pattern
```swift
Button { action() } label: {
    SettingsRow(iconName: "sf.symbol", iconColor: .blue) {
        Text("Label")
    }
}
.buttonStyle(.plain)
```

---

## 6. Code Conventions

### Naming
- **Types**: PascalCase — `WorkoutListScreen`, `HealthMetrics`
- **Properties/Methods**: camelCase — `appleUserId`, `signOut()`
- **Enum cases**: ALL_CAPS for category colors — `STRENGTH`, `HIIT`, `CARDIO`
- **Views**: `*View.swift` or `*Screen.swift`

### File Organization
- `Shared/` — code used by both iOS and watchOS (models, managers, utilities)
- `PulseAnvil/` — iOS-only views, utilities, assets
- `PulseAnvilWatch Watch App/` — watchOS-only code

### Known Typos (Do Not Rename)
- `DeafaultDataSeeder.swift` — "Deafault" is baked in, renaming would break references
- `OnboardongFlowView.swift` — "Onboardong" is baked in

---

## 7. Performance Considerations

### Network Awareness
- WiFi launch: ~2 seconds
- Hotspot/slow network: ~27 seconds (CloudKit handshake, network-bound, cannot be reduced)
- Free users (no CloudKit) always launch fast regardless of network

### Image Handling
- Use `ImageOptimizer` to compress photos before SwiftData storage
- Profile image uses `@Attribute(.externalStorage)` for efficiency
- Progress selfies stored as `[ProgressSelfie]?` (array of Codable structs with Data)

---

## 8. Heart Rate Zone Calculation

| Scenario | Max HR Used | Why |
|---|---|---|
| Observed max HR >= 150 bpm | Observed value | Credible intense effort |
| Observed max HR < 150 bpm | 220 - age | Not credible, use formula |
| No age available | 190 bpm | Conservative fallback |

---

## 9. Onboarding Philosophy

### 4 Pages
1. **Welcome** (blue) — Mission statement + 5 loaded example routines
2. **Track** (cyan) — Live timer, splits, Watch heart rate
3. **Free & Premium** (orange) — What everyone gets vs premium features
4. **Your Data** (green) — Privacy, no ads, no servers, delete anytime

### Tone Guidelines
- Mission-focused: "A tool to help maintain your goals"
- Show value and curiosity, not commercial pressure
- No "hurry up and workout" language
- No low-level sales tactics
- Lead with what makes PulseAnvil genuinely useful

---

## 10. Testing & Debugging

### Quick Validation
- `XcodeRefreshCodeIssuesInFile` — fast live diagnostics without full build
- `ExecuteSnippet` — run code in context of a file like a REPL
- `BuildProject` — full compile check

### Common Debug Steps
1. Check `XcodeListNavigatorIssues` for workspace-wide errors
2. Check `GetBuildLog` filtered by severity
3. For crashes: read the full crash log, look for schema/model issues first
4. For stale behavior on device: clean build + reinstall

### CloudKit Debugging
- Schema violations → fatal crash with clear error message naming the offending attribute
- Slow init → always CloudKit handshake on premium, not a bug on slow networks
- Missing data after delete → CloudKit syncs deletions automatically, give it time
