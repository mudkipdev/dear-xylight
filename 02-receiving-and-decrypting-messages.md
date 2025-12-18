# Receiving and Decrypting Messages

Hey Xylight! Now that your client is set up with crypto, let's get those encrypted messages flowing. This is where the magic happens!

## ⚠️ IMPORTANT UPDATE

**If `getContent()` is returning encrypted garbage instead of decrypted text, skip to [Tutorial #8](./08-the-ACTUAL-truth-about-decryption.md)!** That tutorial has the REAL solution using `client.decryptEventIfNeeded()`.

The examples below work for *new* messages, but there's a gotcha with existing/historical messages.

## How Message Decryption Works

The great news: **decryption happens automatically for new messages!** When you listen to timeline events, the SDK decrypts them for you before passing them to your event handler.

However, for **existing messages** in the timeline, decryption happens asynchronously - see Tutorial #7 for details.

## Listening for Messages

The main event you want to listen to is `RoomEvent.Timeline`:

```typescript
import { RoomEvent } from 'matrix-js-sdk';

client.on(RoomEvent.Timeline, (event, room, toStartOfTimeline) => {
    // Skip pagination events (we only want new messages)
    if (toStartOfTimeline) return;

    // At this point, encrypted messages are ALREADY decrypted!
    const messageType = event.getType();
    const sender = event.getSender();
    const content = event.getContent();

    if (messageType === "m.room.message") {
        console.log(`${sender}: ${content.body}`);
    }
});
```

## Understanding the Event Object

Here's what you can get from a message event:

```typescript
client.on(RoomEvent.Timeline, (event, room) => {
    // Basic info
    const eventId = event.getId();           // "$abc123..."
    const sender = event.getSender();         // "@alice:matrix.org"
    const timestamp = event.getTs();          // 1699564234000
    const eventType = event.getType();        // "m.room.message"

    // Room info
    const roomId = room.roomId;               // "!xyz:matrix.org"
    const roomName = room.name;               // "My Cool Room"

    // Message content (already decrypted!)
    const content = event.getContent();

    if (eventType === "m.room.message") {
        const messageBody = content.body;     // "Hello, world!"
        const messageType = content.msgtype;  // "m.text", "m.image", etc.

        console.log(`New message in ${roomName}: ${messageBody}`);
    }
});
```

## Complete SvelteKit Example

Here's a reactive store-based approach that works great with SvelteKit:

```typescript
// src/lib/stores/messages.ts
import { writable, derived } from 'svelte/store';
import type { MatrixEvent, Room } from 'matrix-js-sdk';

interface Message {
    id: string;
    sender: string;
    body: string;
    timestamp: number;
    roomId: string;
}

export const messages = writable<Message[]>([]);

export function setupMessageListener(client: any) {
    client.on('Room.timeline', (event: MatrixEvent, room: Room, toStartOfTimeline: boolean) => {
        if (toStartOfTimeline) return;

        const eventType = event.getType();

        if (eventType === 'm.room.message') {
            const content = event.getContent();

            const message: Message = {
                id: event.getId()!,
                sender: event.getSender()!,
                body: content.body,
                timestamp: event.getTs(),
                roomId: room.roomId,
            };

            messages.update(msgs => [...msgs, message]);
        }
    });
}
```

Then in your Svelte component:

```svelte
<!-- src/routes/room/[roomId]/+page.svelte -->
<script lang="ts">
    import { messages } from '$lib/stores/messages';
    import { page } from '$app/stores';

    $: roomMessages = $messages.filter(m => m.roomId === $page.params.roomId);
</script>

<div class="messages">
    {#each roomMessages as message}
        <div class="message">
            <span class="sender">{message.sender}</span>
            <span class="body">{message.body}</span>
            <span class="time">{new Date(message.timestamp).toLocaleTimeString()}</span>
        </div>
    {/each}
</div>
```

## Checking if a Message is Encrypted

Want to know if a message was encrypted? Easy:

```typescript
client.on('Room.timeline', (event, room) => {
    const wasEncrypted = event.isEncrypted();

    if (wasEncrypted) {
        console.log("This message came through E2EE! 🔒");

        // Get encryption details
        const encryptionInfo = event.getWireContent();
        console.log("Algorithm:", encryptionInfo.algorithm);
        // Usually "m.megolm.v1.aes-sha2" for room messages
    }

    // Get the decrypted content as normal
    const content = event.getContent();
    console.log("Decrypted message:", content.body);
});
```

## Handling Different Message Types

Not all messages are plain text! Here's how to handle different types:

```typescript
client.on('Room.timeline', (event, room) => {
    const content = event.getContent();
    const msgtype = content.msgtype;

    switch (msgtype) {
        case 'm.text':
            console.log("Text message:", content.body);
            break;

        case 'm.image':
            const imageUrl = client.mxcUrlToHttp(content.url);
            console.log("Image:", imageUrl);
            break;

        case 'm.file':
            const fileUrl = client.mxcUrlToHttp(content.url);
            console.log("File:", content.body, fileUrl);
            break;

        case 'm.emote':
            console.log(`* ${event.getSender()} ${content.body}`);
            break;

        default:
            console.log("Unknown message type:", msgtype);
    }
});
```

## What if Decryption Fails?

Sometimes you'll receive a message that can't be decrypted (missing keys, etc.). Here's how to handle it:

```typescript
client.on('Room.timeline', (event, room) => {
    if (event.isDecryptionFailure()) {
        console.error("Failed to decrypt message!");

        // You can still show something to the user
        const errorContent = event.getContent();
        console.log("Error:", errorContent.msgtype); // "m.bad.encrypted"

        // Show a placeholder like "Unable to decrypt"
        return;
    }

    // Normal decrypted message handling
    const content = event.getContent();
    console.log(content.body);
});
```

## Loading Message History

Want to load older messages? Use `scrollback`:

```typescript
async function loadOlderMessages(room: Room, limit: number = 50) {
    try {
        await client.scrollback(room, limit);
        console.log(`Loaded ${limit} older messages`);
    } catch (error) {
        console.error("Failed to load history:", error);
    }
}
```

## Pro Tip: Wait for Sync

Make sure your client has synced before trying to access rooms:

```typescript
import { ClientEvent } from 'matrix-js-sdk';

client.once(ClientEvent.Sync, (state) => {
    if (state === 'PREPARED') {
        console.log("Client is synced and ready! 🚀");

        // Now you can safely access rooms
        const rooms = client.getRooms();
        console.log(`You're in ${rooms.length} rooms`);
    }
});

await client.startClient();
```

## You're Crushing It, Xylight!

Decryption is totally automatic once you've called `initRustCrypto()`. The SDK handles all the complex crypto stuff - you just listen for events and use the decrypted content. Pretty sweet, right?

Next up: sending your own encrypted messages!
