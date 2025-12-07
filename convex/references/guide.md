# Convex for AI Agents

*A practical guide so you stop screwing up Convex apps.*

You (the AI agent) will often be asked to "build a Convex backend", "wire this UI to Convex", or "add feature X to this existing Convex project". This doc is your cheat‑sheet.

Your priorities:

1. **Compile + typecheck** on first try (or very close).
2. **Respect Convex's constraints** (indexes, limits, auth, schedulers, file storage).
3. **Avoid expensive / subtle bugs** (wrong indexes, misuse of actions, broken hooks, etc.).

---

## 0. Mental model

* Convex is a **hosted backend** with:

  * A **database** (documents + indexes + full‑text search).
  * **Functions**: `query`, `mutation`, `action`, plus internal variants.
  * **Realtime queries**: `useQuery` on the client auto‑subscribes.
  * **File storage**.
  * **Auth** (often via `@convex-dev/auth`).
  * **Scheduler** (cron + `runAfter`).
* Code layout:

  * **Backend**: `convex/*.ts` files.
  * **Client**: React using `convex/react` (`useQuery`, `useMutation`).
  * **Generated API**: `convex/_generated/api` exports `api` + `internal`.

Keep that mental model in your "working set" whenever you write Convex code.

---

## 1. Schema & Validators

### 1.1. Where schema lives

* Always define schema in **`convex/schema.ts`**:

```ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  messages: defineTable({
    body: v.string(),
    author: v.string(),
  }).index("by_author", ["author"]),
});
```

* System fields:

  * `_id: v.id(tableName)`
  * `_creationTime: v.number()`
  * You **never** declare these in the schema; Convex adds them.

### 1.2. Validators to use

Common validators:

* `v.string()`
* `v.number()`
* `v.boolean()`
* `v.id("tableName")`
* `v.array(v.string())`
* `v.object({ ... })`
* `v.record(keyValidator, valueValidator)`
* `v.null()`
* `v.int64()` (for `bigint`)

**Do not use** `v.map()` or `v.set()` – they're not supported.

Example:

```ts
import { v } from "convex/values";

const messageValidator = v.object({
  body: v.string(),
  author: v.string(),
  pinned: v.optional(v.boolean()),
});
```

### 1.3. Return validators: what to do

There are two styles in the Chef prompts:

* Cursor rules often show **both** `args` and `returns`.
* Newer guidelines say: "ALWAYS use argument validators, and **skip return validators at first**."

For your own agent, a safe policy:

* **Always** define `args`.
* For `returns`:

  * You **may omit** `returns` initially for simple endpoints.
  * If you *do* add it, make sure it's correct and stays synced with the actual return value.
  * If a function returns `null`, `returns: v.null()` is fine.

Example with returns:

```ts
export const getUser = query({
  args: { userId: v.id("users") },
  returns: v.union(v.null(), v.object({ name: v.string() })),
  handler: async (ctx, args) => {
    return await ctx.db.get(args.userId);
  },
});
```

---

## 2. Functions: public vs internal, and where to use what

Convex functions are registered via helpers from `./_generated/server`:

* **Public API** (callable from client / HTTP):

  * `query`
  * `mutation`
  * `action`
* **Internal-only** (callable only from other Convex functions):

  * `internalQuery`
  * `internalMutation`
  * `internalAction`

Example:

```ts
import {
  query,
  mutation,
  internalQuery,
  internalMutation,
  internalAction,
} from "./_generated/server";
import { v } from "convex/values";
```

### 2.1. When to use which

* **query**

  * Read-only (no writes).
  * Fast (< 1 second).
  * Realtime subscriptions (via `useQuery`).
* **mutation**

  * Database writes / transactions.
  * Short-lived (same 1 second constraint).
* **action**

  * Long-running and/or network‑heavy tasks.
  * Runs in Node runtime.
  * Can use arbitrary Node modules.
  * **No direct `ctx.db` access** (must use `ctx.runQuery` / `ctx.runMutation`).
* **Internal variants**: same as above, but **not exposed** to clients.

### 2.2. Function registration rules

* You **must** register functions like:

```ts
export const listMessages = query({
  args: { channelId: v.id("channels") },
  handler: async (ctx, args) => {
    // ...
  },
});
```

* You **cannot** dynamically register functions via `api` or `internal`.
* Every function needs **argument validators**.
* If a JS function doesn't return anything, it implicitly returns `null`.

---

## 3. Calling other functions (ctx.runQuery / ctx.runMutation / ctx.runAction)

From inside Convex functions:

* `ctx.runQuery(api.someFile.someQuery, args)`
* `ctx.runMutation(api.someFile.someMutation, args)`
* `ctx.runAction(api.someFile.someAction, args)`

Rules:

* You **must** pass a **function reference** (like `api.example.f`), not the function itself.
* From **actions**, use `ctx.runQuery` / `ctx.runMutation` to talk to the DB – **never** `ctx.db`.
* When calling another function from the **same file**, add a **type annotation** to avoid TypeScript circularity issues.

Example:

```ts
import { api } from "./_generated/api";

export const f = query({
  args: { name: v.string() },
  handler: async (ctx, args) => {
    return "Hello " + args.name;
  },
});

export const g = query({
  args: {},
  handler: async (ctx, args) => {
    const result: string = await ctx.runQuery(api.example.f, { name: "Bob" });
    return null;
  },
});
```

---

## 4. Function references via `api` and `internal`

The generated file `./_generated/api` exports:

```ts
import { api, internal } from "./_generated/api";
```

* For a public function `export const f = query(...)` in `convex/example.ts`:

  * Reference: `api.example.f`.
* For an internal function `export const g = internalMutation(...)` in `convex/example.ts`:

  * Reference: `internal.example.g`.
* Nested directories mirror the path: `convex/messages/access.ts` → `api.messages.access.h`.

Use:

* `api.*` when called from client / other functions for **public** functions.
* `internal.*` for **internal** functions from other Convex functions.

---

## 5. Indexes, queries, and search

This is where most Convex mistakes happen. Be extra careful.

### 5.1. Index basics

* Define indexes on tables with `.index(name, [fields...])`.
* Rules:

  * Index names **must be unique** per table.
  * Index names should include the field names: e.g. `by_channel_and_author` for `["channelId", "authorId"]`.
  * Convex automatically adds `_creationTime` as the **final index column** and provides built‑in index **`by_creation_time`** for all tables.
  * **Never** define your own `.index("by_creation_time", ["_creationTime"])` – that's always wrong.
  * **Never** include `_creationTime` as a column in a custom index.

Bad:

```ts
.index("by_creation_time", ["_creationTime"]) // ❌
.index("by_author_and_creation_time", ["author", "_creationTime"]) // ❌
```

Good:

```ts
.index("by_author", ["author"]) // Convex implicitly adds _creationTime
```

### 5.2. Querying with indexes

You should **not** use `.filter()` in queries. Use `.withIndex()` instead.

Bad:

```ts
// ❌ Do not do this
await ctx.db
  .query("messages")
  .filter(q => q.eq(q.field("author"), args.author))
  .collect();
```

Good:

```ts
const messages = await ctx.db
  .query("messages")
  .withIndex("by_author", q => q.eq("author", args.author))
  .order("desc")
  .collect();
```

### 5.3. Ordering

* By default, queries return docs in ascending `_creationTime`.
* You can explicitly specify:

  * `.order("asc")`
  * `.order("desc")`
* For indexed queries, ordering is based on index columns, with `_creationTime` as last column.

### 5.4. Pagination

Use `paginationOptsValidator` from `convex/server`:

```ts
import { paginationOptsValidator } from "convex/server";

export const listWithExtraArg = query({
  args: {
    paginationOpts: paginationOptsValidator,
    author: v.string(),
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_author", q => q.eq("author", args.author))
      .order("desc")
      .paginate(args.paginationOpts);
  },
});
```

`paginationOpts` fields:

* `numItems: number`
* `cursor: string | null`

Return value of `.paginate()`:

* `page`: array of documents
* `isDone`: boolean
* `continueCursor`: string

### 5.5. Full‑text search

* Define search index in schema:

```ts
messages: defineTable({
  body: v.string(),
  channel: v.string(),
}).searchIndex("search_body", {
  searchField: "body",
  filterFields: ["channel"],
});
```

* Query:

```ts
const messages = await ctx.db
  .query("messages")
  .withSearchIndex("search_body", q =>
    q.search("body", "hello hi").eq("channel", "#general"),
  )
  .take(10);
```

---

## 6. Query & mutation guidelines

### 6.1. Queries

* **Do not** use `.delete()` on queries (it doesn't exist).

  * Instead: `.collect()` results or iterate them, then `ctx.db.delete(doc._id)` from a mutation.
* For a single doc from a query:

  * Use `.unique()` when you expect exactly one match (throws if multiple).
* For large result sets:

  * Prefer async iteration:

```ts
for await (const row of ctx.db.query("messages")) {
  // ...
}
```

* Avoid heavy operations in queries; they must complete within **1 second**.

### 6.2. Mutations

* Use `ctx.db.insert(table, doc)` to insert.
* Use `ctx.db.patch(id, partial)` for **partial updates**.
* Use `ctx.db.replace(id, fullDoc)` for **full replacement** (throws if not exists).

---

## 7. Actions & HTTP endpoints

### 7.1. Actions

* Files containing actions that use Node modules should start with:

```ts
"use node";
```

Rules:

* **Only** actions in `"use node"` files (no queries/mutations).
* **Never** call `ctx.db` inside actions.

  * Use `ctx.runQuery`, `ctx.runMutation`, `ctx.runAction`.
* **No dynamic imports in queries/mutations** - only actions support `await import()`.

  * Bad: `const { foo } = await import('./module');` inside a query/mutation
  * Good: `import { foo } from './module';` at top of file
  * Actions can use dynamic imports because they run in Node.js environment
* Use actions for:

  * Calling external APIs (OpenAI, Resend, etc.).
  * Long‑running tasks (up to **10 minutes**).

Basic action:

```ts
import { action } from "./_generated/server";

export const exampleAction = action({
  args: {},
  handler: async (ctx, args) => {
    console.log("This action does not return anything");
    return null;
  },
});
```

### 7.2. HTTP endpoints (`httpAction`)

* Define in `convex/http.ts` using `httpRouter`:

```ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";

const http = httpRouter();

http.route({
  path: "/echo",
  method: "POST",
  handler: httpAction(async (ctx, req) => {
    const body = await req.bytes();
    return new Response(body, { status: 200 });
  }),
});

export default http;
```

Notes:

* Path is **exact** (`/api/someRoute` → route is exactly that).
* HTTP actions can stream up to **20MiB** out.

---

## 8. Scheduling: cron + scheduler

### 8.1. Cron jobs (`convex/crons.ts`)

Basic pattern:

```ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";
import { internalAction } from "./_generated/server";

const empty = internalAction({
  args: {},
  handler: async (ctx, args) => {
    console.log("empty");
    return null;
  },
});

const crons = cronJobs();

// Run internal.crons.empty every 2 hours.
crons.interval("delete inactive users", { hours: 2 }, internal.crons.empty, {});

export default crons;
```

Rules:

* Use **only** `crons.interval` or `crons.cron` (skip `crons.hourly/daily/weekly` helpers).
* Cron functions should be **internal** (`internalQuery`, `internalMutation`, `internalAction`).
* If a cron calls an internal function in the same file, still import `internal` and call via `internal.*`.

### 8.2. `ctx.scheduler.runAfter`

From mutations or actions, you can enqueue scheduled jobs:

```ts
await ctx.scheduler.runAfter(
  1000, // delay ms
  internal.someFile.someInternalMutation,
  { someArg: "value" },
);
```

Rules & caveats:

* **First arg must be a function reference**, not the function or a string.
* Auth context **does not propagate** into scheduled jobs:

  * `getAuthUserId()` and `ctx.getUserIdentity()` will **always return null**.
  * So scheduled jobs should usually call **internal** functions that don't rely on user identity.
* Do not schedule jobs **too frequently** (not more than once every ~10 seconds for same thing).
* Use scheduled jobs sparingly (e.g. background cleanup, async follow‑ups).

---

## 9. File storage

Core ideas:

* Store **file IDs** (`Id<"_storage">`) in your tables, not URLs.
* Use `_storage` system table + `ctx.storage` methods.

Typical pattern (chat app example):

**Backend:**

```ts
export const list = query({
  args: {},
  handler: async (ctx) => {
    const messages = await ctx.db.query("messages").collect();
    return Promise.all(
      messages.map(async (message) => ({
        ...message,
        ...(message.format === "image"
          ? { url: await ctx.storage.getUrl(message.body) }
          : {}),
      })),
    );
  },
});

export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    return await ctx.storage.generateUploadUrl();
  },
});

export const sendImage = mutation({
  args: { storageId: v.id("_storage"), author: v.string() },
  handler: async (ctx, args) => {
    await ctx.db.insert("messages", {
      body: args.storageId,
      author: args.author,
      format: "image",
    });
  },
});
```

**Client:**

1. Call `generateUploadUrl` to get a signed upload URL.
2. `fetch(POST)` the file to that URL.
3. Extract `{ storageId }` from JSON.
4. Call `sendImage` with `storageId`.

Metadata example:

```ts
import { Id } from "./_generated/dataModel";

type FileMetadata = {
  _id: Id<"_storage">;
  _creationTime: number;
  contentType?: string;
  sha256: string;
  size: number;
};

export const exampleQuery = query({
  args: { fileId: v.id("_storage") },
  handler: async (ctx, args) => {
    const metadata: FileMetadata | null = await ctx.db.system.get(args.fileId);
    console.log(metadata);
    return null;
  },
});
```

---

## 10. Limits you must design around

You **must** avoid designs that hit these limits:

* **Function args + return values**:

  * Max **8 MiB**.
* **Arrays**:

  * Max **8192 elements**.
* **Objects**:

  * Max **1024 entries**.
  * Keys must be ASCII, non‑empty, not starting with `$` or `_`.
  * Nesting depth ≤ 16.
* **Documents**:

  * Size < **1 MiB**.
* **Queries/mutations**:

  * Can **read** up to 8 MiB / 16384 docs.
  * Can **write** up to 8 MiB / 8192 docs.
  * Must finish within **1 second**.
* **Actions / HTTP actions**:

  * Up to **10 minutes** runtime.
* **HTTP actions**:

  * Response streaming max **20 MiB**.

Pattern: if you're tempted to store **huge time‑series** or **entire datasets** per document, instead upload a blob to file storage and handle parsing client‑side.

---

## 11. Environment variables & secrets

### 11.1. Reading env vars

You can use `process.env` inside any Convex function:

```ts
import OpenAI from "openai";
import { action } from "./_generated/server";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export const helloWorld = action({
  args: {},
  handler: async (ctx, args) => {
    const completion = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: "Hello, world!" }],
    });
    return completion.choices[0].message.content;
  },
});
```

### 11.2. How to handle secrets (for agents)

When you need external API keys:

1. **Tell the user exactly which env var name** to set (e.g. `OPENAI_API_KEY`, `RESEND_API_KEY`).
2. Explain how to add it in the Convex dashboard:

   * Open **Database tab → Settings (gear) → Environment variables**.
   * Set the variable.
3. Only then use `process.env.VAR_NAME` in your Convex functions.
4. Never hardcode secrets in code.

---

## 12. React client patterns (convex/react)

### 12.1. Basic usage

```tsx
import { useQuery, useMutation } from "convex/react";
import { api } from "../convex/_generated/api";

export default function App() {
  const messages = useQuery(api.messages.list) ?? [];

  const sendMessage = useMutation(api.messages.sendMessage);
  const [text, setText] = useState("");

  return (
    <main>
      <ul>
        {messages.map(m => (
          <li key={m._id}>{m.body}</li>
        ))}
      </ul>
      <form
        onSubmit={async e => {
          e.preventDefault();
          if (!text) return;
          await sendMessage({ body: text, author: "User 1234" });
          setText("");
        }}
      >
        <input value={text} onChange={e => setText(e.target.value)} />
        <button disabled={!text}>Send</button>
      </form>
    </main>
  );
}
```

Key points:

* `useQuery()` returns:

  * The query result, or
  * `undefined` while loading / when unsubscribed.
* It **live‑updates** when data changes.

### 12.2. Never use hooks conditionally

This is a critical pattern from the Chef prompts.

**Bad:**

```tsx
// ❌ Do not conditionally call useQuery
const avatarUrl = profile?.avatarId
  ? useQuery(api.profiles.getAvatarUrl, { storageId: profile.avatarId })
  : null;
```

**Good:**

```tsx
const avatarUrl = useQuery(
  api.profiles.getAvatarUrl,
  profile?.avatarId ? { storageId: profile.avatarId } : "skip",
);
```

* Use `"skip"` as a special argument to avoid running the query.
* This keeps hooks unconditionally called and satisfies React's rules.

### 12.3. Auth-backed queries

Given a Convex auth setup like:

```ts
// convex/auth.ts
export const loggedInUser = query({
  args: {},
  handler: async (ctx) => {
    const userId = await getAuthUserId(ctx);
    if (!userId) return null;
    return await ctx.db.get(userId);
  },
});
```

React:

```tsx
const user = useQuery(api.auth.loggedInUser); // user | null | undefined
```

---

## 13. Auth patterns (Convex Auth)

Common server pattern:

```ts
import { getAuthUserId } from "@convex-dev/auth/server";

export const currentLoggedInUser = query({
  args: {},
  handler: async (ctx) => {
    const userId = await getAuthUserId(ctx);
    if (!userId) return null;

    const user = await ctx.db.get(userId);
    if (!user) return null;

    return user;
  },
});
```

Notes:

* You can only call `getAuthUserId` inside `convex/` functions (query/mutation/internal*).
* Scheduled jobs do **not** carry auth context – `getAuthUserId` returns `null` there.

---

## 14. Convex Components Cheat Sheet

Components are NPM packages that bundle tables + functions in a sandbox. You:

1. Install via `npm install @convex-dev/<component>`.
2. Add to `convex/convex.config.ts` with `app.use(...)`.
3. Expose API through your own `convex/*.ts` file.
4. Use the generated functions via `components.<name>...` references from `./_generated/api`.

### 14.0. CRITICAL: Querying Component Tables

**Component tables are NOT in your main database namespace.** You cannot use `ctx.db.query('componentTable')`.

**WRONG:**
```ts
// This will fail with "Index not found" error!
const user = await (ctx.db as any)
  .query('user')
  .withIndex('email', (q) => q.eq('email', email))
  .first();
```

**CORRECT:**
```ts
import { components } from './_generated/api';

// Inside a mutation or action (which have ctx.runQuery):
const user = await ctx.runQuery(
  components.betterAuth.adapter.findOne,
  {
    model: 'user',
    where: [{ field: 'email', operator: 'eq', value: email }],
  }
);

// For getting by ID:
const org = await ctx.runQuery(
  components.betterAuth.adapter.findOne,
  {
    model: 'organization',
    where: [{ field: '_id', operator: 'eq', value: orgId }],
  }
);

// For membership queries (Better Auth specific):
const memberships = await ctx.runQuery(
  components.betterAuth.membership.listByUserId,
  { userId: userId }
);
```

**Key points:**
- `ctx.runQuery` is only available in mutations and actions, NOT in queries
- For code that needs to run in both contexts, create an internalMutation wrapper
- Component adapter functions vary by component - check the component's API

### 14.1. Presence component (`@convex-dev/presence`)

Use-case: live, real‑time list of users in a room.

**convex/convex.config.ts**

```ts
import { defineApp } from "convex/server";
import presence from "@convex-dev/presence/convex.config";

const app = defineApp();
app.use(presence);
export default app;
```

**convex/presence.ts**

```ts
import { mutation, query } from "./_generated/server";
import { components } from "./_generated/api";
import { v } from "convex/values";
import { Presence } from "@convex-dev/presence";
import { getAuthUserId } from "@convex-dev/auth/server";
import type { Id } from "./_generated/dataModel";

export const presence = new Presence(components.presence);

export const getUserId = query({
  args: {},
  handler: async (ctx) => {
    return await getAuthUserId(ctx);
  },
});

export const heartbeat = mutation({
  args: { roomId: v.string(), userId: v.string(), sessionId: v.string(), interval: v.number() },
  handler: async (ctx, { roomId, userId, sessionId, interval }) => {
    const authUserId = await getAuthUserId(ctx);
    if (!authUserId) throw new Error("Not authenticated");
    return await presence.heartbeat(ctx, roomId, authUserId, sessionId, interval);
  },
});

export const list = query({
  args: { roomToken: v.string() },
  handler: async (ctx, { roomToken }) => {
    const presenceList = await presence.list(ctx, roomToken);
    const listWithUserInfo = await Promise.all(
      presenceList.map(async (entry) => {
        const user = await ctx.db.get(entry.userId as Id<"users">);
        if (!user) return entry;
        return { ...entry, name: user.name, image: user.image };
      }),
    );
    return listWithUserInfo;
  },
});

export const disconnect = mutation({
  args: { sessionToken: v.string() },
  handler: async (ctx, { sessionToken }) => {
    return await presence.disconnect(ctx, sessionToken);
  },
});
```

**React:**

```tsx
import { api } from "../convex/_generated/api";
import usePresence from "@convex-dev/presence/react";
import FacePile from "@convex-dev/presence/facepile";

function PresenceIndicator({ userId }: { userId: string }) {
  const presenceState = usePresence(api.presence, "my-chat-room", userId);
  return <FacePile presenceState={presenceState ?? []} />;
}
```

* Prefer the built‑in **`FacePile`** component unless user explicitly requests custom UI.
* `usePresence` signature:

```ts
usePresence(
  presenceApiRef, // e.g. api.presence
  roomId: string,
  userId: string,
  interval?: number,
  convexUrl?: string,
): PresenceState[] | undefined;
```

### 14.2. ProseMirror / BlockNote component (`@convex-dev/prosemirror-sync`)

Use-case: collaborative rich text editing with Tiptap / BlockNote / BlockNote + Convex.

**Install + config:**

```bash
npm install @convex-dev/prosemirror-sync
```

```ts
// convex/convex.config.ts
import { defineApp } from "convex/server";
import prosemirrorSync from "@convex-dev/prosemirror-sync/convex.config";

const app = defineApp();
app.use(prosemirrorSync);
export default app;
```

**Expose sync API:**

```ts
// convex/prosemirror.ts
import { components } from "./_generated/api";
import { ProsemirrorSync } from "@convex-dev/prosemirror-sync";

const prosemirrorSync = new ProsemirrorSync(components.prosemirrorSync);
export const {
  getSnapshot,
  submitSnapshot,
  latestVersion,
  getSteps,
  submitSteps,
} = prosemirrorSync.syncApi({
  // optional permission callbacks here
});
```

**React BlockNote usage:**

```tsx
import { useBlockNoteSync } from "@convex-dev/prosemirror-sync/blocknote";
import { BlockNoteView } from "@blocknote/mantine";
import { BlockNoteEditor } from "@blocknote/core";
import { api } from "../convex/_generated/api";

function MyComponent({ id }: { id: string }) {
  const sync = useBlockNoteSync<BlockNoteEditor>(api.prosemirror, id);
  return sync.isLoading ? (
    <p>Loading...</p>
  ) : sync.editor ? (
    <BlockNoteView editor={sync.editor} />
  ) : (
    <button onClick={() => sync.create({ type: "doc", content: [] })}>
      Create document
    </button>
  );
}

export function MyComponentWrapper({ id }: { id: string }) {
  return <MyComponent key={id} id={id} />;
}
```

Important:

* `sync.create` expects a **`JSONContent` object**, **not a string**.
* `JSONContent` type must match Tiptap/ProseMirror schema.
* `MyComponentWrapper` with `key={id}` is a workaround to ensure re‑init when doc ID changes.

### 14.3. Resend component (`@convex-dev/resend`)

Use-case: robust email sending with queueing, batching, retries, status tracking.

**Install + config:**

```bash
npm install @convex-dev/resend
```

```ts
// convex/convex.config.ts
import { defineApp } from "convex/server";
import resend from "@convex-dev/resend/convex.config";

const app = defineApp();
app.use(resend);
export default app;
```

**Env vars:**

* `RESEND_API_KEY`
* `RESEND_DOMAIN`
* (optional) `RESEND_WEBHOOK_SECRET`

**Basic usage:**

```ts
// convex/sendEmails.ts
import { components } from "./_generated/api";
import { Resend } from "@convex-dev/resend";
import { internalMutation } from "./_generated/server";

export const resend = new Resend(components.resend, {});

export const sendTestEmail = internalMutation({
  args: {},
  handler: async (ctx) => {
    await resend.sendEmail(
      ctx,
      `Me <test@${process.env.RESEND_DOMAIN}>`,
      "Resend <delivered@resend.dev>",
      "Hi there",
      "This is a test email",
    );
  },
});
```

**Webhook handling (`convex/http.ts`):**

```ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { resend } from "./sendEmails";

const http = httpRouter();

http.route({
  path: "/resend-webhook",
  method: "POST",
  handler: httpAction(async (ctx, req) => {
    return await resend.handleResendEventWebhook(ctx, req);
  }),
});

export default http;
```

* You must instruct the user to:

  * Configure a webhook in the Resend dashboard pointing at `<deployment>.convex.site/resend-webhook`.
  * Enable `email.*` events.
  * Set `RESEND_WEBHOOK_SECRET`.

**Event handling:**

```ts
import { internalMutation } from "./_generated/server";
import { vEmailId, vEmailEvent, Resend } from "@convex-dev/resend";
import { components, internal } from "./_generated/api";

export const resend = new Resend(components.resend, {
  onEmailEvent: internal.sendEmails.handleEmailEvent,
});

export const handleEmailEvent = internalMutation({
  args: { id: vEmailId, event: vEmailEvent },
  handler: async (ctx, args) => {
    console.log("Email event", args.id, args.event);
    // Update status in your own tables if needed
  },
});
```

`ResendOptions` (constructor) includes:

* `apiKey`, `webhookSecret`
* `testMode` (defaults to **true**; must be set to `false` for real emails)
* `onEmailEvent`

---

## 15. Built-in OpenAI/Resend proxies (Chef-style environments)

If your environment provides built‑in proxies:

### 15.1. Built-in OpenAI proxy (`CONVEX_OPENAI_API_KEY`, `CONVEX_OPENAI_BASE_URL`)

Pattern from Chef prompts:

```ts
import OpenAI from "openai";
import { action } from "./_generated/server";
import { v } from "convex/values";

const openai = new OpenAI({
  baseURL: process.env.CONVEX_OPENAI_BASE_URL,
  apiKey: process.env.CONVEX_OPENAI_API_KEY,
});

export const exampleAction = action({
  args: { prompt: v.string() },
  handler: async (ctx, args) => {
    const resp = await openai.chat.completions.create({
      model: "gpt-4.1-nano", // or "gpt-4o-mini"
      messages: [{ role: "user", content: args.prompt }],
    });
    return resp.choices[0].message.content;
  },
});
```

Rules:

* Only **chat completions** API.
* Only `gpt-4.1-nano` and `gpt-4o-mini`.
* If user has their own API key, **prefer that** and standard OpenAI endpoint.

### 15.2. Built-in Resend proxy (Chef-style)

* Environment vars: `CONVEX_RESEND_API_KEY`, `RESEND_BASE_URL`.
* Only supports sending to the **authenticated user's email address**.
* Email always comes from:
  `"Chef Notifications <{DEPLOYMENT_NAME}@convexchef.app>"` behind the scenes.
* If user configures their own `RESEND_API_KEY`, prefer that and tell them they might need to remove `RESEND_BASE_URL` for normal SDK behavior.

---

## 16. Example: AI-powered chat app pattern

A complete pattern from the prompts (simplified):

### Schema (`convex/schema.ts`)

```ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";
import { authTables } from "@convex-dev/auth/server";

const applicationTables = {
  channels: defineTable({
    name: v.string(),
  }),

  messages: defineTable({
    channelId: v.id("channels"),
    authorId: v.optional(v.id("users")),
    content: v.string(),
  }).index("by_channel_and_author", ["channelId", "authorId"]),
};

export default defineSchema({
  ...authTables,
  ...applicationTables,
});
```

### Functions (`convex/functions.ts`)

Key patterns illustrated:

* `getLoggedInUser` helper using `getAuthUserId`.
* `createChannel` mutation using auth.
* `listMessages` query using `withIndex`.
* `sendMessage` mutation:

  * Validates channel & user.
  * Inserts message.
  * Schedules AI response with `ctx.scheduler.runAfter` and `internal.*` reference.
* `generateResponse` **internalAction**:

  * Uses `ctx.runQuery` to load context.
  * Calls OpenAI.
  * Uses `ctx.runMutation` internal write.
* `loadContext` **internalQuery**:

  * Loads recent messages by index.
  * Builds `messages` array for OpenAI with `role: "user" | "assistant"`.
* `writeAgentResponse` **internalMutation**:

  * Writes AI message to `messages`.

Patterns to copy:

* Use internal functions + scheduler for background AI calls.
* Do **not** rely on auth in scheduled jobs.
* Keep AI logic in actions, DB writes in mutations.

---

## 17. Fixing TS2589 "Type instantiation is excessively deep"

Large Convex codebases (50+ exported functions) can trigger TypeScript's recursion limit when computing the generated `api` type. This manifests as TS2589 errors throughout the codebase.

### Root cause

The generated `fullApi` type in `_generated/api.d.ts` grows exponentially with the number of exported functions. TypeScript exhausts its recursion depth computing this massive type.

### Best solution: Enable static API generation

Add a `convex.json` file at your Convex package root:

```json
{
  "codegen": {
    "staticApi": true,
    "staticDataModel": true
  }
}
```

Then regenerate types with `npx convex codegen` or restart `convex dev`.

**Benefits:**
- Eliminates all TS2589 errors immediately
- Greatly improves autocomplete and incremental typechecking performance

**Tradeoffs:**
- Types only update when `convex dev` is running
- Jump-to-definition no longer works (must manually navigate to files)
- Functions default to `v.any()` without explicit return validators

**Important:** When using static API, you must use dot notation for internal references:
```ts
// Bad - string path indexing doesn't work with static API
internal['actions/foo'].myAction

// Good - use dot notation
internal.actions.foo.myAction
```

### Alternative: Add explicit return types to handlers

If you can't use static API, break the type inference chain by adding explicit return types:

**Bad – TypeScript tries to infer the return type:**

```ts
export const get = query({
  args: { id: v.id("items") },
  handler: async (ctx, args) => {
    return await ctx.db.get(args.id);
  },
});
```

**Good – Explicit return type stops inference chain:**

```ts
export const get = query({
  args: { id: v.id("items") },
  handler: async (ctx, args): Promise<Doc<"items"> | null> => {
    return await ctx.db.get(args.id);
  },
});
```

### Common return type patterns

```ts
import type { Doc, Id } from "./_generated/dataModel";

// Single document
handler: async (ctx, args): Promise<Doc<"items"> | null> => { ... }

// Array of documents
handler: async (ctx, args): Promise<Doc<"items">[]> => { ... }

// Custom object
handler: async (ctx, args): Promise<{ id: Id<"items">; name: string } | null> => { ... }

// Void/null return
handler: async (ctx, args): Promise<null> => { ... return null; }

// Insert returns Id
handler: async (ctx, args): Promise<Id<"items">> => { ... }
```

### Helper functions for shared logic

Instead of `ctx.runQuery` within the same file (which creates circular type references), extract shared logic to plain async functions:

```ts
// Bad – circular reference
export const getItem = query({
  handler: async (ctx, args) => {
    return await ctx.runQuery(api.items.getItemInternal, args); // ❌ Circular
  },
});

// Good – helper function
async function getItemById(ctx: QueryCtx, id: Id<"items">): Promise<Doc<"items"> | null> {
  return await ctx.db.get(id);
}

export const getItem = query({
  args: { id: v.id("items") },
  handler: async (ctx, args): Promise<Doc<"items"> | null> => {
    return await getItemById(ctx, args.id);
  },
});
```

### When calling from actions

Use `internalMutation`/`internalQuery` to avoid circular `api` references:

```ts
// dataRooms.ts
export const addDocumentInternal = internalMutation({
  args: { roomId: v.id("dataRooms"), documentId: v.id("documents") },
  handler: async (ctx, args): Promise<null> => {
    // ... implementation
    return null;
  },
});

export const addDocumentFromExternal = action({
  handler: async (ctx, args) => {
    // Use internal.* instead of api.* to avoid circular reference
    await ctx.runMutation(internal.dataRooms.addDocumentInternal, args);
  },
});
```

### Priority order for fixing

1. Start with files that export the most functions
2. Focus on queries/mutations called from actions first
3. Add return types to public API functions before internal ones

---

## 18. Common pitfalls & quick checklist

When your agent is about to emit Convex code, mentally run through this list:

1. **Functions**

   * [ ] Imported from `./_generated/server`?
   * [ ] Using new syntax (`query({ args, handler })`)?
   * [ ] Args have validators (`v.*`)?
   * [ ] Using `internal*` for internal / scheduled / sensitive functions?

2. **Indexes & queries**

   * [ ] No `.filter()` – using `.withIndex()` or `.withSearchIndex()` instead?
   * [ ] No custom `"by_creation_time"` index?
   * [ ] No `_creationTime` in custom index fields?
   * [ ] Using `.order("desc")` when needed?

3. **Actions**

   * [ ] `"use node";` at top of action files that use Node modules?
   * [ ] No `ctx.db` in actions – using `ctx.runQuery`/`ctx.runMutation`?
   * [ ] External APIs called **only** from actions?

4. **Scheduler**

   * [ ] Using function references (`internal.file.fn`) in `runAfter`?
   * [ ] Not relying on auth inside scheduled job handlers?

5. **File storage**

   * [ ] Storing file IDs (`Id<"_storage">`), not URLs?
   * [ ] Using `ctx.storage.generateUploadUrl` + `ctx.storage.getUrl`?

6. **React**

   * [ ] No conditional `useQuery` / `useMutation`?
   * [ ] Use `"skip"` pattern when you want to disable a query?
   * [ ] Handling `undefined` from `useQuery` while loading?

7. **Limits**

   * [ ] Not returning or passing massive blobs (> 8MiB) in args/returns?
   * [ ] Not reading/writing thousands of docs in a single query/mutation in a way that might hit limits?

8. **Secrets**

   * [ ] Instructed user to set env vars for keys instead of hardcoding?
   * [ ] Using `process.env` for secrets?

9. **Dynamic Imports**

   * [ ] No `await import()` in queries or mutations? (Only actions support dynamic imports)
   * [ ] Using static imports at top of file for all query/mutation dependencies?

10. **Convex Components (e.g., Better Auth)**

    * [ ] Not using `ctx.db.query('componentTable')` to query component tables?
    * [ ] Using `ctx.runQuery(components.<name>.adapter.findOne, {...})` for component table lookups?
    * [ ] Component table queries only from mutations/actions (which have `runQuery`)?

If you keep these rules in your "Convex mental model", you'll produce way fewer broken Convex apps and your users will be much happier.
