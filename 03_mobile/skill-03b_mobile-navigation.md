# SKILL 03-B — How to Handle Navigation in Mobile Apps
Category: Mobile Development
Applies to: Expo Router, React Navigation v6, React Native

## What this skill covers
Navigation is the skeleton of a mobile app. Incorrect navigation setup causes back button failures, auth state persistence bugs, deep link errors, and lost scroll position. This skill covers how to structure navigation with Expo Router (preferred) or React Navigation, how to protect routes with auth guards, how to deep link correctly, and how to pass typed params between screens.

## When to activate this skill
- When setting up navigation for any new React Native app
- When adding deep linking to an existing app
- When implementing an auth flow that redirects based on login state
- When params passed between screens are losing their types

## Core principles
1. **Navigation state is the source of truth for screen flow.** Never use React state to decide which screen to show — use navigation guards and redirects.
2. **Typed params everywhere.** Every route's params must be typed and validated — untyped navigation params are a source of runtime crashes.
3. **Auth guard at the navigation level, not the screen level.** Protect entire navigator groups, not individual screens.
4. **Back button behavior must be intentional.** Every modal, auth screen, and confirmation screen has specific back behavior — define it explicitly.
5. **Deep links require testing.** Deep links break in subtle ways. Test them on device before shipping.

## Step-by-step guide

### Using Expo Router (Recommended):
1. File-based routing mirrors Next.js: `app/(tabs)/home.tsx` → `/home`, `app/(auth)/login.tsx` → `/login`.
2. Use route groups `(group)` for layout sharing without URL segments: `(tabs)/` for tab navigation, `(auth)/` for auth screens.
3. Protect authenticated routes using a redirect in the layout:
   ```tsx
   // app/(app)/_layout.tsx
   const { user } = useAuth();
   if (!user) return <Redirect href="/login" />;
   ```
4. Type-safe navigation: Expo Router generates types automatically. Use `href` for links and `router.push('/path')` with typed params.
5. Deep links are automatic in Expo Router — configure the scheme in `app.json`: `"scheme": "myapp"`.

### Using React Navigation:
1. Define the RootStackParamList type for complete type safety.
2. Group screens in navigators: `BottomTabNavigator` for main app, `StackNavigator` for auth flow.
3. Auth guard: check `isAuthenticated` in the root navigator and render auth stack or app stack accordingly.
4. Pass params: `navigation.navigate('Profile', { userId: 'abc' })` — typed by the param list.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `if (!user) return null` inside a screen component | Auth guard in `_layout.tsx` that redirects to `/login` |
| Pass params as untyped objects | Define typed route params and use `{ params }` from typed hooks |
| Hard-navigate to login on every auth check | Conditional rendering in the root layout — keeps nav stack clean |
| No back button handling on modals | Set `presentation: 'modal'` and configure gestureEnabled correctly |
| Test deep links only in simulator | Test deep links on physical device with `npx uri-scheme open` |

## Stack-specific notes
**Expo Router:** Deep link scheme configured in `app.json`. Universal links require Apple/Google signing file at domain level. Use `expo-linking` for in-code deep link handling.
**React Navigation:** Define `linking` config object with screen path mapping. Test with `Linking.openURL('myapp://profile/123')`.
**Node.js / Python (API):** Not applicable, but ensure your API supports the data structures anticipated by deep link target screens.

## Common mistakes
1. **Auth check every individual screen.** Auth guard belongs at the navigator group level — one check protects all screens in the group.
2. **Not handling the initial route.** When the app opens from a deep link, the initial route must be set correctly or the user lands on the wrong screen.
3. **Mutating navigation params.** Params should be treated as read-only. Never mutate them from the receiving screen.
4. **Using `navigation.navigate()` instead of `navigation.replace()` for auth redirects.** After logout, use `replace` so the user cannot go back to the authenticated screen with the back button.

## Checklist
- [ ] File-based routing with Expo Router or typed RootStackParamList with React Navigation
- [ ] Auth guard at navigator group level (not individual screens)
- [ ] All route params typed — no `any` in navigation types
- [ ] Deep link scheme configured in `app.json`
- [ ] Back button behavior handled explicitly for modals and auth screens
- [ ] `router.replace()` used for auth redirects (not `router.push()`)
- [ ] Navigation structure reflects the app's UX intended flow
- [ ] Deep links tested on physical device
- [ ] Tab bar and stack navigators nested correctly
- [ ] Splash screen shown during initial auth check (not a flash of wrong screen)
