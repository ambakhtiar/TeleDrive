  # TeleDrive Full Implementation Plan

  ## Product Summary

  Build TeleDrive as an Android-only private APK for personal and community use. It is offline-first and local-only: users
  authenticate directly with Telegram, select device folders, and upload original files to Telegram forum topics. No backend, cloud
  database, analytics, or account data service will exist.

  The existing Python/FastAPI uploader remains as a legacy reference. The new React Native application will live in mobile/.

  ## Repository And Build Foundation

  - Replace the existing Git metadata with a new main repository connected to https://github.com/ambakhtiar/TeleDrive.git.
  - Preserve project files and the current Python uploader; exclude all local agent/editor configuration from Git.
  - Scaffold mobile/ with Expo, TypeScript, Expo Router, Expo SQLite, EAS configuration, and a development client.
  - Add EAS profiles:
      - development: development-client APK for physical-device debugging.
      - preview: signed private APK for testers.
      - production: signed Android release configuration for future Play Store use.

  - Add an Expo config plugin that injects Android permissions, Kotlin sources, Gradle dependencies, TDLib binaries, WorkManager,
    foreground-service setup, and notification configuration.

  - Use EAS Cloud for native builds. Use VS Code, Android Platform Tools, USB debugging, and adb logcat; Android Studio and an
    emulator are not required.

  ## Native Android Design

  - Build a Kotlin TeleDriveModule with strongly typed React Native methods and events.
  - Embed the global Telegram API_ID and API_HASH through native build configuration. Users never provide API credentials.
  - Use TDLib for phone-number login, OTP, optional 2FA password, session persistence, logout, group discovery, topic discovery,
    topic creation, and document upload.

  - Store TDLib session files in app-private storage and protect sensitive local values with Android Keystore encryption.
  - Use ACTION_OPEN_DOCUMENT_TREE only. Persist URI grants and recursively scan the selected folder tree and all nested folders.
  - Use a foreground service for continuous backup and visible upload notifications.
  - Use WorkManager for durable one-time synchronization, retry scheduling, device-restart recovery, Wi-Fi-only constraints, and
    charging-only constraints.

  - Run 1-4 TDLib uploads concurrently with Kotlin coroutines and a semaphore. Never exceed the configured concurrency limit.
  - Emit typed real-time progress events for every active upload, including bytes transferred, total bytes, speed, percentage, ETA,
    queue position, and error state.

  ## SQLite Data Design

  - Create a local upload_queue table with:
      - id, file_uri, filename, file_size, checksum, mime_type, modified_time, source_folder_id, destination_topic_id, status,
        retry_count, error_message, telegram_msg_link, created_at, updated_at.

  - Add indexes for status, filename, modified_time, source_folder_id, and daily reporting queries.
  - Add local tables for folder sources, routing rules, upload settings, daily upload summaries, and schema migrations.
  - Use Expo SQLite for React Native history/dashboard reads. Kotlin owns migrations and queue writes through the same app-local
    SQLite database, avoiding duplicate queue state.

  - Use checksum plus URI, size, and modified timestamp to prevent duplicate uploads.

  ## Full Feature List

  ### Authentication

  - Phone number login through TDLib.
  - OTP verification screen.
  - Optional Telegram 2FA password screen.
  - Clear loading, invalid-code, rate-limit, network, and login-failure states.
  - Logout and secure local session removal.
  - No email login, no backend account, and no user-provided API key.

  ### Folder Management

  - System folder picker for DCIM, Downloads, SD card, or another user-selected root.
  - Persistent access to selected folders after app restart.
  - Recursive scanning of all nested folders.
  - Add, edit, disable, rescan, and remove folder sources.
  - Folder-permission error state with a “Grant access again” action.
  - File count and size preview before first sync.

  ### Telegram Groups And Topics

  - List eligible Telegram forum groups for the signed-in user.
  - Validate forum status and topic-management permissions.
  - List existing topics in the selected group.
  - Create missing topics using folder names or extension names when allowed.
  - Display a clear permission failure when the user cannot create topics.

  ### Smart Routing

  - Routing priority is fixed:
      1. extension rule;
      2. folder-name rule;
      3. selected fallback topic.

  - Extension examples: .mp4 -> Videos, .pdf -> Documents.
  - Folder examples: Camera -> Photos, WhatsApp Video -> Videos.
  - Topic routing preview showing the destination for sampled files.
  - Custom tags stored per rule or folder source.

  ### Upload Queue

  - Manual “Sync Now” scans folders and adds missing files to the durable queue.
  - Continuous Backup scans selected folders while the foreground service is enabled.
  - Original document upload only, without media compression.
  - Captions include filename, exact size, available local date, #extension, #folder_name, and custom tags.
  - Use “Modified date” when Android cannot provide a trustworthy creation date.
  - Pause, resume, cancel item, retry item, and Retry All controls.
  - Automatic retry with bounded retry count and stored failure reason.
  - Duplicate prevention and post-restart resume.
  - Telegram message link saved after successful TDLib confirmation.
  - Auto-delete disabled by default; delete only after confirmed upload, stored message link, and final local-file validation.

  ### Dashboard And Queue UI

  - Daily upload count and total uploaded bytes.
  - Active queue with 1-4 live parallel progress bars.
  - Queue statuses:
      - green: success;
      - yellow: pending or uploading;
      - red: failed.

  - Upload speed, ETA, remaining files, current size, and recent activity.
  - Empty states, permission states, offline states, and retry actions.

  ### History And Reporting

  - Infinite scrolling with FlatList, LIMIT 20 OFFSET ?, and stable pagination.
  - Filename search using indexed LIKE '%keyword%'.
  - Filters for status and source folder.
  - Sorting by date newest/oldest and file size.
  - Clickable Telegram message links through Linking.openURL().
  - Daily reports with file count and total bytes.
  - Local CSV export for upload history.
  - No remote report storage or analytics.

  ### Settings And Onboarding

  - Three-step onboarding: Telegram login, folder permission, battery-optimization settings.
  - Open Android battery-optimization settings and explain why continuous backup needs exemption.
  - Max concurrent uploads selector from 1 to 4.
  - Wi-Fi-only toggle.
  - Charging-only toggle.
  - Auto-delete toggle with explicit destructive confirmation.
  - Continuous Backup toggle.
  - Clear local queue/history and logout controls with confirmation.
  - Privacy screen stating that TeleDrive stores data only on the device and uploads only to the user-selected Telegram
    destination.

  ## React Native Application Structure

  - mobile/app/: Expo Router screens for onboarding, dashboard, folders, routing, queue, history, settings, and privacy.
  - mobile/src/native/: typed TypeScript interface for TeleDriveModule and native event payloads.
  - mobile/src/database/: Expo SQLite queries for dashboard, queue, history, search, filters, and migrations validation.
  - mobile/src/features/: feature-specific screens, hooks, types, validation, loading states, error states, and empty states.
  - mobile/plugins/: Expo config plugin for native Android injection.
  - mobile/android/: generated/prebuilt Android project containing the Kotlin module, TDLib integration, foreground service,
    WorkManager workers, notification channels, and SAF support.

  ## USB Development Workflow

  - Enable Developer Options and USB Debugging on the Android phone.
  - Install Android Platform Tools only.
  - Create and install the first development APK:

  eas build --profile development --platform android

  - Start the development server:

  npx expo start --dev-client

  - Use USB with adb and adb logcat for native logs and troubleshooting.
  - Rebuild through EAS only when native Kotlin, TDLib, Android permissions, or Expo plugin configuration changes.
  - React Native TypeScript/UI changes reload through the installed development client.
  - Build shareable private APKs with:

  eas build --profile preview --platform android

  ## Testing And Acceptance Criteria

  - Verify Telegram login with OTP, optional 2FA, bad code, rate limit, and logout.
  - Verify SAF access survives app restart and scans nested folders.
  - Verify extension, folder, and fallback routing priority.
  - Verify topic listing, topic creation, missing permissions, and non-forum group failures.
  - Verify original file integrity, caption formatting, custom tags, message-link storage, duplicate prevention, and auto-delete
    safety.

  - Verify 1, 2, 3, and 4 concurrent upload limits and real-time progress rendering.
  - Verify queue recovery after network loss, app close, device restart, low battery, and Telegram errors.
  - Verify Wi-Fi-only, charging-only, battery-optimization guidance, and foreground notification behavior.
  - Verify history pagination, search, filtering, sorting, deep links, and local daily stats.
  - Verify development and preview APK installation on a physical Android device through USB.

  ## Assumptions

  - Phase 1 is Android-only and private-APK distribution only.
  - Each person uses their own Telegram account and selects their own groups and folders.
  - The global Telegram API credentials are bundled as the app’s identity and are never shown to users or committed as plaintext.
  - Team/shared workspace functionality, cross-device sync, and any remote administration are deferred to a later phase because
    they require a separate backend and privacy model.

