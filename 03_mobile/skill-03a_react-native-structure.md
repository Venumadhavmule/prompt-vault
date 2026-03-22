# SKILL 03-A — How to Structure a React Native / Expo Application
Category: Mobile Development
Applies to: React Native, Expo SDK, Expo Router

## What this skill covers
React Native applications without structure become maintenance nightmares as they grow. The same component-layering and feature-folder principles from React apply — but with mobile-specific concerns: platform differences (iOS vs Android), native module integration, and the Expo managed vs bare workflow distinction. This skill defines the correct project structure for any React Native / Expo application, from first commit to production.

## When to activate this skill
- When starting a new React Native or Expo project
- When the codebase has grown beyond a basic to-do app structure
- When adding a feature that requires platform-specific behavior
- When deciding between Expo managed workflow and bare workflow

## Core principles
1. **Expo managed workflow first.** Start with managed workflow unless you have a confirmed need for a native module that Expo does not support.
2. **Feature folders, not layer folders.** Same as React web: `features/auth/`, `features/home/`, not `screens/` and `components/` at the root.
3. **Platform files for platform-specific code.** Use `.ios.ts` and `.android.ts` file extensions — React Native's resolver picks the right file automatically.
4. **All navigation config in one place.** Navigation structure is the application skeleton — it deserves its own top-level `navigation/` directory.
5. **Never import `react-native` primitives directly in feature components.** Wrap them in a design system (`components/ui/`) so you can swap platform implementations without touching feature code.

## Step-by-step guide
1. Initialize with Expo: `npx create-expo-app@latest --template tabs` for a tabs starter, or `--template blank-typescript` for clean start.
2. Directory structure:
   ```
   app/                 # Expo Router file-based routes (preferred in Expo SDK 50+)
   components/
     ui/                # Design system primitives (Text, Button, Card, Input)
   features/
     auth/              # Auth screens, hooks, services
     home/              # Home feature screens, hooks, services
   lib/                 # Third-party clients (supabase, axios instance, mmkv)
   hooks/               # Global hooks (useColorScheme, useSafeArea)
   store/               # Zustand stores for global client state
   types/               # Shared TypeScript types
   constants/           # Colors, sizes, breakpoints
   ```
3. Use Expo Router for navigation — file-based routing in `app/` directory mirrors the web `next.js` mental model.
4. Create platform-specific implementations using the `.native.ts`, `.ios.ts`, `.android.ts` extension convention.
5. For native modules: check Expo's built-in SDK first (`expo-camera`, `expo-notifications`, etc.) before reaching for React Native community modules.
6. For state: same rules as React web — server data in React Query, global UI state in Zustand, local in `useState`.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `screens/` folder at root with 50 screen files | `features/[name]/` grouping each feature's screens, hooks, and services |
| `if (Platform.OS === 'ios')` scattered throughout feature code | Platform-specific `.ios.tsx` / `.android.tsx` files |
| Start with bare workflow before trying managed | Start managed; eject only when a confirmed native module requirement appears |
| Inline styles in every component | `StyleSheet.create` in UI component files, or NativeWind (Tailwind for RN) |
| Custom navigation setup with `react-navigation` config files | Expo Router file-based routing for simpler, Next.js-familiar structure |

## Stack-specific notes
**Expo Managed Workflow:** Expo EAS Build handles native builds. No Xcode or Android Studio required for most features. Use `expo-dev-client` when you need custom native modules in managed workflow.
**React Navigation (without Expo Router):** Use typed navigation with `@react-navigation/native` + TypeScript. Define the RootStackParamList and use `NativeStackNavigator`.
**Node.js / Python (API):** The backend serves the mobile app. Ensure API responses are optimized for mobile (pagination, minimal payloads, proper caching headers).

## Common mistakes
1. **Putting navigation logic inside screen components.** Navigation config belongs in `app/` (Expo Router) or `navigation/`. Screen components call `router.push()` but do not define routes.
2. **Not handling safe area insets.** `SafeAreaView` or `useSafeAreaInsets()` must be applied to every screen — missing it cuts off content on notched devices.
3. **No design system layer.** Direct `<View>`, `<Text>`, `<TouchableOpacity>` imports in feature components create inconsistency and platform-specific issues.
4. **Ignoring the Android/iOS behavior difference.** Shadows, fonts, input behavior, keyboard avoidance — these all differ. Test on both platforms before merging.

## Checklist
- [ ] Expo managed workflow used unless confirmed native module requirement
- [ ] Feature-folder structure (`features/`, `components/ui/`)
- [ ] Expo Router used for file-based navigation
- [ ] Platform-specific files use `.ios.ts` / `.android.ts` extension convention
- [ ] All native primitives wrapped in UI component layer
- [ ] `SafeAreaView` or `useSafeAreaInsets` applied to every screen
- [ ] React Query for server state, Zustand for client state
- [ ] TypeScript enabled and strict
- [ ] EAS Build configured for CI/CD
- [ ] Tested on both iOS and Android
