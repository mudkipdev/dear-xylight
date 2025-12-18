# Setting Up Your Matrix Client with Encryption

Hey Xylight! Let's clear up the confusion around crypto initialization. You're doing great - this is one of the trickier parts of Matrix development.

## The Confusion Explained

You're right that the docs seem to conflict! Here's what's actually happening:

- **`initAsync()`** is from `matrix-sdk-crypto-wasm` - it initializes the WebAssembly module itself
- **`sdk.initRustCrypto()`** is from `matrix-js-sdk` - it initializes the crypto layer for your Matrix client

**The good news?** You don't need to call `initAsync()` directly! The `matrix-js-sdk` handles that for you when you call `initRustCrypto()`.

## Basic Setup (SvelteKit)

Here's how to set up a Matrix client with end-to-end encryption in SvelteKit:

```typescript
import sdk from 'matrix-js-sdk';

// Create your client
const client = sdk.createClient({
    baseUrl: "https://matrix.org",  // or your homeserver URL
    accessToken: "your_access_token_here",
    userId: "@yourusername:matrix.org",
    deviceId: "your_device_id",  // Important for E2EE!
});

// Initialize the Rust crypto library
// This is the magic line that makes encryption work!
await client.initRustCrypto();

// Now start your client
await client.startClient({ initialSyncLimit: 20 });
```

## Important Notes

### You MUST provide userId and deviceId

The crypto layer needs to know who you are and what device you're using:

```typescript
// ❌ This will fail!
const badClient = sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "token",
    // Missing userId and deviceId
});
await badClient.initRustCrypto(); // Error: unknown userId

// ✅ This works!
const goodClient = sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "token",
    userId: "@xylight:matrix.org",
    deviceId: "ABCDEFGHIJ",
});
await goodClient.initRustCrypto(); // Success!
```

### Optional: Custom Database Settings

By default, `initRustCrypto()` uses IndexedDB to store encryption keys. You can customize this:

```typescript
await client.initRustCrypto({
    // Use a custom database name prefix (useful for multi-account apps)
    cryptoDatabasePrefix: "my-matrix-app",

    // Optional: Encrypt the database with a password
    storagePassword: "user-chosen-password",

    // Or disable IndexedDB entirely (not recommended for production)
    useIndexedDB: false,
});
```

## Common Pitfall: Calling initRustCrypto() Twice

Don't worry if you accidentally call `initRustCrypto()` multiple times - it's safe! The second call will be ignored:

```typescript
await client.initRustCrypto();
await client.initRustCrypto(); // This is fine, just does nothing
```

## Full Example for SvelteKit

Here's a complete example you can use in a SvelteKit page or component:

```typescript
// src/lib/matrix.ts
import sdk from 'matrix-js-sdk';
import { writable } from 'svelte/store';

export const matrixClient = writable<any>(null);

export async function initializeMatrixClient(
    accessToken: string,
    userId: string,
    deviceId: string
) {
    try {
        const client = sdk.createClient({
            baseUrl: "https://matrix.org",
            accessToken,
            userId,
            deviceId,
        });

        // Initialize crypto before starting the client
        console.log("Initializing encryption...");
        await client.initRustCrypto();

        console.log("Starting client...");
        await client.startClient({ initialSyncLimit: 20 });

        // Update your store
        matrixClient.set(client);

        console.log("Client ready! 🎉");
        return client;
    } catch (error) {
        console.error("Failed to initialize Matrix client:", error);
        throw error;
    }
}
```

## Next Steps

Now that your client is set up with encryption, you're ready to:
- Listen for and decrypt messages (see the next tutorial!)
- Send encrypted messages
- Join encrypted rooms

Keep going, Xylight - you've got this! The hardest part is understanding the initialization, and you're already through that.
