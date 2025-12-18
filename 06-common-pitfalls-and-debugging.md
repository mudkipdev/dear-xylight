# Common Pitfalls and Debugging

Hey Xylight! Let's go over the most common issues you'll run into and how to fix them. Save yourself hours of debugging! 🛠️

## The #1 Mistake: Forgetting to Wait for Sync

**Problem:** You try to access rooms or send messages before the client has synced.

```typescript
// ❌ This will fail!
const client = sdk.createClient({...});
await client.initRustCrypto();
client.startClient();

// Trying to access rooms immediately
const rooms = client.getRooms();  // Empty array! Client hasn't synced yet
```

**Solution:** Always wait for the sync state:

```typescript
// ✅ This works!
import { ClientEvent } from 'matrix-js-sdk';

const client = sdk.createClient({...});
await client.initRustCrypto();

// Wait for sync to complete
await new Promise((resolve) => {
    client.once(ClientEvent.Sync, (state) => {
        if (state === 'PREPARED') {
            resolve();
        }
    });
    client.startClient();
});

// Now you can access rooms
const rooms = client.getRooms();  // Works! 🎉
```

## Pitfall #2: Missing userId or deviceId for Crypto

**Problem:** You forget to provide userId or deviceId when creating the client.

```typescript
// ❌ This will crash when you call initRustCrypto()
const client = sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "token",
});
await client.initRustCrypto();  // Error: unknown userId
```

**Solution:** Always provide both:

```typescript
// ✅ This works!
const client = sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "token",
    userId: "@xylight:matrix.org",
    deviceId: "ABCDEFG",
});
await client.initRustCrypto();  // Success!
```

## Pitfall #3: Calling initRustCrypto() After startClient()

**Problem:** You start the client before initializing crypto.

```typescript
// ❌ Wrong order!
const client = sdk.createClient({...});
await client.startClient();
await client.initRustCrypto();  // Might not work correctly
```

**Solution:** Always initialize crypto BEFORE starting:

```typescript
// ✅ Correct order!
const client = sdk.createClient({...});
await client.initRustCrypto();  // First!
await client.startClient();      // Then start
```

## Pitfall #4: Not Handling Decryption Failures

**Problem:** You assume all encrypted messages will decrypt successfully.

```typescript
// ❌ This might crash on encrypted messages
client.on('Room.timeline', (event) => {
    const body = event.getContent().body;  // Might be undefined!
    console.log(body.toUpperCase());  // Crash!
});
```

**Solution:** Check for decryption failures:

```typescript
// ✅ Safe handling
client.on('Room.timeline', (event) => {
    if (event.isDecryptionFailure()) {
        console.log("Unable to decrypt message");
        return;
    }

    const content = event.getContent();
    if (content.body) {
        console.log(content.body);
    }
});
```

## Pitfall #5: Memory Leaks from Event Listeners

**Problem:** You add event listeners but never remove them.

```svelte
<!-- ❌ Memory leak in SvelteKit! -->
<script lang="ts">
    import { onMount } from 'svelte';
    import { matrixClient } from '$lib/stores/matrix';

    onMount(() => {
        $matrixClient.on('Room.timeline', handleMessage);
        // Missing cleanup!
    });

    function handleMessage(event) {
        console.log(event);
    }
</script>
```

**Solution:** Always clean up event listeners:

```svelte
<!-- ✅ Proper cleanup -->
<script lang="ts">
    import { onMount } from 'svelte';
    import { matrixClient } from '$lib/stores/matrix';

    onMount(() => {
        $matrixClient.on('Room.timeline', handleMessage);

        // Clean up when component is destroyed
        return () => {
            $matrixClient.off('Room.timeline', handleMessage);
        };
    });

    function handleMessage(event) {
        console.log(event);
    }
</script>
```

## Pitfall #6: Accessing Rooms Before They're Ready

**Problem:** You try to get a specific room before sync completes.

```typescript
// ❌ Room might not be loaded yet
const room = client.getRoom(roomId);
console.log(room.name);  // Crash! room is null
```

**Solution:** Check if room exists:

```typescript
// ✅ Safe access
const room = client.getRoom(roomId);
if (!room) {
    console.log("Room not found or not loaded yet");
    return;
}
console.log(room.name);
```

## Debugging: Enable Verbose Logging

Can't figure out what's going wrong? Turn on debug logging:

```typescript
import sdk from 'matrix-js-sdk';

// Enable verbose logging
sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "token",
    userId: "@xylight:matrix.org",
    deviceId: "ABC",
    // Add this:
    useAuthorizationHeader: true,

    // Or for even more detail:
    logger: {
        log: console.log,
        warn: console.warn,
        error: console.error,
        debug: console.debug,
    }
});
```

In the browser console, you can also do:

```javascript
// Enable all matrix-js-sdk logs
localStorage.setItem('matrix-js-sdk:log', '*');

// Then refresh the page
```

## Debugging: Common Error Messages

### "Unknown userId" or "Unknown deviceId"

**Cause:** You didn't provide userId/deviceId when creating the client.

**Fix:**
```typescript
const client = sdk.createClient({
    baseUrl: homeserver,
    accessToken: token,
    userId: userId,      // Add this
    deviceId: deviceId,  // Add this
});
```

### "Cannot read properties of null"

**Cause:** Trying to access a room or event that hasn't loaded yet.

**Fix:** Wait for sync and check for null:
```typescript
client.once(ClientEvent.Sync, (state) => {
    if (state === 'PREPARED') {
        const room = client.getRoom(roomId);
        if (room) {
            // Safe to use room
        }
    }
});
```

### "M_FORBIDDEN" when sending messages

**Cause:** You don't have permission to send messages in that room.

**Fix:** Check your power level:
```typescript
const myPowerLevel = room.getMember(client.getUserId())?.powerLevel || 0;
console.log("My power level:", myPowerLevel);

// Check if you can send messages (usually need level 0 or higher)
const canSend = myPowerLevel >= 0;
```

### "Session expired" / 401 errors

**Cause:** Your access token is invalid or expired.

**Fix:** Catch auth errors and redirect to login:
```typescript
client.on(ClientEvent.Sync, (state, prevState, data) => {
    if (state === 'ERROR' && data?.error?.httpStatus === 401) {
        console.log("Session expired");
        authStore.logout();
        goto('/login');
    }
});
```

## Debugging: Inspecting Events

Want to see what's inside an event? Log it:

```typescript
client.on('Room.timeline', (event) => {
    console.log("Full event:", event.event);  // Raw event data
    console.log("Type:", event.getType());
    console.log("Sender:", event.getSender());
    console.log("Content:", event.getContent());
    console.log("Encrypted?", event.isEncrypted());
    console.log("Decryption failure?", event.isDecryptionFailure());
});
```

## Debugging: Check Encryption Status

Is encryption working? Check this:

```typescript
// After initRustCrypto()
const crypto = client.getCrypto();
if (crypto) {
    console.log("✅ Encryption is enabled!");
} else {
    console.log("❌ Encryption not initialized");
}

// Check if a specific room is encrypted
const isEncrypted = client.isRoomEncrypted(roomId);
console.log(`Room ${roomId} encrypted:`, isEncrypted);
```

## Performance: Don't Re-render on Every Event

**Problem:** Your UI re-renders on every message, causing lag.

```svelte
<!-- ❌ This re-renders the entire list on every message -->
<script lang="ts">
    let messages = [];

    $matrixClient.on('Room.timeline', (event) => {
        messages = [...messages, event];  // Triggers full re-render
    });
</script>

{#each messages as msg}
    <Message {msg} />
{/each}
```

**Solution:** Use keyed each blocks and stores:

```svelte
<!-- ✅ Only renders new messages -->
<script lang="ts">
    import { messages } from '$lib/stores/messages';  // Writable store
</script>

{#each $messages as msg (msg.getId())}
    <Message {msg} />
{/each}
```

## Common SvelteKit-Specific Issues

### Issue: "window is not defined"

**Cause:** Trying to access browser APIs during SSR.

**Fix:** Check if you're in the browser:
```typescript
import { browser } from '$app/environment';

if (browser) {
    // Only runs in browser
    const client = sdk.createClient({...});
}
```

### Issue: Stores not updating

**Cause:** Not using reactive statements or derived stores.

**Fix:**
```svelte
<script lang="ts">
    import { matrixClient } from '$lib/stores/matrix';

    // ✅ This is reactive
    $: rooms = $matrixClient?.getRooms() || [];

    // ❌ This is not reactive
    let rooms = $matrixClient?.getRooms() || [];
</script>
```

## Quick Debugging Checklist

When something's not working, check these in order:

1. ✅ Is the client created with userId and deviceId?
2. ✅ Did you call `initRustCrypto()` before `startClient()`?
3. ✅ Did you wait for sync state to be 'PREPARED'?
4. ✅ Are you checking for null/undefined before accessing properties?
5. ✅ Did you handle decryption failures?
6. ✅ Are you cleaning up event listeners?
7. ✅ Is verbose logging enabled?

## You're Doing Amazing, Xylight!

Debugging can be frustrating, but now you know the most common pitfalls and how to avoid them. Save this guide - you'll refer back to it often!

Remember:
- Always wait for sync
- Check for null/undefined
- Clean up event listeners
- Enable logging when stuck
- Read error messages carefully

You've got this! 🚀
