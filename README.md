# @requence/service

Connects a TypeScript / Bun service to the Requence operator: it receives messages, runs your
handler, and delivers the result back.

> **This README is a getting-started guide.** The reference — every context method, the store,
> callbacks, surfaces, deferred delivery, the CLI — lives at
> **<https://docs.requence.cloud>**.

## Installation

```bash
npm install @requence/service
```

## Authentication

Every service needs an **access token** to connect. Copy it from the **Services** list view in
the Requence UI by clicking **Copy credentials**.

The token is resolved in this order:

1. `accessToken` option passed to `createService()`
2. `REQUENCE_SERVICE_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.service.accessToken` or `requence.accessToken` in `package.json`

```bash
REQUENCE_SERVICE_ACCESS_TOKEN=your-token bun run index.ts
```

## Your first service

Create the service in the Requence UI first — its input schema, configuration schema and
outputs. Then implement the version:

```typescript
import { createService } from '@requence/service'

createService('1.0.0', (ctx) => {
  return { message: `Hello, ${ctx.input.name}!` }
})
```

The second argument to `createService` is the **version** of the service definition you are
implementing. When Requence dispatches a message, the handler runs and the return value is sent
back as the result.

Generate types for `ctx.input`, `ctx.configuration` and `ctx.toOutput()` from the schemas you
defined in the UI:

```bash
npx requence-service generate-types
```

## Where to read next

| Topic                                                                    | Page                                                        |
| ------------------------------------------------------------------------ | ----------------------------------------------------------- |
| The full walkthrough                                                     | [Getting Started][started]                                  |
| `ctx.input`, `ctx.debug`, `retry`, `abort`, `skip`, `toOutput`, `defer`   | [Context API][context]                                      |
| `prefetch`, `connectionTimeout`, `silent`, TLS via environment           | [Service Options][options]                                  |
| `service.open()`, `service.close()`                                      | [Starting & Stopping][lifecycle]                            |
| Generator handlers and `ctx.terminated`                                  | [Continuous Mode][continuous]                               |
| `ctx.store` — what a node remembers past this message                    | [The Store][store] · [`ctx.store`][context-store]           |
| `createDurableCallback`, `.for(…)`, the per-version queue                | [Callbacks][callbacks]                                      |
| `createUI`, `ctx.render`, handler props                                  | [Surfaces][surfaces]                                        |
| `ctx.defer()` → `service.act()`                                          | [Deferred Delivery][deferred]                               |
| `wrapIterable`, `asyncEventEmitter`                                      | [Utility Helpers][utilities]                                |
| Running locally against real tasks                                       | [Dev Overlay][overlay]                                      |
| `requence-service generate-types`                                        | [Typing / CLI][typing]                                      |

[started]: https://docs.requence.cloud/implementation/getting-started/
[context]: https://docs.requence.cloud/implementation/context-api/
[context-store]: https://docs.requence.cloud/implementation/context-api/#ctxstoreprivatekey-binding--ctxstoresharedkey-binding
[options]: https://docs.requence.cloud/implementation/advanced/service-options/
[lifecycle]: https://docs.requence.cloud/implementation/advanced/
[continuous]: https://docs.requence.cloud/planning/continuous-mode/
[store]: https://docs.requence.cloud/planning/store/
[callbacks]: https://docs.requence.cloud/implementation/callbacks/
[surfaces]: https://docs.requence.cloud/implementation/surfaces/
[deferred]: https://docs.requence.cloud/implementation/advanced/deferred-delivery/
[utilities]: https://docs.requence.cloud/implementation/advanced/utilities/
[overlay]: https://docs.requence.cloud/implementation/advanced/dev-overlay/
[typing]: https://docs.requence.cloud/implementation/typing-cli/
