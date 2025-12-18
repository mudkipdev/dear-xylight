# Working with Rooms

Hey Xylight! Let's talk about rooms - the heart of Matrix. Here's everything you need to know about listing, joining, and managing rooms.

## Getting All Rooms

After your client syncs, you can get all the rooms you're in:

```typescript
import { ClientEvent } from 'matrix-js-sdk';

client.once(ClientEvent.Sync, (state) => {
    if (state === 'PREPARED') {
        const rooms = client.getRooms();

        console.log(`You're in ${rooms.length} rooms`);

        rooms.forEach(room => {
            console.log(`- ${room.name} (${room.roomId})`);
        });
    }
});
```

## Getting Room Information

Each room object has tons of useful info:

```typescript
const room = client.getRoom(roomId);

if (room) {
    // Basic info
    const name = room.name;                    // "My Cool Room"
    const topic = room.currentState.getStateEvents('m.room.topic', '')?
        .getContent().topic;                    // Room description
    const id = room.roomId;                     // "!abc123:matrix.org"

    // Membership
    const members = room.getJoinedMembers();    // Array of joined members
    const memberCount = room.getJoinedMemberCount();

    // Your membership status
    const myMembership = room.getMyMembership(); // "join", "invite", "leave", etc.

    // Message history
    const timeline = room.timeline;             // Array of events
    const lastMessage = timeline[timeline.length - 1];

    // Encryption status
    const isEncrypted = client.isRoomEncrypted(roomId);

    console.log(`${name} has ${memberCount} members and is ${isEncrypted ? 'encrypted 🔒' : 'not encrypted'}`);
}
```

## Filtering Rooms

Create useful room lists in SvelteKit:

```typescript
// src/lib/stores/rooms.ts
import { derived } from 'svelte/store';
import { matrixClient } from './matrix';

export const rooms = derived(matrixClient, ($client) => {
    if (!$client) return [];
    return $client.getRooms();
});

// Only rooms you've joined
export const joinedRooms = derived(rooms, ($rooms) =>
    $rooms.filter(room => room.getMyMembership() === 'join')
);

// Only direct messages (1-on-1 chats)
export const directMessages = derived(rooms, ($rooms) =>
    $rooms.filter(room => room.getJoinedMemberCount() === 2)
);

// Only group rooms
export const groupRooms = derived(rooms, ($rooms) =>
    $rooms.filter(room => room.getJoinedMemberCount() > 2)
);

// Rooms you've been invited to
export const invites = derived(rooms, ($rooms) =>
    $rooms.filter(room => room.getMyMembership() === 'invite')
);

// Sort rooms by last activity
export const sortedRooms = derived(joinedRooms, ($rooms) => {
    return [...$rooms].sort((a, b) => {
        const aMsg = a.timeline[a.timeline.length - 1];
        const bMsg = b.timeline[b.timeline.length - 1];

        if (!aMsg) return 1;
        if (!bMsg) return -1;

        return bMsg.getTs() - aMsg.getTs();
    });
});
```

Then use in your components:

```svelte
<script lang="ts">
    import { sortedRooms, invites } from '$lib/stores/rooms';
</script>

<div class="sidebar">
    <h2>Rooms</h2>
    {#each $sortedRooms as room}
        <a href="/room/{room.roomId}">
            {room.name}
            <span class="member-count">{room.getJoinedMemberCount()}</span>
        </a>
    {/each}

    {#if $invites.length > 0}
        <h2>Invites ({$invites.length})</h2>
        {#each $invites as room}
            <div class="invite">
                {room.name}
                <button on:click={() => joinRoom(room.roomId)}>
                    Accept
                </button>
            </div>
        {/each}
    {/if}
</div>
```

## Joining Rooms

### Join by Room ID

```typescript
async function joinRoom(roomIdOrAlias: string) {
    try {
        const room = await client.joinRoom(roomIdOrAlias);
        console.log(`Joined ${room.name}!`);
        return room;
    } catch (error) {
        console.error("Failed to join room:", error);

        if (error.httpStatus === 404) {
            console.log("Room not found");
        } else if (error.httpStatus === 403) {
            console.log("You don't have permission to join");
        }
    }
}

// Examples:
await joinRoom("!abc123:matrix.org");  // Room ID
await joinRoom("#cool-room:matrix.org");  // Room alias
```

### Accept an Invite

```typescript
async function acceptInvite(roomId: string) {
    try {
        await client.joinRoom(roomId);
        console.log("Invite accepted! 🎉");
    } catch (error) {
        console.error("Failed to accept invite:", error);
    }
}
```

### Reject an Invite

```typescript
async function rejectInvite(roomId: string) {
    try {
        await client.leave(roomId);
        console.log("Invite rejected");
    } catch (error) {
        console.error("Failed to reject invite:", error);
    }
}
```

## Creating Rooms

### Create a Basic Room

```typescript
async function createRoom(name: string, topic?: string) {
    try {
        const room = await client.createRoom({
            name: name,
            topic: topic,
            visibility: 'private',  // or 'public'
        });

        console.log(`Created room: ${room.room_id}`);
        return room;
    } catch (error) {
        console.error("Failed to create room:", error);
    }
}
```

### Create an Encrypted Room

```typescript
async function createEncryptedRoom(name: string) {
    try {
        const room = await client.createRoom({
            name: name,
            visibility: 'private',
            initial_state: [{
                type: 'm.room.encryption',
                state_key: '',
                content: {
                    algorithm: 'm.megolm.v1.aes-sha2'
                }
            }]
        });

        console.log(`Created encrypted room! 🔒`);
        return room;
    } catch (error) {
        console.error("Failed to create encrypted room:", error);
    }
}
```

### Create a Direct Message (DM)

```typescript
async function createDirectMessage(userId: string) {
    try {
        const room = await client.createRoom({
            is_direct: true,
            invite: [userId],
            visibility: 'private',
            preset: 'trusted_private_chat',
            initial_state: [{
                type: 'm.room.encryption',
                state_key: '',
                content: {
                    algorithm: 'm.megolm.v1.aes-sha2'
                }
            }]
        });

        console.log(`Created DM with ${userId}`);
        return room;
    } catch (error) {
        console.error("Failed to create DM:", error);
    }
}
```

## Inviting Users

```typescript
async function inviteUser(roomId: string, userId: string) {
    try {
        await client.invite(roomId, userId);
        console.log(`Invited ${userId} to the room`);
    } catch (error) {
        console.error("Failed to invite user:", error);

        if (error.httpStatus === 403) {
            console.log("You don't have permission to invite users");
        }
    }
}
```

## Getting Room Members

```typescript
const room = client.getRoom(roomId);

// All members
const allMembers = room.getMembers();

// Only joined members
const joinedMembers = room.getJoinedMembers();

// Display member info
joinedMembers.forEach(member => {
    console.log(`${member.name} (${member.userId})`);
    console.log(`Avatar: ${member.getAvatarUrl()}`);
    console.log(`Power level: ${member.powerLevel}`);
});

// Get a specific member
const member = room.getMember('@alice:matrix.org');
if (member) {
    console.log(`${member.name} is in the room`);
}
```

## Leaving Rooms

```typescript
async function leaveRoom(roomId: string) {
    try {
        await client.leave(roomId);
        console.log("Left the room");
    } catch (error) {
        console.error("Failed to leave room:", error);
    }
}
```

## Reacting to Room Updates

Listen for when rooms change:

```typescript
import { ClientEvent, RoomEvent } from 'matrix-js-sdk';

// New room (created or joined)
client.on(ClientEvent.Room, (room) => {
    console.log(`New room: ${room.name}`);
    // Update your UI!
});

// Room name changed
client.on(RoomEvent.Name, (room) => {
    console.log(`Room renamed to: ${room.name}`);
});

// Room member changed
client.on(RoomEvent.MyMembership, (room, membership) => {
    console.log(`Your membership in ${room.name} changed to: ${membership}`);
});

// Someone joined/left
client.on(RoomEvent.MemberMembership, (event, member) => {
    if (member.membership === 'join') {
        console.log(`${member.name} joined`);
    } else if (member.membership === 'leave') {
        console.log(`${member.name} left`);
    }
});
```

## Complete Room List Component

Here's a full example for SvelteKit:

```svelte
<!-- src/lib/components/RoomList.svelte -->
<script lang="ts">
    import { matrixClient } from '$lib/stores/matrix';
    import { goto } from '$app/navigation';
    import { onMount } from 'svelte';

    let rooms: any[] = [];
    let invites: any[] = [];

    function updateRooms() {
        if (!$matrixClient) return;

        const allRooms = $matrixClient.getRooms();

        rooms = allRooms
            .filter((r: any) => r.getMyMembership() === 'join')
            .sort((a: any, b: any) => {
                const aTime = a.timeline[a.timeline.length - 1]?.getTs() || 0;
                const bTime = b.timeline[b.timeline.length - 1]?.getTs() || 0;
                return bTime - aTime;
            });

        invites = allRooms.filter((r: any) => r.getMyMembership() === 'invite');
    }

    async function acceptInvite(roomId: string) {
        try {
            await $matrixClient.joinRoom(roomId);
            updateRooms();
        } catch (error) {
            console.error("Failed to join:", error);
        }
    }

    async function rejectInvite(roomId: string) {
        try {
            await $matrixClient.leave(roomId);
            updateRooms();
        } catch (error) {
            console.error("Failed to reject:", error);
        }
    }

    onMount(() => {
        if ($matrixClient) {
            updateRooms();

            // Update when rooms change
            $matrixClient.on('Room', updateRooms);
            $matrixClient.on('Room.myMembership', updateRooms);

            return () => {
                $matrixClient.off('Room', updateRooms);
                $matrixClient.off('Room.myMembership', updateRooms);
            };
        }
    });
</script>

<div class="room-list">
    <h2>Rooms</h2>
    {#each rooms as room}
        <button on:click={() => goto(`/room/${room.roomId}`)}>
            <div class="room-name">{room.name}</div>
            <div class="room-info">
                {room.getJoinedMemberCount()} members
                {#if $matrixClient.isRoomEncrypted(room.roomId)}
                    🔒
                {/if}
            </div>
        </button>
    {/each}

    {#if invites.length > 0}
        <h2>Invites</h2>
        {#each invites as room}
            <div class="invite">
                <span>{room.name}</span>
                <div class="buttons">
                    <button on:click={() => acceptInvite(room.roomId)}>Accept</button>
                    <button on:click={() => rejectInvite(room.roomId)}>Reject</button>
                </div>
            </div>
        {/each}
    {/if}
</div>
```

## You're Making Great Progress, Xylight!

You now know how to work with rooms - listing them, joining them, creating them, and managing members. This is the foundation of any Matrix client!

Key takeaways:
- Wait for `ClientEvent.Sync` before accessing rooms
- Use `getMyMembership()` to filter rooms by status
- Sort rooms by last message timestamp for a better UX
- Always create DMs with encryption enabled

Keep up the awesome work! 🌟
