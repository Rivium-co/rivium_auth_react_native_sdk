<p align="center">
  <a href="https://rivium.co">
    <img src="https://rivium.co/logo.png" alt="Rivium" width="120" />
  </a>
</p>

<h3 align="center">Rivium Auth React Native SDK</h3>

<p align="center">
  Hosted sign-in for your React Native app's users. The SDK keeps the session fresh and gives the other Rivium SDKs a user token.
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/@rivium/auth-react-native"><img src="https://img.shields.io/npm/v/@rivium/auth-react-native.svg" alt="npm" /></a>
  <img src="https://img.shields.io/badge/React_Native-0.70+-61DAFB?logo=react&logoColor=black" alt="React Native 0.70+" />
  <img src="https://img.shields.io/badge/Expo-supported-000020?logo=expo&logoColor=white" alt="Expo supported" />
  <img src="https://img.shields.io/badge/TypeScript-5+-3178C6?logo=typescript&logoColor=white" alt="TypeScript 5+" />
  <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="MIT License" />
</p>

---

## Features

- **Six sign-in methods**: anonymous, Apple, Google, e-mail + password, magic link and Telegram
- **Upgrade guests**: `linkApple`, `linkGoogle` and `linkPassword` keep the user id
- **Secure storage**: the refresh token stays on this device (Keychain / Keystore); the access token lives in memory
- **Expo or bare**: `react-native-keychain` or `expo-secure-store`
- **Fresh tokens**: the access token is renewed before it expires; concurrent calls share one renewal
- **One user for every Rivium product**: hand the token to Rivium Sync, Chat, Push and A/B Testing
- **Same API as the web SDK**, pure TypeScript, no native code of its own

## Installation

```bash
npm install @rivium/auth-react-native react-native-keychain
# or, with Expo
npx expo install @rivium/auth-react-native expo-secure-store
```

For Apple / Google sign-in nonces you also need `crypto.getRandomValues`:

```bash
npm install react-native-get-random-values
```

```typescript
// index.js, before anything else
import 'react-native-get-random-values';
```

## Quick start

Turn on Rivium Auth and the sign-in methods you want in [Rivium Console](https://console.rivium.co), then:

```typescript
import * as Keychain from 'react-native-keychain';
import { createRiviumAuth, keychainStorage } from '@rivium/auth-react-native';

export const auth = createRiviumAuth({
  apiKey: 'rv_live_YOUR_PUBLIC_KEY',
  storage: keychainStorage(Keychain),
});

auth.onAuthStateChanged((userId) => {
  console.log(userId ? `signed in as ${userId}` : 'signed out');
});

await auth.signInAnonymously();
```

With Expo:

```typescript
import * as SecureStore from 'expo-secure-store';
import { createRiviumAuth, secureStoreStorage } from '@rivium/auth-react-native';

export const auth = createRiviumAuth({ apiKey: 'rv_live_YOUR_PUBLIC_KEY', storage: secureStoreStorage(SecureStore) });
```

Only the refresh token is stored (Keychain on iOS, Keystore-backed storage on Android, this device only). The access
token stays in memory. Without a `storage`, nothing survives an app restart.

Show only the methods your project has turned on:

```typescript
const { methods } = await auth.getMethods(); // e.g. ['anonymous', 'apple', 'google']
```

## Use with other Rivium products

The signed-in user's access token is the standard Rivium user token, so the other Rivium SDKs act as that user and
Sync security rules see it as `auth.uid`. No backend needed.

- **Rivium Sync, Push and A/B Testing** accept a provider that returns `null` while nobody is signed in: pass
  `() => auth.getToken()`.
- **Rivium Chat** needs a provider that always returns a token: pass `auth.tokenProvider`, which rejects with
  `UNAUTHENTICATED` while signed out, and create the Chat client after sign-in.

```typescript
import RiviumSync from '@rivium/sync-react-native';
import RiviumPush from 'rivium-push-react-native';
import RiviumAbTesting from 'rivium-ab-testing-react-native';
import { RiviumChatClient } from '@rivium/react-native-chat';

// Rivium Sync, Push and A/B Testing: null while signed out
await RiviumSync.init({ apiKey: 'rv_live_YOUR_PUBLIC_KEY', tokenProvider: () => auth.getToken() });
await RiviumPush.init({ apiKey: 'rv_live_YOUR_PUBLIC_KEY', tokenProvider: () => auth.getToken() });
await RiviumAbTesting.init({ apiKey: 'rv_live_YOUR_PUBLIC_KEY', tokenProvider: () => auth.getToken() });

// Rivium Chat: create the client after sign-in
const chat = new RiviumChatClient({
  apiKey: 'rv_live_YOUR_PUBLIC_KEY',
  userId: auth.currentUserId!,
  tokenProvider: auth.tokenProvider,
});
```

When the user signs in or out, ask the other SDKs to fetch a new token (for example
`RiviumSync.refreshUserToken()` and `RiviumAbTesting.refreshUserToken()`), ideally from `onAuthStateChanged`. More:
[Use with other Rivium products](https://rivium.co/cloud/rivium-auth/docs/other-products).

## Sign-in methods

```typescript
// E-mail + password
await auth.signUpWithEmail({ email, password, redirectUrl: 'myapp://verify' });
await auth.signInWithEmail({ email, password });
await auth.sendPasswordReset({ email, redirectUrl: 'myapp://reset' });
await auth.resetPassword({ token, password });          // token from the link's ?token=
await auth.verifyEmail({ token });

// Magic link
await auth.sendMagicLink({ email, redirectUrl: 'myapp://magic' });
await auth.signInWithMagicLink({ token });

// Telegram
const { sent, botUrl } = await auth.requestTelegramCode({ phone: '+37491234567' });
await auth.verifyTelegramCode({ phone: '+37491234567', code: '123456' });
```

`redirectUrl` must start with one of the redirect URLs allowed in your project's Rivium Auth settings.

### Apple and Google

Run the native sign-in with a nonce from the SDK, then hand its ID token to Rivium:

```typescript
const { rawNonce, hashedNonce } = await auth.createNonce();
// Give hashedNonce to your Apple / Google sign-in library as the `nonce`.
await auth.signInWithApple({ idToken, rawNonce, authorizationCode });
await auth.signInWithGoogle({ idToken, rawNonce });
```

### Upgrading an anonymous user

Linking keeps the user id, so the user's data stays theirs:

```typescript
await auth.linkApple({ idToken, rawNonce });
await auth.linkGoogle({ idToken, rawNonce });
await auth.linkPassword({ email, password });
```

If that Apple or Google account already belongs to another user, the call fails with `PROVIDER_ALREADY_LINKED`.

## Account

```typescript
const me = await auth.getMe();     // { id, anonymous, emailVerified, createdAt, providers }
await auth.signOut();              // this device
await auth.signOutEverywhere();    // every device
await auth.deleteAccount();        // deletes the user, then signs out (required by the App Store)
```

## Errors

Every failure is a `RiviumAuthError` with a `code`:

```typescript
import { isRiviumAuthError } from '@rivium/auth-react-native';

try {
  await auth.signInWithEmail({ email, password });
} catch (e) {
  if (isRiviumAuthError(e) && e.code === 'INVALID_CREDENTIALS') showError('Wrong e-mail or password');
  if (isRiviumAuthError(e) && e.code === 'RATE_LIMITED') retryIn(e.retryAfterSec);
}
```

Common codes: `INVALID_CREDENTIALS`, `EMAIL_TAKEN`, `WEAK_PASSWORD`, `EMAIL_NOT_VERIFIED`, `PROVIDER_ALREADY_LINKED`,
`METHOD_DISABLED`, `RATE_LIMITED`, `MAU_LIMIT`, `PRODUCT_DISABLED`, `NETWORK_ERROR`.

## Sessions

- The access token is refreshed 2 minutes before it expires. Calls made at the same time share one refresh.
- A revoked or expired session signs the user out (`onAuthStateChanged(null)`). A network error does not.

## Example app

[`example-app/`](example-app) is a React Native app (iOS and Android) with anonymous sign-in, e-mail sign-up and
sign-in, sign-out, account deletion and the current user id. It uses this SDK from the repository
(`"@rivium/auth-react-native": "file:.."`). See [example-app/README.md](example-app/README.md) to run it.

## Documentation

- [Rivium Auth Docs](https://rivium.co/cloud/rivium-auth/docs)
- [React Native SDK reference](https://rivium.co/cloud/rivium-auth/docs/sdks-react-native)
- [Use with other Rivium products](https://rivium.co/cloud/rivium-auth/docs/other-products)
- [Rivium Console](https://console.rivium.co)

## Support

- **Documentation**: [Rivium Auth Docs](https://rivium.co/cloud/rivium-auth/docs)
- **Issues**: [GitHub Issues](https://github.com/Rivium-co/rivium_auth_react_native_sdk/issues)
- **Email**: support@rivium.co

## License

MIT License - see [LICENSE](LICENSE) for details.
