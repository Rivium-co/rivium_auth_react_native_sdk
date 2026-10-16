## [0.1.0] - 2026-10-16

- Initial release: anonymous, Apple, Google, e-mail + password, magic link and Telegram sign-in; linking a method to
  the signed-in user; `getMe`, `signOut`, `signOutEverywhere`, `deleteAccount`.
- Sessions refresh themselves (one request for concurrent callers); rotated refresh tokens are saved.
- Works with the other Rivium SDKs: `() => auth.getToken()` for Rivium Sync, Push and A/B Testing (null while signed
  out), `auth.tokenProvider` for Rivium Chat (rejects while signed out).
- Refresh token in the Keychain / Keystore with `keychainStorage` (react-native-keychain) or `secureStoreStorage` (expo-secure-store); the access token stays in memory.
- Example app in `example-app/` (iOS and Android).
