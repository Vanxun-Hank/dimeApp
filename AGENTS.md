# Repository Guidance

## Start Here

- Read [README.md](README.md) for setup, feature entry points, and verification commands. Run commands from the repository root.
- Open `app/dime.xcodeproj`; the app scheme is `dime`. This is a SwiftUI iOS app with WidgetKit and intent extensions, not a Swift package.
- The Xcode project uses explicit file references. When adding or moving compiled files, update `app/dime.xcodeproj/project.pbxproj` and the affected targets' source/resource membership. Keep `Secrets.example.swift` outside build sources.

## Data and Shared Targets

- Use `DataController.shared` and the existing transaction creation/save paths for writes. App UI, shortcuts, and widgets share Core Data and preferences.
- The compiled model is `app/dime/Data/MainModel.xcdatamodeld`, with `MainModel 2.xcdatamodel` selected. Check project references before touching the duplicate model under `app/MainModel.xcdatamodeld`. Preserve existing stores and model migration compatibility when changing the schema.
- `group.com.vanxun.dime` is used in persistence paths, `UserDefaults` suites, widgets, shortcuts, and entitlements. Treat a rename as a coordinated data migration, not a signing-only edit.
- This fork uses `NSPersistentContainer` for local storage. Legacy iCloud settings and dependencies remain; do not describe transaction sync as working without restoring and verifying it.

## Voice and Receipt Input

- Voice entry starts in `app/dime/Views/CustomTabBar.swift`, records through `app/dime/Utilities/SpeechRecognizer.swift`, parses through `GeminiService.parseVoiceInput`, and fills `app/dime/Views/TransactionView.swift`. Trace the whole flow for changes to voice input.
- Receipt scanning and voice parsing share `ReceiptScanResult` in `app/dime/Utilities/GeminiService.swift`. Changes to its JSON fields must reach single-transaction fill, category/date matching, and `BatchImportSheet` in `app/dime/Views/TransactionView.swift`.
- Keep the user review/import step for generated transactions. Preserve amount checks and handling of missing fields, unknown categories, permission denial, and network errors.
- Keep API keys in the ignored `app/dime/Utilities/Secrets.swift`; only commit the placeholder template. Do not log keys, receipt images, or speech transcripts. Camera, photo, microphone, and speech permission strings live in `app/dime/Info.plist`.

## Verification

- Follow the README simulator build command after Swift or project changes; an empty API key is sufficient to compile. Do not call Gemini merely to validate a build.
- There is no test target or CI workflow configured. Report build results separately from manual checks; do not claim `xcodebuild test` covers the app.
- For input changes, manually check permission denial, empty/failed recognition, and single/multiple transaction review. For shared-data changes, check widgets and shortcut-created transactions as well as the app.
- Preserve the surrounding SwiftUI styles, named colors, and Dynamic Type behavior. Check `app/Localizations/` and `app/dime/Views/Localizable.strings` when changing user-facing strings.
