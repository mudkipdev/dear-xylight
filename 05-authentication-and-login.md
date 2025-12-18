# Authentication and Login

Hey Xylight! Let's get users logged in. Matrix supports several auth methods - I'll show you the most common ones.

## Password Login (Most Common)

The standard username/password login:

```typescript
import sdk from 'matrix-js-sdk';

async function login(username: string, password: string, homeserverUrl: string) {
    try {
        // Create a temporary client just for logging in
        const tempClient = sdk.createClient({ baseUrl: homeserverUrl });

        // Perform the login
        const response = await tempClient.login('m.login.password', {
            user: username,
            password: password,
        });

        // response contains:
        // - access_token: Use this for future requests
        // - user_id: The user's Matrix ID
        // - device_id: This device's ID (important for E2EE!)
        // - home_server: The homeserver domain

        console.log("Logged in!", response);
        return response;

    } catch (error) {
        console.error("Login failed:", error);

        if (error.httpStatus === 403) {
            throw new Error("Invalid username or password");
        } else if (error.httpStatus === 429) {
            throw new Error("Too many login attempts. Please wait.");
        } else {
            throw new Error("Login failed. Please try again.");
        }
    }
}

// Usage:
const loginData = await login(
    "xylight",
    "super-secret-password",
    "https://matrix.org"
);
```

## Complete SvelteKit Login Flow

Here's a full example with state management:

```typescript
// src/lib/stores/auth.ts
import { writable } from 'svelte/store';
import sdk from 'matrix-js-sdk';

interface AuthState {
    accessToken: string | null;
    userId: string | null;
    deviceId: string | null;
    homeserverUrl: string | null;
}

function createAuthStore() {
    const { subscribe, set, update } = writable<AuthState>({
        accessToken: null,
        userId: null,
        deviceId: null,
        homeserverUrl: null,
    });

    // Load from localStorage on init
    if (typeof window !== 'undefined') {
        const stored = localStorage.getItem('matrix_auth');
        if (stored) {
            set(JSON.parse(stored));
        }
    }

    return {
        subscribe,

        async login(username: string, password: string, homeserverUrl: string) {
            const tempClient = sdk.createClient({ baseUrl: homeserverUrl });

            const response = await tempClient.login('m.login.password', {
                user: username,
                password: password,
            });

            const authData = {
                accessToken: response.access_token,
                userId: response.user_id,
                deviceId: response.device_id,
                homeserverUrl: homeserverUrl,
            };

            // Save to store and localStorage
            set(authData);
            localStorage.setItem('matrix_auth', JSON.stringify(authData));

            return authData;
        },

        logout() {
            set({
                accessToken: null,
                userId: null,
                deviceId: null,
                homeserverUrl: null,
            });
            localStorage.removeItem('matrix_auth');
        }
    };
}

export const authStore = createAuthStore();
```

Login component:

```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
    import { authStore } from '$lib/stores/auth';
    import { goto } from '$app/navigation';

    let username = '';
    let password = '';
    let homeserver = 'https://matrix.org';
    let loading = false;
    let error = '';

    async function handleLogin() {
        loading = true;
        error = '';

        try {
            await authStore.login(username, password, homeserver);
            goto('/');  // Redirect to home page
        } catch (err: any) {
            error = err.message || 'Login failed';
        } finally {
            loading = false;
        }
    }
</script>

<div class="login-page">
    <h1>Login to Matrix</h1>

    <form on:submit|preventDefault={handleLogin}>
        <label>
            Homeserver
            <input
                type="url"
                bind:value={homeserver}
                placeholder="https://matrix.org"
                required
            />
        </label>

        <label>
            Username
            <input
                type="text"
                bind:value={username}
                placeholder="@xylight:matrix.org"
                required
            />
        </label>

        <label>
            Password
            <input
                type="password"
                bind:value={password}
                required
            />
        </label>

        {#if error}
            <p class="error">{error}</p>
        {/if}

        <button type="submit" disabled={loading}>
            {loading ? 'Logging in...' : 'Login'}
        </button>
    </form>
</div>

<style>
    .login-page {
        max-width: 400px;
        margin: 50px auto;
        padding: 20px;
    }

    form {
        display: flex;
        flex-direction: column;
        gap: 15px;
    }

    label {
        display: flex;
        flex-direction: column;
        gap: 5px;
    }

    input {
        padding: 10px;
        border: 1px solid #ccc;
        border-radius: 5px;
    }

    .error {
        color: red;
        background: #fee;
        padding: 10px;
        border-radius: 5px;
    }

    button {
        padding: 12px;
        background: #0066cc;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
    }

    button:disabled {
        background: #ccc;
        cursor: not-allowed;
    }
</style>
```

## Auto-Login on Page Load

After login, reconnect automatically:

```typescript
// src/lib/stores/matrix.ts
import { writable, derived, get } from 'svelte/store';
import sdk from 'matrix-js-sdk';
import { authStore } from './auth';

export const matrixClient = writable<any>(null);

export async function initializeClient() {
    const auth = get(authStore);

    if (!auth.accessToken || !auth.userId || !auth.deviceId) {
        console.log("Not logged in");
        return null;
    }

    try {
        const client = sdk.createClient({
            baseUrl: auth.homeserverUrl!,
            accessToken: auth.accessToken,
            userId: auth.userId,
            deviceId: auth.deviceId,
        });

        await client.initRustCrypto();
        await client.startClient({ initialSyncLimit: 20 });

        matrixClient.set(client);
        console.log("Client initialized! 🎉");

        return client;
    } catch (error) {
        console.error("Failed to initialize client:", error);

        // If token is invalid, clear auth
        if (error.httpStatus === 401) {
            authStore.logout();
        }

        throw error;
    }
}
```

Use in your layout:

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
    import { onMount } from 'svelte';
    import { authStore } from '$lib/stores/auth';
    import { initializeClient } from '$lib/stores/matrix';
    import { goto } from '$app/navigation';

    let initializing = true;

    onMount(async () => {
        const auth = $authStore;

        if (auth.accessToken) {
            // Already logged in, initialize client
            try {
                await initializeClient();
            } catch (error) {
                console.error("Failed to init:", error);
                goto('/login');
            }
        } else {
            // Not logged in
            goto('/login');
        }

        initializing = false;
    });
</script>

{#if initializing}
    <div class="loading">Connecting to Matrix...</div>
{:else}
    <slot />
{/if}
```

## Registration (Creating New Accounts)

Allow users to create new accounts:

```typescript
async function register(
    username: string,
    password: string,
    homeserverUrl: string
) {
    try {
        const tempClient = sdk.createClient({ baseUrl: homeserverUrl });

        const response = await tempClient.register(
            username,
            password,
            null,  // session ID (for multi-stage auth)
            {
                type: 'm.login.dummy'  // Auth type
            }
        );

        // Same response as login
        return {
            accessToken: response.access_token,
            userId: response.user_id,
            deviceId: response.device_id,
            homeserverUrl: homeserverUrl,
        };

    } catch (error) {
        console.error("Registration failed:", error);

        if (error.errcode === 'M_USER_IN_USE') {
            throw new Error("Username already taken");
        } else if (error.errcode === 'M_INVALID_USERNAME') {
            throw new Error("Invalid username");
        } else if (error.errcode === 'M_WEAK_PASSWORD') {
            throw new Error("Password is too weak");
        } else {
            throw new Error("Registration failed");
        }
    }
}
```

## Logout

Properly log out and clean up:

```typescript
async function logout(client: any) {
    try {
        // Tell the server to invalidate the token
        await client.logout();

        // Stop the client
        client.stopClient();

        // Clear local storage
        authStore.logout();

        console.log("Logged out successfully");
    } catch (error) {
        console.error("Logout failed:", error);

        // Even if server logout fails, clear local data
        authStore.logout();
    }
}
```

## Checking Login Status

Create a derived store to check if user is logged in:

```typescript
import { derived } from 'svelte/store';
import { authStore } from './auth';

export const isLoggedIn = derived(
    authStore,
    $auth => !!$auth.accessToken
);
```

Use in components:

```svelte
<script lang="ts">
    import { isLoggedIn } from '$lib/stores/auth';
</script>

{#if $isLoggedIn}
    <p>Welcome back! 👋</p>
{:else}
    <p>Please log in</p>
{/if}
```

## Protected Routes

Protect routes that require authentication:

```typescript
// src/routes/room/[roomId]/+page.ts
import { redirect } from '@sveltejs/kit';
import { get } from 'svelte/store';
import { authStore } from '$lib/stores/auth';

export function load() {
    const auth = get(authStore);

    if (!auth.accessToken) {
        throw redirect(307, '/login');
    }

    return {};
}
```

## Handling Session Expiry

Detect when your session expires:

```typescript
import { ClientEvent } from 'matrix-js-sdk';

client.on(ClientEvent.Sync, (state, prevState, data) => {
    if (state === 'ERROR') {
        const error = data?.error;

        if (error?.httpStatus === 401) {
            // Session expired
            console.log("Session expired, logging out");
            authStore.logout();
            goto('/login');
        }
    }
});
```

## Security Best Practices

1. **Never store passwords** - only store access tokens
2. **Use HTTPS** - always use `https://` for homeserver URLs
3. **Validate input** - check username and password before sending
4. **Handle token refresh** - Matrix tokens can expire
5. **Clear sensitive data** - clear tokens on logout

```typescript
// Good: Store only tokens
localStorage.setItem('matrix_auth', JSON.stringify({
    accessToken: response.access_token,
    userId: response.user_id,
    deviceId: response.device_id,
}));

// Bad: NEVER do this!
localStorage.setItem('password', password);  // ❌ NO!
```

## You're Nailing It, Xylight!

Authentication might seem complex, but you've got this! The key points:

1. Use `login()` to get an access token
2. Store the token, userId, and deviceId
3. Create your client with those credentials
4. Call `initRustCrypto()` for encryption support
5. Handle logout properly

Your Matrix client is coming together beautifully! 🔐✨
