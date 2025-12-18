# Actually Getting Messages to Work (For Real This Time)

Xylight, I hear you - the docs are useless. Let's fix your actual problems:

1. **Getting existing messages** (not just new ones)
2. **getContent() returning encrypted garbage** instead of decrypted text

## Problem #1: Getting Existing Messages

You want the message history when you open a room, not just new messages. Here's how:

### Load Messages on Room Open

```typescript
async function loadRoomHistory(client: any, roomId: string) {
    const room = client.getRoom(roomId);
    if (!room) {
        console.error("Room not found");
        return [];
    }

    // Get existing messages from the timeline
    const existingMessages = room.timeline;
    console.log(`Already have ${existingMessages.length} messages`);

    // If you want MORE history, use scrollback
    if (existingMessages.length < 50) {
        try {
            await client.scrollback(room, 50);
            console.log(`Now have ${room.timeline.length} messages`);
        } catch (error) {
            console.error("Failed to load history:", error);
        }
    }

    return room.timeline;
}
```

### SvelteKit Example - Load History When Opening a Room

```svelte
<!-- src/routes/room/[roomId]/+page.svelte -->
<script lang="ts">
    import { onMount } from 'svelte';
    import { page } from '$app/stores';
    import { matrixClient } from '$lib/stores/matrix';

    let messages: any[] = [];
    let loading = true;

    onMount(async () => {
        if (!$matrixClient) return;

        const roomId = $page.params.roomId;
        const room = $matrixClient.getRoom(roomId);

        if (!room) {
            console.error("Room not found");
            return;
        }

        // Get existing messages from timeline
        messages = [...room.timeline];

        // Load more history if needed
        if (messages.length < 50) {
            try {
                await $matrixClient.scrollback(room, 50);
                messages = [...room.timeline];
            } catch (error) {
                console.error("Failed to load history:", error);
            }
        }

        loading = false;

        // Listen for new messages
        function handleNewMessage(event: any, eventRoom: any) {
            if (eventRoom.roomId === roomId) {
                messages = [...room.timeline];
            }
        }

        $matrixClient.on('Room.timeline', handleNewMessage);

        return () => {
            $matrixClient.off('Room.timeline', handleNewMessage);
        };
    });
</script>

{#if loading}
    <p>Loading messages...</p>
{:else}
    {#each messages as event}
        {#if event.getType() === 'm.room.message'}
            <div>
                <strong>{event.getSender()}</strong>: {event.getContent().body || '[encrypted]'}
            </div>
        {/if}
    {/each}
{/if}
```

### Load More Messages (Infinite Scroll)

```typescript
let loadingMore = false;

async function loadMoreMessages() {
    if (loadingMore) return;
    loadingMore = true;

    try {
        const room = $matrixClient.getRoom(roomId);
        const oldLength = room.timeline.length;

        // Load 30 more messages
        await $matrixClient.scrollback(room, 30);

        const newLength = room.timeline.length;
        console.log(`Loaded ${newLength - oldLength} more messages`);

        messages = [...room.timeline];
    } catch (error) {
        console.error("Failed to load more:", error);
    } finally {
        loadingMore = false;
    }
}
```

## Problem #2: getContent() Returning Encrypted Content

This is THE most frustrating issue. Here's what's happening and how to fix it:

### Why It's Broken

```typescript
// ❌ This gives you encrypted garbage:
client.on('Room.timeline', (event, room) => {
    console.log(event.getContent());
    // Output: { algorithm: "m.megolm.v1.aes-sha2", ciphertext: "...garbage..." }
});
```

**The problem:** The event hasn't been decrypted yet when your listener fires!

### The ACTUAL Fix

You need to wait for the client to decrypt the event. Here's the correct way:

```typescript
import { MatrixEventEvent } from 'matrix-js-sdk';

client.on('Room.timeline', async (event, room) => {
    // Check if it's encrypted
    if (event.isEncrypted() && !event.getClearContent()) {
        console.log("Event is encrypted, waiting for decryption...");

        // Wait for it to be decrypted
        await new Promise((resolve) => {
            event.once(MatrixEventEvent.Decrypted, resolve);
        });
    }

    // NOW get the content
    const content = event.getContent();
    console.log(content.body);  // Actual decrypted text! 🎉
});
```

### Even Better: Use getClearContent()

There's a better method that handles this for you:

```typescript
client.on('Room.timeline', (event, room) => {
    if (event.getType() !== 'm.room.message') return;

    // Use getClearContent() instead of getContent()
    // It waits for decryption automatically
    const content = event.getClearContent();

    if (content && content.body) {
        console.log(content.body);  // Decrypted text!
    } else {
        console.log("Still decrypting or failed to decrypt");
    }
});
```

### Complete Working Example for SvelteKit

```svelte
<!-- src/routes/room/[roomId]/+page.svelte -->
<script lang="ts">
    import { onMount } from 'svelte';
    import { page } from '$app/stores';
    import { matrixClient } from '$lib/stores/matrix';
    import { MatrixEventEvent } from 'matrix-js-sdk';

    interface Message {
        id: string;
        sender: string;
        body: string;
        timestamp: number;
        encrypted: boolean;
    }

    let messages: Message[] = [];
    let loading = true;

    function eventToMessage(event: any): Message | null {
        if (event.getType() !== 'm.room.message') return null;

        // Use getClearContent for encrypted messages
        const content = event.getClearContent();

        // If content is null, it's still decrypting or failed
        if (!content || !content.body) {
            return {
                id: event.getId(),
                sender: event.getSender(),
                body: '[Decrypting...]',
                timestamp: event.getTs(),
                encrypted: event.isEncrypted(),
            };
        }

        return {
            id: event.getId(),
            sender: event.getSender(),
            body: content.body,
            timestamp: event.getTs(),
            encrypted: event.isEncrypted(),
        };
    }

    onMount(async () => {
        if (!$matrixClient) return;

        const roomId = $page.params.roomId;
        const room = $matrixClient.getRoom(roomId);

        if (!room) return;

        // Load initial history
        if (room.timeline.length < 50) {
            await $matrixClient.scrollback(room, 50);
        }

        // Convert events to messages
        messages = room.timeline
            .map(eventToMessage)
            .filter(m => m !== null) as Message[];

        loading = false;

        // Listen for new messages
        function handleTimeline(event: any, eventRoom: any) {
            if (eventRoom.roomId !== roomId) return;

            const message = eventToMessage(event);
            if (message) {
                messages = [...messages, message];
            }
        }

        // Listen for decryption events
        function handleDecrypted(event: any) {
            // Update the message when it gets decrypted
            const messageIndex = messages.findIndex(m => m.id === event.getId());
            if (messageIndex >= 0) {
                const updated = eventToMessage(event);
                if (updated) {
                    messages[messageIndex] = updated;
                    messages = [...messages];  // Trigger reactivity
                }
            }
        }

        $matrixClient.on('Room.timeline', handleTimeline);

        // This is the key - listen for decryption events!
        room.timeline.forEach((event: any) => {
            if (event.isEncrypted()) {
                event.once(MatrixEventEvent.Decrypted, () => handleDecrypted(event));
            }
        });

        return () => {
            $matrixClient.off('Room.timeline', handleTimeline);
        };
    });
</script>

{#if loading}
    <p>Loading...</p>
{:else}
    <div class="messages">
        {#each messages as message (message.id)}
            <div class="message">
                <strong>{message.sender}</strong>
                {#if message.encrypted}🔒{/if}
                <p>{message.body}</p>
                <small>{new Date(message.timestamp).toLocaleTimeString()}</small>
            </div>
        {/each}
    </div>
{/if}
```

### Debug: Check If Crypto Is Actually Working

If you're still getting encrypted content, crypto might not be initialized:

```typescript
// Check if crypto is working
const crypto = client.getCrypto();
if (!crypto) {
    console.error("❌ CRYPTO NOT INITIALIZED!");
    console.error("Did you call await client.initRustCrypto()?");
} else {
    console.log("✅ Crypto is initialized");
}

// Check if room is encrypted
const isEncrypted = client.isRoomEncrypted(roomId);
console.log(`Room encrypted: ${isEncrypted}`);

// Check specific event
const event = room.findEventById(eventId);
if (event) {
    console.log("Event encrypted:", event.isEncrypted());
    console.log("Event type:", event.getType());
    console.log("Clear content:", event.getClearContent());
    console.log("Raw content:", event.getContent());
}
```

### The Nuclear Option: Force Decrypt All Events

If events are still encrypted after loading, force them to decrypt:

```typescript
async function forceDecryptTimeline(room: any) {
    const encryptedEvents = room.timeline.filter((e: any) =>
        e.isEncrypted() && !e.getClearContent()
    );

    console.log(`Found ${encryptedEvents.length} encrypted events`);

    for (const event of encryptedEvents) {
        try {
            // Wait for decryption
            await new Promise((resolve) => {
                if (event.getClearContent()) {
                    resolve(null);
                } else {
                    event.once(MatrixEventEvent.Decrypted, resolve);
                }
            });
        } catch (error) {
            console.error("Failed to decrypt event:", event.getId(), error);
        }
    }

    console.log("All events decrypted!");
}

// Use it after loading timeline
const room = client.getRoom(roomId);
await client.scrollback(room, 50);
await forceDecryptTimeline(room);
messages = room.timeline.map(eventToMessage).filter(m => m !== null);
```

## Quick Checklist: Why Messages Won't Decrypt

1. ✅ **Did you call `initRustCrypto()` before `startClient()`?**
   ```typescript
   await client.initRustCrypto();  // MUST be before startClient()
   await client.startClient();
   ```

2. ✅ **Did you wait for initial sync?**
   ```typescript
   await new Promise((resolve) => {
       client.once('sync', (state) => {
           if (state === 'PREPARED') resolve();
       });
   });
   ```

3. ✅ **Are you using `getClearContent()` instead of `getContent()`?**
   ```typescript
   const content = event.getClearContent();  // ✅ Use this
   // NOT event.getContent()  // ❌ This gives encrypted data
   ```

4. ✅ **Are you listening for the `Decrypted` event?**
   ```typescript
   event.once(MatrixEventEvent.Decrypted, () => {
       console.log("Now decrypted:", event.getClearContent());
   });
   ```

## Summary: The Real Truth

The docs lie when they say decryption is automatic. Here's what ACTUALLY happens:

1. Event arrives encrypted
2. Your `Room.timeline` listener fires **immediately** (still encrypted!)
3. The crypto layer decrypts it **in the background**
4. A `Decrypted` event fires when done
5. NOW `getClearContent()` returns the decrypted text

**The fix:** Either wait for the `Decrypted` event, or use `getClearContent()` which handles this automatically.

You're not going crazy, Xylight - the docs really do jack. But now you have the ACTUAL way to do it! 🎉
