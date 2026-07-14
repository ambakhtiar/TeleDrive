# TeleDrive — Expo React Native Android App

Local-first file backup to Telegram forum topics. Android-only, uses TDLib for native Telegram integration.

## Commands

```bash
cd TeleDrive

# Typecheck (run after edits)
npm run typecheck

# Lint
npm run lint

# Dev server (requires Android dev client installed on device)
npx expo start --dev-client
```

## Build

```bash
# Development APK (for USB debugging)
eas build --profile development --platform android

# Shareable private APK
eas build --profile preview --platform android
```

Requires `TELEDRIVE_API_ID` and `TELEDRIVE_API_HASH` set in EAS build environment. Never commit these.

## Critical: Expo v57

Read versioned docs at `https://docs.expo.dev/versions/v57.0.0/` before writing any code. The Expo SDK has changed significantly — do not rely on memory or older docs.

## No Expo Go

This app depends on native modules (TDLib, folder access, foreground service, WorkManager). **Expo Go cannot run it.** A development client APK must be installed on the device first.

## Key Stack

- Expo Router (file-based routing)
- expo-sqlite (local database)
- react-native-tdlib (Telegram integration)
- TypeScript (run `npm run typecheck` = `tsc --noEmit`)

## Project Structure

- `src/` — feature code, hooks, types, database queries
- `plugins/` — Expo config plugin for native Android injection
- `android/` — generated native project (Kotlin module, TDLib, WorkManager)
- `scripts/` — utility scripts
