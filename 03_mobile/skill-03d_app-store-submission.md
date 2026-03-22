# SKILL 03-D — How to Prepare and Submit a React Native App to App Store and Play Store
Category: Mobile Development
Applies to: React Native, Expo, EAS Build, App Store Connect, Google Play Console

## What this skill covers
Submitting to both app stores requires preparing signing credentials, configuring build variants, incrementing versions correctly, filling app metadata, meeting screenshot and review requirements, and managing staged rollouts. This skill covers the full pipeline from first EAS Build to a live production release on both iOS App Store and Google Play, and explains the recurring maintenance cycle for subsequent releases.

## When to activate this skill
- When preparing a React Native app for first-time store submission
- When configuring EAS Build for CI/CD releases
- When debugging store review rejections
- When setting up TestFlight or Play internal testing

## Core principles
1. **Use EAS for all credential management.** EAS Build handles certificate provisioning and keystore generation securely — never manage these manually unless forced to.
2. **Version code and build number are separate concerns.** `versionName` / `CFBundleShortVersionString` is user-facing. `versionCode` / `CFBundleVersion` is store-only and must be strictly incremented.
3. **Screenshots are not optional.** Both stores reject apps with missing, incorrect-resolution, or emulator-only screenshots.
4. **TestFlight and Play Internal testing gate all production pushes.** Never submit directly to production without internal review first.
5. **Review guidelines are asymmetric.** Apple reviews every build; Google is faster but blocks more features based on permission declarations.

## Step-by-step guide
1. **Init EAS:** `npx eas-cli@latest init`, configure `eas.json` with `development`, `preview`, and `production` profiles.
2. **Configure credentials:**
   - iOS: `eas credentials` — EAS generates a Distribution Certificate and Provisioning Profile automatically (if Apple Dev account is linked)
   - Android: `eas credentials` — EAS generates a keystore; download and back it up to a password manager
3. **Set versions in `app.json`:**
   ```json
   "version": "1.0.0",
   "ios": { "buildNumber": "1" },
   "android": { "versionCode": 1 }
   ```
4. **Trigger production builds:** `eas build --profile production --platform all`
5. **Test on TestFlight (iOS):** upload IPA via `eas submit --platform ios` or Transporter. Add internal testers. Run for at least 24 hours.
6. **Test on Play Internal track:** upload AAB via `eas submit --platform android` or Play Console. Add testers. Promote to Production when satisfied.
7. **Prepare App Store Connect metadata:** app name, subtitle, description, what's new text, support URL, privacy policy URL.
8. **Prepare Play Console metadata:** short description (80 chars), full description (4000 chars), feature graphic, promo video (optional).
9. **Screenshots:** required sizes — iPhone 6.7", iPad Pro 12.9" for iOS; a 16:9 phone screenshot for Android (1080×1920 minimum). Use real device or accurate simulator with production UI, no placeholder data.
10. **Submit for review:** in App Store Connect select the build and submit. Google Play: advance the release from Internal → Production with a staged rollout (start at 10–20%).

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Manually managing signing certificates in Keychain | Use `eas credentials` — EAS handles provisioning and rotation |
| Incrementing only `version` string for each build | Increment `versionCode` (Android) and `buildNumber` (iOS) on every upload regardless of user-visible version change |
| Screenshots from a simulator with default content | Real device or simulator with actual app UI and real-looking data |
| Submit directly to production without internal test | Internal testing track → TestFlight/Internal → staged rollout |
| Hardcoding the bundle identifier in multiple places | Define once in `app.json` `bundleIdentifier`/`package`, EAS propagates it |

## Stack-specific notes
**Expo Managed Workflow:** EAS Build handles native build steps. No Xcode or Android Studio required on dev machine. `app.json` is the single config source.
**Bare Workflow:** `android/app/build.gradle` and `ios/*.xcconfig` hold native version values — keep them in sync with `app.json` manually or via EAS native build config.
**CI/CD:** use `eas build --non-interactive` in GitHub Actions or similar. Secrets (EXPO_TOKEN) stored as environment secrets. Auto-increment `versionCode` via `eas.json` build hooks or a pre-build script.

## Common mistakes
1. **Reusing the same `buildNumber` on a new IPA upload.** App Store Connect rejects uploads with a duplicate build number silently. Always increment before every upload.
2. **Missing privacy manifest (iOS 17+).** Apps using certain APIs (file system, user defaults, keyboard) must include a `PrivacyInfo.xcprivacy` file or Apple review will reject the build.
3. **Incorrect permission strings.** iOS requires a human-readable justification string for every permission used (`NSCameraUsageDescription`, etc.). Missing or boilerplate strings like "This app uses the camera" are grounds for rejection.
4. **Uploading a debug build.** EAS `production` profile generates a release build; `preview` is for TestFlight internal only. Double-check the profile before submission.

## Checklist
- [ ] EAS project initialized and linked to Expo account
- [ ] iOS Distribution Certificate and Provisioning Profile generated via EAS
- [ ] Android keystore generated via EAS and backed up
- [ ] `versionCode` and `buildNumber` incremented for every upload
- [ ] `eas.json` has `development`, `preview`, and `production` profiles
- [ ] All required permission usage description strings set in `app.json`
- [ ] Privacy manifest included for iOS 17+ (if required APIs are used)
- [ ] Screenshots at required resolutions for all required device sizes
- [ ] App metadata complete in App Store Connect and Play Console
- [ ] Internal testing completed before production submission
- [ ] Staged rollout (10–20%) configured for Play Store production release
- [ ] Build tested on real device before store submission
