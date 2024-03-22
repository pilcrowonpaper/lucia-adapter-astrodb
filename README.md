# lucia-adapter-astrodb

Astro DB adapter for Lucia.

```
npm i lucia-adapter-astrodb
```

## Setup

You need to manually mark `astro:db` as an external dependency.

```ts
export default defineConfig({
	// ...
	vite: {
		optimizeDeps: {
			exclude: ["astro:db"]
		}
	}
});
```

Define the schema and initialize the adapter.

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
