# Sending Messages

What's up, Xylight! Time to send some messages. Just like with receiving, encryption happens automatically - you don't need to think about it!

## Sending a Simple Text Message

The easiest way to send a message:

```typescript
const roomId = "!abc123:matrix.org";
const messageText = "Hello, world!";

await client.sendTextMessage(roomId, messageText);
```

That's it! If the room is encrypted, the SDK will automatically encrypt the message before sending.

## Sending Different Message Types

### Text Messages (the detailed way)

```typescript
import { MsgType } from 'matrix-js-sdk';

await client.sendMessage(roomId, {
    msgtype: MsgType.Text,
    body: "This is a text message!"
});
```

### Emotes (actions)

```typescript
// Sends "* Xylight waves hello"
await client.sendMessage(roomId, {
    msgtype: MsgType.Emote,
    body: "waves hello"
});
```

### Notices (bot messages, less intrusive)

```typescript
await client.sendMessage(roomId, {
    msgtype: MsgType.Notice,
    body: "This is a notice - won't trigger notifications as aggressively"
});
```

## Sending Images

Sending images is a two-step process: upload, then send.

```typescript
async function sendImage(roomId: string, file: File) {
    try {
        // Step 1: Upload the file
        const uploadResponse = await client.uploadContent(file, {
            name: file.name,
            type: file.type,
        });

        // Step 2: Send the message with the uploaded URL
        await client.sendMessage(roomId, {
            msgtype: MsgType.Image,
            body: file.name,
            url: uploadResponse.content_uri,
            info: {
                mimetype: file.type,
                size: file.size,
                // Optional: image dimensions
                w: 1920,
                h: 1080,
            }
        });

        console.log("Image sent! 📸");
    } catch (error) {
        console.error("Failed to send image:", error);
    }
}
```

## Sending Files

Very similar to images:

```typescript
async function sendFile(roomId: string, file: File) {
    try {
        const uploadResponse = await client.uploadContent(file, {
            name: file.name,
            type: file.type,
        });

        await client.sendMessage(roomId, {
            msgtype: MsgType.File,
            body: file.name,
            url: uploadResponse.content_uri,
            info: {
                mimetype: file.type,
                size: file.size,
            }
        });

        console.log("File sent! 📎");
    } catch (error) {
        console.error("Failed to send file:", error);
    }
}
```

## SvelteKit File Upload Example

Here's a complete example with a file input:

```svelte
<!-- src/routes/room/[roomId]/+page.svelte -->
<script lang="ts">
    import { page } from '$app/stores';
    import { matrixClient } from '$lib/stores/matrix';
    import { MsgType } from 'matrix-js-sdk';

    let fileInput: HTMLInputElement;
    let uploading = false;

    async function handleFileUpload(event: Event) {
        const target = event.target as HTMLInputElement;
        const file = target.files?.[0];

        if (!file || !$matrixClient) return;

        uploading = true;

        try {
            // Upload the file
            const response = await $matrixClient.uploadContent(file, {
                name: file.name,
                type: file.type,
            });

            // Determine message type based on file type
            let msgtype = MsgType.File;
            if (file.type.startsWith('image/')) {
                msgtype = MsgType.Image;
            } else if (file.type.startsWith('video/')) {
                msgtype = MsgType.Video;
            } else if (file.type.startsWith('audio/')) {
                msgtype = MsgType.Audio;
            }

            // Send the message
            await $matrixClient.sendMessage($page.params.roomId, {
                msgtype,
                body: file.name,
                url: response.content_uri,
                info: {
                    mimetype: file.type,
                    size: file.size,
                }
            });

            console.log("Upload successful! ✨");
        } catch (error) {
            console.error("Upload failed:", error);
            alert("Failed to upload file");
        } finally {
            uploading = false;
            // Clear the input
            if (fileInput) fileInput.value = '';
        }
    }
</script>

<input
    type="file"
    bind:this={fileInput}
    on:change={handleFileUpload}
    disabled={uploading}
/>

{#if uploading}
    <p>Uploading...</p>
{/if}
```

## Sending Formatted Messages (HTML)

Want bold, italic, or other formatting?

```typescript
await client.sendMessage(roomId, {
    msgtype: MsgType.Text,
    body: "This is **bold** text",  // Fallback for clients that don't support HTML
    format: "org.matrix.custom.html",
    formatted_body: "This is <strong>bold</strong> text"
});
```

## Sending Replies

To reply to a message:

```typescript
import { makeReply } from 'matrix-js-sdk';

// Get the event you want to reply to
const originalEvent = room.findEventById(eventId);

if (originalEvent) {
    const replyContent = makeReply(originalEvent, {
        msgtype: MsgType.Text,
        body: "This is my reply!"
    });

    await client.sendMessage(roomId, replyContent);
}
```

## Handling Send Failures

Messages can fail to send. Here's how to handle that:

```typescript
import { EventStatus } from 'matrix-js-sdk';

try {
    const result = await client.sendMessage(roomId, {
        msgtype: MsgType.Text,
        body: "Hello!"
    });

    // result.event_id is the ID of the sent message
    console.log("Message sent with ID:", result.event_id);

} catch (error) {
    console.error("Failed to send message:", error);

    // Check if it's a network error, server error, etc.
    if (error.name === 'ConnectionError') {
        console.log("No internet connection");
    } else if (error.httpStatus === 403) {
        console.log("You don't have permission to send messages here");
    }
}
```

## Tracking Send Status

You can watch the status of messages you send:

```typescript
client.on('Room.localEchoUpdated', (event, room, oldEventId, oldStatus) => {
    const status = event.status;

    switch (status) {
        case EventStatus.SENDING:
            console.log("Sending message...");
            break;

        case EventStatus.SENT:
            console.log("Message sent! ✓");
            break;

        case EventStatus.NOT_SENT:
            console.log("Message failed to send ✗");
            break;

        case EventStatus.QUEUED:
            console.log("Message queued (client is offline)");
            break;
    }
});
```

## Typing Indicators

Let users know when you're typing:

```typescript
let typingTimer: NodeJS.Timeout;

function handleInputChange(roomId: string) {
    // Send typing notification
    client.sendTyping(roomId, true, 3000); // typing for 3 seconds

    // Reset the timer
    clearTimeout(typingTimer);
    typingTimer = setTimeout(() => {
        client.sendTyping(roomId, false, 0);
    }, 3000);
}
```

## SvelteKit Message Input Example

Complete example with typing indicators:

```svelte
<script lang="ts">
    import { matrixClient } from '$lib/stores/matrix';
    import { page } from '$app/stores';

    let messageText = '';
    let sending = false;
    let typingTimer: NodeJS.Timeout;

    function handleInput() {
        if (!$matrixClient) return;

        const roomId = $page.params.roomId;

        // Send typing notification
        $matrixClient.sendTyping(roomId, true, 3000);

        // Stop typing after 3 seconds of inactivity
        clearTimeout(typingTimer);
        typingTimer = setTimeout(() => {
            $matrixClient.sendTyping(roomId, false, 0);
        }, 3000);
    }

    async function sendMessage() {
        if (!messageText.trim() || !$matrixClient || sending) return;

        const roomId = $page.params.roomId;
        sending = true;

        try {
            // Stop typing notification
            await $matrixClient.sendTyping(roomId, false, 0);

            // Send the message
            await $matrixClient.sendTextMessage(roomId, messageText);

            // Clear input
            messageText = '';
        } catch (error) {
            console.error("Failed to send:", error);
            alert("Failed to send message");
        } finally {
            sending = false;
        }
    }

    function handleKeydown(event: KeyboardEvent) {
        if (event.key === 'Enter' && !event.shiftKey) {
            event.preventDefault();
            sendMessage();
        }
    }
</script>

<div class="message-input">
    <input
        type="text"
        bind:value={messageText}
        on:input={handleInput}
        on:keydown={handleKeydown}
        placeholder="Type a message..."
        disabled={sending}
    />
    <button on:click={sendMessage} disabled={sending || !messageText.trim()}>
        {sending ? 'Sending...' : 'Send'}
    </button>
</div>
```

## Awesome Work, Xylight!

You're sending encrypted messages like a pro! The SDK handles all the encryption behind the scenes - you just call `sendMessage()` and it works.

Remember:
- `sendTextMessage()` for quick text messages
- `sendMessage()` for more control over message type
- Upload files first, then send with the `url`
- Typing indicators make your app feel more responsive

Keep building! 🚀
