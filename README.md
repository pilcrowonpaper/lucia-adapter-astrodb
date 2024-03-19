# lucia-adapter-astrodb

Astro DB adapter for Lucia.

```
npm i lucia-adapter-astrodb
```

## Setup

```ts
// db/config.ts
import { column, defineDb, defineTable } from "astro:db";

const User = defineTable({
	columns: {
		id: column.text({
			primaryKey: true
		})
	}
});

const Session = defineTable({
	columns: {
		id: column.text({
			primaryKey: true
		}),
		expiresAt: column.date(),
		userId: column.text({
			references: () => User.columns.id
		})
	}
});

export default defineDb({
	tables: {
		User,
		Session
	}
});
```

```ts
import { AstroDBAdapter } from "lucia-adapter-astrodb";
import { db, Session, User } from "astro:db";

const adapter = new AstroDBAdapter(db, Session, User);
```

## How to setup Astro DB with Astro the lucia-adapter-astrodb

### Create User and Session Table
In the db/config.ts file you have to add a Table to store the Session and a Table to Store the User:
```ts
const User = defineTable({
  columns: {
    id: column.text({ primaryKey: true }),
    username: column.text({ unique: true }),
    hashed_password: column.text(),
  }
});

const Session = defineTable({
	columns: {
		id: column.text({
			primaryKey: true
		}),
		expiresAt: column.date(),
		userId: column.text({
			references: () => User.columns.id
		})
	}
});
```

### Create the Lucia Auth Setup with the Adapter
Create a @/lib/auth.ts file to create all the Lucia Auth configurations.

```ts
import { Lucia } from "lucia";
import { AstroDBAdapter } from "lucia-adapter-astrodb";
import { db, User, Session } from "astro:db";

const adapter = new AstroDBAdapter(db, Session, User);


export const lucia = new Lucia(adapter, {
	sessionCookie: {
		attributes: {
			secure: import.meta.env.PROD
		}
	},
	getUserAttributes: (attributes) => {
		return {
			// attributes has the type of DatabaseUserAttributes
			username: attributes.username
		};
	}
});

declare module "lucia" {
	interface Register {
		Lucia: typeof lucia;
		DatabaseUserAttributes: DatabaseUserAttributes;
	}
}

interface DatabaseUserAttributes {
	username: string;
}
```

### Create the different Routes for Login and SignUp


```ts
// pages/api/login.ts
import { lucia } from "@/lib/auth";
import { Argon2id } from "oslo/password";

import type { APIContext } from "astro";
import { User, db, eq } from "astro:db";

export async function POST(context: APIContext): Promise<Response> {
	const formData = await context.request.formData();
	const username = formData.get("username");
	if (
		typeof username !== "string" ||
		username.length < 3 ||
		username.length > 31 ||
		!/^[a-z0-9_-]+$/.test(username)
	) {
		return new Response("Invalid username", {
			status: 400
		});
	}
	const password = formData.get("password");
	if (typeof password !== "string" || password.length < 6 || password.length > 255) {
		return new Response("Invalid password", {
			status: 400
		});
	}

    const existingUser = await db.select().from(User).where(eq(User.username, username.toLowerCase()));


	if (!existingUser) {
		// NOTE:
		// Returning immediately allows malicious actors to figure out valid usernames from response times,
		// allowing them to only focus on guessing passwords in brute-force attacks.
		// As a preventive measure, you may want to hash passwords even for invalid usernames.
		// However, valid usernames can be already be revealed with the signup page among other methods.
		// It will also be much more resource intensive.
		// Since protecting against this is none-trivial,
		// it is crucial your implementation is protected against brute-force attacks with login throttling etc.
		// If usernames are public, you may outright tell the user that the username is invalid.
		return new Response("Incorrect username or password", {
			status: 400
		});
	}

	const validPassword = await new Argon2id().verify(existingUser[0].hashed_password, password);
	if (!validPassword) {
		return new Response("Incorrect username or password", {
			status: 400
		});
	}

	const session = await lucia.createSession(existingUser[0].id, {});
	const sessionCookie = lucia.createSessionCookie(session.id);
	context.cookies.set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);

	return context.redirect("/");
}
```


#### Signup Route

```ts
// pages/api/signup.ts
import { lucia } from "@/lib/auth";
import { generateId } from "lucia";
import { Argon2id } from "oslo/password";

import type { APIContext } from "astro";
import { User, db } from "astro:db";

export async function POST(context: APIContext): Promise<Response> {
	const formData = await context.request.formData();
	const username = formData.get("username");
	// username must be between 4 ~ 31 characters, and only consists of lowercase letters, 0-9, -, and _
	// keep in mind some database (e.g. mysql) are case insensitive
	if (
		typeof username !== "string" ||
		username.length < 3 ||
		username.length > 31 ||
		!/^[a-z0-9_-]+$/.test(username)
	) {
		return new Response("Invalid username", {
			status: 400
		});
	}
	const password = formData.get("password");
	if (typeof password !== "string" || password.length < 6 || password.length > 255) {
		return new Response("Invalid password", {
			status: 400
		});
	}

	const userId = generateId(15);
	const hashedPassword = await new Argon2id().hash(password);

	// TODO: check if username is already used


	await db.insert(User).values({
		id: userId,
		username: username,
		hashed_password: hashedPassword
	});

	const session = await lucia.createSession(userId, {});
	const sessionCookie = lucia.createSessionCookie(session.id);
	context.cookies.set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);

	return context.redirect("/");
}
```

### Create a Middleware to Update the Cookie
Create a middleware.ts file to keep the Cookie and Session up to date, put it in the /src Folder to call it every time you call a route.


```ts
import { lucia } from "./lib/auth";
import { verifyRequestOrigin } from "lucia";
import { defineMiddleware } from "astro:middleware";

export const onRequest = defineMiddleware(async (context, next) => {
	if (context.request.method !== "GET") {
		const originHeader = context.request.headers.get("Origin");
		const hostHeader = context.request.headers.get("Host");
		if (!originHeader || !hostHeader || !verifyRequestOrigin(originHeader, [hostHeader])) {
			return new Response(null, {
				status: 403
			});
		}
	}

	const sessionId = context.cookies.get(lucia.sessionCookieName)?.value ?? null;
	if (!sessionId) {
		context.locals.user = null;
		context.locals.session = null;
		return next();
	}

	const { session, user } = await lucia.validateSession(sessionId);
	if (session && session.fresh) {
		const sessionCookie = lucia.createSessionCookie(session.id);
		context.cookies.set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
	}
	if (!session) {
		const sessionCookie = lucia.createBlankSessionCookie();
		context.cookies.set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
	}
	context.locals.session = session;
	context.locals.user = user;
	return next();
});
```
