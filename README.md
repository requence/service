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
