## Installation

```bash
npm install lucia-sveltekit
```

## Set up Lucia

In `$lib/lucia.js`, import `lucia` and export it (in this case as `auth`). During this step, lucia requires 3 things: an adapter, a secret key, and the current environment. An adapter connects lucia to your database, secret is used to encrypt and hash your data, and env tells Lucia if it's running in development or production environment. Different adapters are needed for different databases, and it can be easily created if Lucia doesn't provide one.

```js
import lucia from "lucia-sveltekit";
import supabase from "@lucia-sveltekit/adapter-supabase";
import { dev } from "$app/environment";

export const auth = lucia({
    adapter: supabase(),
    secret: "aWmJoT0gOdjh2-Zc2Zv3BTErb29qQNWEunlj",
    env: dev ? "DEV" : "PROD",
});
```

For Lucia to work, its own handle functions must be added to hooks (`/src/hooks.server.js`).

```ts
import { auth } from "$lib/lucia.js";
export const handle = auth.handleHooks();
```

This is so Lucia can listen for requests to endpoints that Lucia exposes (creates) for token refresh and sign outs. Make sure to not have existing endpoints that overlaps with them:

-   /api/auth/refresh
-   /api/auth/logout

Finally, create `/+layout.svelte` and `/+layout.server.js`.

In `/+layout.server.js`, create a load function. This will check if the access token has expired and refresh it if so. This also exposes the user data to the client.

```ts
import { handleSession } from "lucia-sveltekit";

export const load = auth.handleServerLoad(handleSession());
```

`handleServerLoad()` can take multiple load functions and run it in sequence.

```ts
export const load = auth.handleServerLoad(handleSession(), async () => {
    return {
        message: "hello",
    };
});
```

In `/+layout.svelte`, import `handleSilentRefresh`. This will refresh the access token when it nears expiration.

```ts
import { handleSilentRefresh } from "lucia-sveltekit/client";

handleSilentRefresh();
```

## Create a user

### The basic steps

1. Send the user's input to an endpoint
2. Verify the user's input (check if password is secure, verify the user's email, etc)
3. Create a new user using Lucia
4. Save the cookies returned by Lucia
5. In the client, redirect the user

> When redirecting a user after auth state change (signin, signout), use `window.location.href` in the client or http response redirect in the server instead of SvelteKit's `goto()` as `goto()` will not update the cookies.

### Implementation

`auth.createUser` creates a new user and returns a few tokens and cookies.

The first parameter is the auth id, and the second parameter is the identifier. The third parameter is optional, and you can provide a password and user data to be saved alongside other data. In the example below, `email` will be saved as its own column in the `user` table.

After creating a user, Lucia will return a set of tokens and cookies. These cookies should be saved to the user using the `set-cookie` headers.

```js
export const POST = async () => {
    // ...
    try {
        const userSession = await auth.createUser("email", email, {
            password,
            user_data: {
                email,
            },
        });
        return {
            headers: {
                "set-cookie": userSession.cookies, // set cookies
            },
        };
    } catch {
        // handle errors
    }
};
```

```ts
// for POST actions
export const POST = async ({ cookies }) => {
    // ...
    try {
        // same as above
        setCookie(cookies, userSession.cookies)
        return;
    } catch {
        // handle errors
    }
```

## Authenticate users

### The basic steps

1. Authenticate the user using Lucia
2. Save the cookies returned by Lucia

### Implementation

`auth.authenticateUser` authenticates a user using an identifier and (if necessary) a password.

The first parameter is the auth id and the second parameter is the identifier. The third parameter is the password (if used for that auth id).

```js
import { setCookie } from "lucia-sveltekit"

export const actions = {
    default: async ({ cookies }) => {
        // ...
        try {
            const userSession = await auth.authenticateUser(
                "email",
                email,
                password
            );
            setCookie(cookies, ...userSession.cookies)
            // redirect if needed
        } catch {
            // invalid input
        }
    };
}
```

For standalone endpoints, cookies can be set using `setHeaders()`.

```ts
export const POST = async ({ setHeaders }) => {
    const userSession = await auth.authenticateUser("email", email, password);
    setHeaders({
        "set-cookie", userSession.cookies.join()
    });
};
```

## Check if the user is authenticated

[`Session`](/references/types#session) is an object when a user if authenticated and `null` if not.

### In the client

```ts
import { getSession } from "lucia-sveltekit/client";

const session = getSession();

if ($session) {
    // authenticated
}
```

### In a load function

[`handleLoad()`](/load#handleLoad) takes a normal load function (but with added params like `getSession()`).

```ts
import { handleLoad } from "lucia-sveltekit";

export const load = handleLoad(async ({ getSession }) => {
    const session = await getSession();
    if (session) {
        // authenticated
    }
});
```

### In a server load function

Similar to `handleLoad()` but for server context.

```ts
export const load = auth.handleServerLoad(async ({ getSession }) => {
    const session = await getSession();
    if (session) {
        // authenticated
    }
});
```

### In an endpoint

Lucia provides 2 functions that verifies if a request is valid. **These should not be used interchangeably.**

#### GET and POST requests

The access token should be send as a bearer token in the authorization header.

```ts
// endpoint
export const GET = async ({ request }) => {
    try {
        const user = await auth.validateRequest(request);
        // authenticated
    } catch {
        // not authenticated
    }
};
```

```ts
// send request
await fetch("/some-endpoint", {
    headers: {
        Authorization: `Bearer ${access_token}`,
    },
});
```

#### GET requests

Can be used for page endpoints. **Do NOT use this for POST requests as it is vulnerable to CSRF attacks**, and it will throw an error if it is not a GET request.

```js
// endpoint
export const GET = async ({ request }) => {
    try {
        const user = await auth.validateRequestByCookie(request);
        // authenticated
    } catch {
        // not authenticated
    }
};
```

```js
// send request
await fetch("/some-endpoint");
```

## Signing out users

```js
import { signOut } from "lucia-sveltekit/client";

const signOutUser = async () => {
    try {
        await signOut();
        window.location.href = "/";
    } catch {
        // handle error
    }
};
```

## Types

`User` and `Session` returned by Lucia's APIs can be typed using the `Lucia` namespace. In `src/app.d.ts`, copy the following:

```ts
declare namespace Lucia {
    interface UserData {}
}
```

## Deploying the app

Apps using Lucia cannot be deployed to edge functions (CloudFlare Workers, Vercel Edge Functions, Netlify Edge Functions) because it has a dependency on Node's crypto module and other Node native APIs.
