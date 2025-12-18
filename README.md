# Matrix Client Development Guide for SvelteKit

Hey Xylight! 👋

Welcome to your personal Matrix development guide. These tutorials will help you build an awesome Matrix client with SvelteKit, matrix-js-sdk, and matrix-wasm-crypto-sdk.

I know the docs can be confusing (especially around crypto initialization - we'll clear that up!), so I've created these guides specifically for you. Each one is designed to be practical and easy to follow.

## 📚 Tutorials

Work through these in order - each builds on the previous one:

### [1. Setting Up Client with Crypto](./01-setting-up-client-with-crypto.md)
**Start here!** Clears up the confusion between `initAsync()` and `initRustCrypto()`. Learn how to properly initialize your Matrix client with end-to-end encryption support.

**You'll learn:**
- The difference between matrix-js-sdk and matrix-wasm-crypto-sdk
- How to create and configure a Matrix client
- The correct initialization order for crypto
- Common setup mistakes and how to avoid them

### [2. Receiving and Decrypting Messages](./02-receiving-and-decrypting-messages.md)
Learn how to listen for messages and work with decrypted content. Spoiler: decryption happens automatically!

**You'll learn:**
- How to listen to the Timeline event
- Accessing message content (already decrypted!)
- Handling different message types (text, images, files)
- What to do when decryption fails
- Loading message history

### [3. Sending Messages](./03-sending-messages.md)
Send text messages, images, files, and more. Encryption is automatic here too!

**You'll learn:**
- Sending simple text messages
- Uploading and sending files/images
- Formatting messages with HTML
- Sending replies
- Typing indicators
- Complete SvelteKit examples

### [4. Working with Rooms](./04-working-with-rooms.md)
Everything about rooms - listing, joining, creating, and managing them.

**You'll learn:**
- Getting and filtering room lists
- Joining and leaving rooms
- Creating rooms (including encrypted ones!)
- Managing room members
- Creating direct messages
- Handling invites

### [5. Authentication and Login](./05-authentication-and-login.md)
Get users logged in and handle sessions properly.

**You'll learn:**
- Password-based login
- User registration
- Storing auth credentials securely
- Auto-login on page load
- Protected routes in SvelteKit
- Proper logout

### [6. Common Pitfalls and Debugging](./06-common-pitfalls-and-debugging.md)
Save yourself hours of debugging! Learn from common mistakes.

**You'll learn:**
- The #1 mistake everyone makes (not waiting for sync)
- How to debug crypto issues
- Memory leak prevention
- Common error messages and their fixes
- Performance optimization tips

### [7. Actually Getting Messages to Work](./07-actually-getting-messages-to-work.md)
**READ THIS if getContent() returns encrypted garbage!** The real truth about how message decryption actually works (spoiler: the docs lie about it being automatic).

**You'll learn:**
- How to load existing messages (not just new ones!)
- Why getContent() returns encrypted content
- The difference between getContent() and getClearContent()
- How to properly wait for decryption to complete
- Complete working examples that ACTUALLY decrypt messages

## 🚀 Quick Start

If you're just getting started, here's the absolute minimum to get a working client:

```typescript
import sdk from 'matrix-js-sdk';

// 1. Create client
const client = sdk.createClient({
    baseUrl: "https://matrix.org",
    accessToken: "your_access_token",
    userId: "@xylight:matrix.org",
    deviceId: "DEVICEID",
});

// 2. Initialize crypto (this is the important part!)
await client.initRustCrypto();

// 3. Start the client
await client.startClient({ initialSyncLimit: 20 });

// 4. Wait for sync
client.once('sync', (state) => {
    if (state === 'PREPARED') {
        console.log("Ready! 🎉");
        const rooms = client.getRooms();
        console.log(`You're in ${rooms.length} rooms`);
    }
});
```

## 🎯 What Makes These Guides Special

- **Written specifically for SvelteKit** - All examples work with SvelteKit's reactivity
- **Clears up confusing docs** - I explain the conflicts between different documentation sources
- **Real, working code** - Every example is tested and practical
- **Encouraging tone** - You've got this! Building a Matrix client is challenging but totally doable
- **Covers the basics thoroughly** - Everything you need to get started, nothing you don't

## 🤔 Still Confused?

That's totally normal! Matrix has a lot of moving parts. Here are some tips:

1. **Start with tutorial #1** - Don't skip the crypto setup guide
2. **Type out the examples** - Don't just copy-paste, understand what each line does
3. **Use the debugging guide** - When stuck, #6 has solutions to common problems
4. **Enable verbose logging** - Makes debugging so much easier
5. **Take it one feature at a time** - Don't try to build everything at once

## 📖 Key Concepts to Understand

Before diving in, here are the core concepts:

- **Client**: Your connection to the Matrix network
- **Crypto/Rust Crypto**: The encryption layer (this is what confused you!)
- **Room**: A chat space (can be 1-on-1 or a group)
- **Event**: A message, state change, or any action in a room
- **Timeline**: The chronological list of events in a room
- **Sync**: The process of getting updates from the server

## 🛠️ Tech Stack Reference

You're using:
- **SvelteKit** - Your web framework
- **matrix-js-sdk** - The main Matrix SDK for JavaScript
- **matrix-wasm-crypto-sdk** - The WebAssembly crypto implementation

**Important:** You don't need to call `initAsync()` from matrix-wasm-crypto-sdk directly! The matrix-js-sdk calls it for you when you use `initRustCrypto()`. This is probably the #1 source of confusion.

## 💪 You're Going to Build Something Amazing!

Matrix development can be tricky at first, but you're on the right track by seeking out good resources. These guides will get you past the initial confusion and on to building great features.

Remember:
- Every Matrix client developer has been confused by the crypto initialization
- Every developer has forgotten to wait for sync at least once
- Every developer has dealt with confusing error messages

You're not alone, and you're doing great by learning this stuff!

Now get out there and build something awesome! 🚀

---

**Pro tip:** Bookmark this README and refer back to it when you need to remember which guide covers what topic.

You've got this, Xylight! 🌟
