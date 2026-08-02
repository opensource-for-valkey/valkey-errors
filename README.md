# valkey-errors

Shared error classes for Valkey clients and parsers.

Every layer of a Valkey client stack — parser, client, application — needs to signal and react to failures with different meanings: the server refused a command, the protocol stream is corrupt, a command never received an answer. This package defines those failure types once, in a single dependency-free module, so every layer throws the exact same classes and consumers can identify failures reliably with `instanceof` instead of matching on message strings.

## Install

```bash
npm install valkey-errors
```

## Usage

```js
const { ValkeyError, ReplyError, ParserError, AbortError, InterruptError } = require('valkey-errors')

client.get('key').catch(err => {
  if (err instanceof ReplyError)  { /* server refused the command — handle in app logic */ }
  if (err instanceof ParserError) { /* protocol stream corrupt — reconnect */ }
  if (err instanceof AbortError)  { /* no answer came — retry if idempotent */ }
})
```

## Error classes

| Class            | Extends       | Meaning                                                                 |
| ---------------- | ------------- | ----------------------------------------------------------------------- |
| `ValkeyError`    | `Error`       | Base class. `err instanceof ValkeyError` catches anything from the Valkey stack. |
| `ReplyError`     | `ValkeyError` | The server received the command and deliberately refused it (RESP `-` reply, e.g. `WRONGTYPE`, `ERR unknown command`). Connection and protocol are healthy — handle in application logic. |
| `ParserError`    | `ValkeyError` | Incoming bytes do not match the RESP protocol. Carries `buffer` (the bytes under parse) and `offset` (exact failure position). The stream is unreliable — destroy the connection and reconnect. |
| `AbortError`     | `ValkeyError` | A command was still pending when the connection closed. The server may or may not have executed it — retry only if the command is idempotent. |
| `InterruptError` | `AbortError`  | A pending command was deliberately interrupted (e.g. a forced shutdown). Handlers for `AbortError` cover it automatically. |

Clients may decorate thrown errors with context: `err.command`, `err.args`, `err.code` on `ReplyError`/`AbortError`, and `err.origin` on `InterruptError`.

## Performance

`ReplyError` and `ParserError` are constructed on the hot path — a busy client can create thousands per second. Their constructors temporarily cap `Error.stackTraceLimit` at 2, keeping the throw site in the trace while making construction cheap. Global stack depth is untouched.

## TypeScript

Type declarations are included (`index.d.ts`) — no separate `@types` package needed.

## License

[MIT](LICENSE)
