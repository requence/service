# @requence/service

This package connects a TypeScript / Bun service to the Requence operator. It manages message retrieval, processing, and result delivery.

## Installation

```bash
npm install @requence/service
```

## Authentication

Every service needs an **access token** to connect. Copy it from the **Services** list view in the Requence UI by clicking **Copy credentials**.

The token is resolved in this order:

1. `accessToken` option passed to `createService()`
2. `REQUENCE_SERVICE_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.service.accessToken` or `requence.accessToken` in `package.json`

```bash
REQUENCE_SERVICE_ACCESS_TOKEN=your-token bun run index.ts
```

## Usage

```typescript
import { createService } from '@requence/service'

createService('1.0.0', (ctx) => {
  return { message: `Hello, ${ctx.input.name}!` }
})
```

The second argument to `createService` is the **version** of the service definition you are implementing. When Requence dispatches a message, the handler runs and the return value is sent back as the result.

### Options object

Instead of a bare version string, you can pass an options object:

```typescript
import { createService } from '@requence/service'

createService(
  {
    version: '1.0.0',
    prefetch: 5, // process up to 5 messages in parallel (default: 1)
    connectionTimeout: 30_000, // ms to wait before throwing (default: 3000)
    silent: false, // suppress lifecycle logs
  },
  async (ctx) => {
    return processData(ctx.input)
  },
)
```

### TLS connection options via environment

For `amqps://` connections, TLS settings can be overridden with environment
variables — useful for supplying a private CA or relaxing verification in a
deployment without changing code:

| Variable              | Effect                                                                |
| --------------------- | --------------------------------------------------------------------- |
| `CA`                  | Inline PEM certificate(s) to trust as the CA.                         |
| `CA_FILE`             | Path to a PEM file to trust as the CA (`CA` wins when both are set).  |
| `REJECT_UNAUTHORIZED` | Set to `false`/`0`/`no` to disable certificate verification.          |

These are a **fallback**: any matching value passed in code via
`connectionOptions` (`ca`, `rejectUnauthorized`) takes precedence.

### Return value

`createService` returns a `ServiceApi` object immediately (the connection is established in the background):

```typescript
const service = createService('1.0.0', handler)

// wait for the connection to be ready
await service.open()

// gracefully close the connection
await service.close()
```

## Context API

Every handler receives a `ctx` object.

### Data access

| Property | Description |
|---|---|
| `ctx.input` | The input data routed to this service node from the task template |
| `ctx.configuration` | The static configuration set on the service node in the UI |
| `ctx.taskId` | The unique ID of the current task execution |

### Logging

`ctx.debug` sends log messages to the Requence UI in real time:

```typescript
ctx.debug.log('Processing started')
ctx.debug.info('Step complete', { step: 1 })
ctx.debug.warn('Something looks off')
ctx.debug.error('An error occurred', error)
```

### Flow control

#### `ctx.retry(delay?)`

Instructs Requence to retry this service after an optional delay in milliseconds (minimum 100 ms). No code executes after this call.

```typescript
createService('1.0.0', async (ctx) => {
  const db = await getDbConnection()

  if (!db.isConnected) {
    ctx.retry(2_000) // retry in 2 seconds
  }

  return db.query('SELECT ...')
})
```

> **Note:** It is your responsibility to prevent infinite retry loops.

#### `ctx.abort(reason?)`

Instructs Requence to abort this service immediately. If the service node's **on fail** output is not connected, the entire task fails.

```typescript
createService('1.0.0', (ctx) => {
  if (!ctx.input.requiredField) {
    ctx.abort('Missing required field')
  }

  return processData(ctx.input)
})
```

#### `ctx.skip()`

Puts the message back on the queue without processing it. The next available service instance will receive it instead.

#### `ctx.toOutput(name, value)`

Routes the result to a specific **named output** on the service node. Use this when your service definition has multiple outputs:

```typescript
createService('1.0.0', (ctx) => {
  if (ctx.input.type === 'pdf') {
    return ctx.toOutput('pdf', { url: '...' })
  }

  return ctx.toOutput('other', { raw: ctx.input })
})
```

#### `ctx.defer(reason?)`

Marks the message as **deferred**. The service acknowledges the message but signals that the result will be delivered later via `service.act()`. Returns a **message key** that you must store and pass to `act()`:

```typescript
createService('1.0.0', (ctx) => {
  const messageKey = ctx.defer('waiting for external process')
  externalQueue.push({ messageKey, payload: ctx.input })
  // handler returns — message is held open until act() is called
})
```

#### `ctx.terminated`

A `Promise` that resolves when the task is stopped (cancelled via the UI or API, or terminated by another node). Use it in continuous/generator services to know when to stop producing values.

`ctx.terminated.signal` is a native [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) that fires at the same moment — pass it to any abort-aware API:

```typescript
import { setTimeout } from 'node:timers/promises'
import { createService } from '@requence/service'

// Poll every 5 seconds, exit immediately when the task is stopped
createService('1.0.0', async function* (ctx) {
  while (true) {
    await setTimeout(5_000, null, { signal: ctx.terminated.signal })
    yield await pollForUpdates()
  }
})
```

Or race against it directly:

```typescript
const result = await Promise.race([
  fetchData(),
  ctx.terminated,
])
```

> The framework automatically catches the `AbortError` thrown by abort-aware APIs when the task is terminated — no `try/catch` needed inside a generator.

## Continuous (Generator) Mode

When a service node is configured in **continuous mode**, the handler can be an async generator. Each `yield` sends an incremental result to Requence; the final `return` (or generator completion) signals the end of processing:

```typescript
createService('1.0.0', async function* (ctx) {
  for (const chunk of await fetchChunks(ctx.input)) {
    yield { chunk }
  }
})
```

The generator is terminated automatically when `ctx.terminated` resolves. Yielded values are still sent before the generator exits.

## Deferred Delivery via `service.act()`

After deferring a message with `ctx.defer()`, use `service.act()` to deliver the result later — even from a different process run:

```typescript
const service = createService('1.0.0', (ctx) => {
  const messageKey = ctx.defer()
  saveMessageKey(ctx.taskId, messageKey)
})

// ... in a webhook handler or background job:
await service.act(messageKey, async (api) => {
  api.send({ result: 'done' })
  // or: api.sendToOutput('success', { result: 'done' })
  // or: api.abort('something went wrong')
})
```

`act()` also supports returning a value or an iterable directly from the actor function, which behaves the same as calling `api.send()` for each value.

### One instance per defer key — your lock, not ours

> **You own the exclusivity.** You own the pending-key storage, so you own the
> lock. A single connection refuses a key it is already acting on, but that check
> is per-connection and in memory: **two replicas that both load the pending keys
> on boot will both `act()` the same key**, both register callbacks, and both
> render — and whoever rendered last takes the surface over.

Requence does not arbitrate this: enforcing it would need cross-replica holder
state plus a takeover path, so a dead holder could not block a key forever.
Instead the symptom is made loud — when a render replaces a surface that is still
live on another connection, the server logs a `surface takeover` error naming both
connections.

Take a lease on the key in your own store before acting (`SELECT … FOR UPDATE SKIP
LOCKED`, a Redis `SET NX`, one designated attacher), and only act on keys you hold.

`act()` is also the live window of a deferred node, so it can draw and redraw the
node's [surface](#surfaces--a-ui-of-your-own-on-the-node) — see
[`api.render`](#apirenderui-props-options--rendering-from-act).

## Surfaces — a UI of your own on the node

A service can ship its own UI component and have Requence render it **on the
publishing node** in the task canvas, inside a sandboxed iframe. There is no UI
protocol: the contract is **props**, and a function inside them arrives in the
component as an invokable function.

This section is the reference for surface behaviour in every SDK — the Python SDK
(`python/README.md`) publishes the same render payload and answers the same
invocation. The design authority is
[`knowledge/service-ui-surfaces.md`](../../knowledge/service-ui-surfaces.md).

### `createUI(url, defaults?)` — where a component comes from

```typescript
const ui = createUI<{ title: string; at: string; dense: boolean }>(
  // A URL, so it resolves against THIS module — not against whatever directory
  // the service was started from. `{ url, id }` instead overrides the stable id,
  // which defaults to the built file's name.
  new URL('./dist/panel.js', import.meta.url),
  // Optional default props: what does not change from one message to the next.
  { dense: true },
)

// A render carries only what differs.
ctx.render(ui, { title: 'Processing', at: new Date().toISOString() })
```

**A render is a patch.** What the surface shows is the *fold* of three layers —
the UI's defaults, everything previously rendered onto that surface, and this
render — so redrawing one field takes one prop and leaves the rest standing:

```typescript
ctx.render(ui, { title: 'Processing', at: now() })
ctx.render(ui, { at: now() })          // still says "Processing"
ctx.render(ui, { title: 'Done' })      // and still carries the last `at`
```

The merge is **shallow**: a nested object is replaced wholesale, so changing part
of one means restating that subtree. Nothing is ever removed — an omitted prop
means "nothing to say about this", which is what makes a render a patch — so a
prop is taken back by saying something else about it (`{ error: null }`).

Defaults are the **weakest** layer: they are what a surface shows until some
render says otherwise, and a render that overrides one keeps it overridden for
every later render that stays silent about it. The flip side of all of this is
that every render's props are optional in TypeScript, so it can no longer catch a
*first* render that forgot half of them — completing the fold is yours.

The bundle is read and hashed once, at registration, and capped at 1 MiB — an
oversized component fails at startup rather than on its first render in
production. Requence never bundles for you: build the component with your own
toolchain into one ES module with a default `mount(element, ctx)` export.

**In React, you do not write `mount`.** Leave `react`, `react-dom/client`,
`react/jsx-runtime` and `@requence/surface-ui` external and the surface host
resolves them from its shared runtime — which also ships `createSurface`, so the
component is one default export and the surface state is its own props:

```tsx
import { Button, createSurface, Stack, Text } from '@requence/surface-ui'

import type { PropsOf } from '@requence/service'
import type { buttonUI } from './button.ts'

export default createSurface<PropsOf<typeof buttonUI>>(({ props, canInvoke }) => (
  <Stack space={2}>
    <Button disabled={!canInvoke} onClick={() => props.onClick()}>
      {props.label}
    </Button>
    <Text variant="muted">{props.clicks} dispatched</Text>
  </Stack>
))
```

No root, no `useState`, no `useEffect`, no teardown to forget. Anything nested
reads the same state with `useSurface<Props>()` instead of having `canInvoke`
drilled down to it. A bundle that ships its own React writes `mount` itself —
create a root, subscribe to `ctx.onUpdate`, unmount on teardown.

Every component also has a **stable id** — its built file's name, or whatever
`createUI({ url, id })` says. It survives rebuilds and restarts, which the content
hash deliberately does not, and it is what a durable handler is addressed by. Two
handles claiming one id is a startup error, so two `index.js`-shaped components —
or one file registered twice with different defaults — need explicit ids.

### Handler props — where you write the function decides how long it can be pressed

A function in a surface's props arrives in the component as an invokable function.
There are two kinds, and you pick by **where you write it**:

```typescript
const askUI = createUI<{ question: string; onSubmit(answer: string): void }>(
  new URL('./dist/ask.js', import.meta.url),
  {
    // DURABLE. Module scope, so any instance of this version can run it — this
    // one, or the one that replaces it in a redeploy. An invocation waits in the
    // version's handler queue for ~60s, so the handler below is free to defer
    // and exit, and a rolling restart is invisible to the viewer.
    onSubmit(answer) {
      void record(answer)
    },
  },
)

const service = createService('1.0.0', function* (ctx) {
  ctx.render(askUI, {
    question: 'ship it?',
    // CONNECTION-BOUND. Closes over this running handler, so only this process
    // can serve it and it dies when the handler returns — which is also why it
    // needs a generator handler.
    onSubmit: (answer) => ctx.debug.log(answer),
  })
})
```

A durable handler gets **no injected context**: it reaches `service.act()`, your
stores, anything else through its own closure, exactly as a module-scope function
would in any other process of the version. An invocation naming a component or a
prop path this process does not have is reported on stderr and dropped — the click
behind it was accepted long before, so there is no request left to fail.

Consuming the handler queue is automatic. It is `service-<name>@<version>-handlers`,
one per version, created by Requence when the version is provisioned and asserted
again by the SDK on connect.

**A durable handler still needs someone home.** The node can wait days; the
*click* cannot. While no instance of the version is running, every handler prop of
that version's surfaces reports `reachable: false` and the component withholds its
control — and an invocation that does get published expires in the queue after
~60s and is reported back to the viewer beside the surface, in Requence's own
error box, rather than running whenever somebody next deploys. So
the queue rides out a redeploy, not a weekend. Scale your version to zero and the
buttons it drew go quiet until you scale it back up, at which point they come back
on the open page with no reload.

### `api.render(ui, props, options?)` — rendering from `act()`

A deferring handler cannot register a *connection-bound* handler at all (they die
the moment it returns), so a surface whose buttons are written inline has to be
rendered from the [actor](#deferred-delivery-via-serviceact) — where they belong to
the `act()` call and live exactly as long as it does. (A handler written in the UI's
own defaults is durable and needs none of this.)

```typescript
const ui = createUI(new URL('./dist/button.js', import.meta.url))

const service = createService('1.0.0', (ctx) => {
  pending.save(ctx.defer('waiting for a click'))
})

await service.open()

// On boot AND after each defer — this loop is what survives a restart.
for (const key of await pending.all()) {
  void attach(key)
}

async function attach(key: string) {
  const clicks = asyncEventEmitter<{ timestamp: number }>()

  await service.act(key, async (api) => {
    api.render(ui, {
      onClick: createCallback<{ timestamp: number }>((arg) => clicks.push(arg)),
    })

    for await (const click of clicks) {
      api.send(click) // completes the deferred node
      break
    }
  })

  await pending.remove(key)
}
```

Restart the process, reload the keys, `act()` again: the new render — fresh
connection, fresh callbacks — replaces the stale surface, and the browser needs to
know nothing about it. `props` is type-checked against `createUI<Props>` exactly as
it is for `ctx.render`.

### Three rules a component author does not get to skip

The design took these deliberately, and none of them is enforced for you.

**1. A handler's argument comes from the browser, and it is forgeable.** A viewer
with `task: UPDATE` can invoke any handler reference the stored props hold, with
anything under the byte cap — the component that was rendered is not what sends it.
Requence bounds *which* handler (only a reference some render actually published)
and *how big* the argument is, and deliberately nothing else: no schema declares an
author's props anywhere on the wire, and a handler is ordinary service code that can
check its own input. **Validate the argument in the handler**, exactly as you would
a webhook body — and keep anything the handler must not be *told* in its closure
rather than in the props. A durable handler reading `arg.deferKey` is fine (a key is
single-use and belongs to that node); the same handler reading `arg.amount` or
`arg.userId` is not.

**2. An interaction is *accepted*, not awaited.** The click returns the moment the
command is on the task's shard: a handler prop returns `void`, there is no promise
and no request left to answer — a durable handler may run minutes later, in a
process the viewer never had a connection to. **Feedback is a re-render**: state
what happened in the props (`{ submitted: true }`, `{ error: 'unknown address' }`)
and let the component read them through `ctx.onUpdate`. Two consequences a component
must respect — it may not lock itself optimistically (nothing on the server changed,
so no update arrives to unlock it), and a design that needs a synchronous answer
needs a different shape.

**3. Network egress is open, and the sandbox denies credentials.** A component may
`fetch` any URL and load images, fonts and media from any host (plus `data:` and
`blob:`); only `script-src` stays closed, so it can send data out but cannot pull
code in. Its origin is opaque, so it carries **no cookies** and `Origin: null` — it
acts on its own behalf, never as the viewer, and cannot reach an authenticated
endpoint of your deployment. Therefore **never put a credential in props for the
component to fetch with**: the service fetches and re-renders. The flip side is the
accepted cost — third-party component code holds the props you sent it and can post
them anywhere, unaudited.

## Dev Overlay

When developing locally alongside a production service, use a **dev token** (your personal access token) to register a dev instance. Requence routes your own tasks to the dev instance instead of the production pool:

```typescript
import { createService } from '@requence/service'

createService(
  {
    version: '1.0.0',
    devToken: process.env.REQUENCE_DEV_TOKEN,
  },
  (ctx) => ctx.input,
)
```

The dev token is resolved in the same order as the access token — option, `REQUENCE_SERVICE_DEV_TOKEN` / `REQUENCE_DEV_TOKEN` env var, or `requence.service.devToken` / `requence.devToken` in `package.json`.

### Component reload

A dev instance watches the built file behind every `createUI`. Rebuild it and the
frame swaps to the new bundle in place — same props, same handlers, same node,
no re-render and no new task.

It reaches every surface your dev session ever drew, and nothing else: a live
panel, a form on a node that is still deferred, and a surface on a task that
finished last week all swap alike, while a production surface of the same
component is untouched. Restarting your service does not strand the surfaces it
drew a moment before.

One limit worth knowing: what swaps is the code, never the props. A component
that only shows what the last render sent it will show exactly that, until your
service renders again.

While your dev process is stopped, the surfaces it drew report their handlers as
unreachable and say so in those terms — *the dev process that drew this surface is
not running* — not as a missing deployment. Start it again and they come back.

## CLI — Generate Types

The package ships with a CLI that generates TypeScript types derived from the schemas you defined in the Requence UI (input, configuration, and outputs):

```bash
npx requence-service generate-types
```

This creates a `requence-env.d.ts` file in your project root. TypeScript picks it up automatically — `ctx.input`, `ctx.configuration`, and `ctx.toOutput()` calls become fully typed.

### Options

| Option | Default | Description |
|---|---|---|
| `--access-token` | — | Service access token (falls back to env / config file) |
| `--dev-token` | — | Personal access token for branch-specific types |
| `--outfile` | `requence-env.d.ts` | Name of the generated type file |
| `--outdir` | `.` | Directory to write the type file to |
| `--watch` | `false` | Watch for schema changes and regenerate automatically |
| `--clear` / `--no-clear` | `true` | Clear the terminal on watch updates |

### Watch mode

```bash
npx requence-service generate-types --watch
```

The CLI connects via server-sent events and regenerates types whenever a schema changes in the UI.

## Utility Helpers

### `wrapIterable(iterable)`

Wraps any `Iterable` or `AsyncIterable` so that iteration stops automatically when `ctx.terminated` resolves:

```typescript
import { wrapIterable } from '@requence/service'

createService('1.0.0', async function* (ctx) {
  for await (const event of wrapIterable(eventStream)) {
    yield processEvent(event)
  }
})
```

### `asyncEventEmitter(initial?)`

Creates an async-iterable event emitter that integrates with `ctx.terminated`. Push values from outside (e.g. callbacks or event listeners) and iterate them inside the generator:

```typescript
import { asyncEventEmitter, createService } from '@requence/service'

createService('1.0.0', async function* (ctx) {
  const emitter = asyncEventEmitter<string>()

  externalSource.on('data', (value) => emitter.push(value))

  for await (const value of emitter) {
    yield { value }
  }
})
```

The iterator exits automatically when the task is terminated.

## Full Example

```typescript
import { createService } from '@requence/service'
import { db } from './database.js'

createService(
  {
    version: '1.2.3',
    prefetch: 2, // process 2 messages in parallel
  },
  async (ctx) => {
    if (!db.isConnected) {
      ctx.retry(2_000) // wait 2s for the DB to recover
    }

    if (!ctx.input.ocrData) {
      ctx.abort('OCR data is mandatory')
    }

    const result = await db.getDataBasedOnOcr(ctx.input.ocrData)

    return result
  },
)
```
