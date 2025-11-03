# RxJS in React — WebSockets Cheat Sheet (TypeScript-first)

A practical, copy-pasteable reference for building real-time React apps with RxJS and WebSockets. Focuses on **typed streams**, **reconnection**, **multicasting**, **performance**, and **clean React integration**.

---

## Install & Project Setup

```bash
npm i rxjs
# If you’ll use the RxJS WebSocket helper:
# (already part of rxjs, no extra install needed)
```

`tsconfig` tips:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

---

## Core RxJS Primitives Recap

* **Observable<T>**: a lazy stream of `T` values over time.
* **Observer**: `{ next, error, complete }`.
* **Subject<T>**: both `Observable` **and** `Observer`. Use for **manual pushes** or **multicast**.
* **BehaviorSubject<T>**: Subject with **current value** (good for state).
* **Operators**: pure functions like `map`, `filter`, `scan`, `retryWhen`, `switchMap`, etc.
* **Hot vs Cold**: WebSocket streams are typically **hot** (shared source); you’ll often **multicast** (`share`, `shareReplay`).

---

## Using RxJS WebSocket Helper

Prefer RxJS’s `webSocket` over the raw `WebSocket` API; you get an `Observable` of messages and a `next()` sink for sending.

```ts
import { webSocket, WebSocketSubject } from "rxjs/webSocket";

type ServerMsg =
  | { type: "hello"; id: string }
  | { type: "tick"; t: number }
  | { type: "error"; message: string };

type ClientMsg =
  | { type: "subscribe"; topic: string }
  | { type: "ping"; ts: number };

const WS_URL = "wss://example.com/stream";

const socket$: WebSocketSubject<ServerMsg | ClientMsg> = webSocket({
  url: WS_URL,
  // Optional (de)serializers for JSON/Protobuf/etc:
  deserializer: e => JSON.parse(e.data) as ServerMsg,
  serializer: (v: ClientMsg) => JSON.stringify(v),
  // Handy hooks:
  openObserver: { next: () => console.log("WS open") },
  closeObserver: { next: () => console.log("WS closed") },
});
```

Send data:

```ts
socket$.next({ type: "subscribe", topic: "prices" } as ClientMsg);
```

Receive data:

```ts
const sub = socket$.subscribe({
  next: msg => console.log("msg", msg),
  error: err => console.error("ws error", err),
  complete: () => console.log("ws complete"),
});
```

Cleanup:

```ts
sub.unsubscribe();
socket$.complete(); // closes the socket
```

---

## Reconnection Patterns (Exponential Backoff)

```ts
import { timer, defer } from "rxjs";
import { retryWhen, scan, tap, delayWhen } from "rxjs/operators";

function withBackoff<T>(maxRetries = Infinity, baseMs = 500, factor = 2) {
  return retryWhen<T>(errors =>
    errors.pipe(
      scan((acc, err) => {
        console.warn("WS error, retrying", { attempt: acc.attempt + 1, err });
        return { attempt: acc.attempt + 1 };
      }, { attempt: 0 }),
      delayWhen(({ attempt }) => timer(baseMs * Math.pow(factor, attempt))),
      // Stop condition if you want to cap attempts:
      // takeWhile(({ attempt }) => attempt <= maxRetries)
    )
  );
}

// Usage:
const resilient$ = defer(() => socket$).pipe(withBackoff());
```

> Tip: Create the `webSocket(...)` inside `defer` so each retry gets a fresh connection.

---

## Multicasting & Caching Latest

Multiple React components? Share a single connection:

```ts
import { shareReplay, filter } from "rxjs/operators";

const shared$ = defer(() => webSocket<ServerMsg | ClientMsg>({ url: WS_URL }))
  .pipe(
    withBackoff(),
    shareReplay({ bufferSize: 1, refCount: true }) // last value cached, auto-teardown
  );

// Topic filtering:
const ticks$ = shared$.pipe(
  filter((m): m is Extract<ServerMsg, { type: "tick" }> => (m as any).type === "tick")
);
```

---

## Heartbeats & Keep-Alive

```ts
import { interval, merge } from "rxjs";
import { mapTo, takeUntil } from "rxjs/operators";

function heartbeat(socket: WebSocketSubject<ClientMsg | ServerMsg>, ms = 20_000) {
  const stop$ = socket.closed as unknown as import("rxjs").Observable<unknown>; // or manage your own stop Subject
  return interval(ms).pipe(
    mapTo<ClientMsg>({ type: "ping", ts: Date.now() }),
    takeUntil(stop$),
  ).subscribe(msg => socket.next(msg));
}
```

---

## Subprotocols & Multiplexing

Use `multiplex` to join/leave topics on a single WS:

```ts
const prices$ = socket$.multiplex(
  () => ({ type: "subscribe", topic: "prices" } as ClientMsg),
  () => ({ type: "unsubscribe", topic: "prices" } as ClientMsg),
  (msg: ServerMsg) => msg.type === "tick"
);
```

---

## Backpressure & UI Performance

* **Reduce render frequency** for fast streams:

  * `auditTime(100)` – emit last value every 100ms.
  * `throttleTime(100)` – emit first value then ignore for 100ms.
  * `sampleTime(100)` – sample the latest every 100ms.
* **Batch updates** for lists/tables:

  * `bufferTime(200)` then `mergeMap(batch => from(batch))` or accumulate with `scan`.
* **Use React state setters minimally**; prefer a **single source** (e.g., `useState` with reducer) updated at a **fixed cadence**.

---

## A Reusable React Hook: `useWebSocketStream`

```ts
import * as React from "react";
import { defer, Subscription, Observable } from "rxjs";
import { webSocket, WebSocketSubject } from "rxjs/webSocket";
import { shareReplay } from "rxjs/operators";

type UseWSOptions<MIn, MOut> = {
  url: string;
  // Optional transforms:
  select?: (msg: MOut) => unknown;         // projector to narrow data
  onOpen?: () => void;
  onClose?: () => void;
  deserializer?: (e: MessageEvent) => MOut;
  serializer?: (v: MIn) => string | ArrayBufferLike | Blob | ArrayBufferView;
};

export function useWebSocketStream<MIn = unknown, MOut = unknown>(
  opts: UseWSOptions<MIn, MOut>
) {
  const { url, select, onOpen, onClose, deserializer, serializer } = opts;

  const socketRef = React.useRef<WebSocketSubject<MIn | MOut> | null>(null);

  // Stable shared stream
  const streamRef = React.useRef<Observable<MOut> | null>(null);

  if (!streamRef.current) {
    streamRef.current = defer(() =>
      webSocket<MIn | MOut>({
        url,
        deserializer: deserializer ? e => deserializer(e as MessageEvent) : undefined,
        serializer: serializer ? (v: any) => serializer(v) : undefined,
        openObserver: onOpen ? { next: onOpen } : undefined,
        closeObserver: onClose ? { next: onClose } : undefined,
      })
    )
      .pipe(shareReplay({ bufferSize: 1, refCount: true })) as unknown as Observable<MOut>;
  }

  React.useEffect(() => {
    // ensure single socket instance for send()
    socketRef.current = webSocket<MIn | MOut>({
      url,
      deserializer: deserializer ? e => deserializer(e as MessageEvent) : undefined,
      serializer: serializer ? (v: any) => serializer(v) : undefined,
      openObserver: onOpen ? { next: onOpen } : undefined,
      closeObserver: onClose ? { next: onClose } : undefined,
    });

    return () => {
      socketRef.current?.complete();
      socketRef.current = null;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [url]); // recreate on URL change

  const send = React.useCallback((msg: MIn) => {
    socketRef.current?.next(msg);
  }, []);

  // Hook consumer subscribes via `useEffect`
  const useSubscribe = <T,>(
    projector: (o: Observable<MOut>) => Observable<T>,
    onValue: (v: T) => void
  ) => {
    React.useEffect(() => {
      const src = projector(streamRef.current!);
      let sub: Subscription | undefined;
      sub = src.subscribe(onValue);
      return () => sub?.unsubscribe();
    }, [projector, onValue]);
  };

  return { send, useSubscribe };
}
```

**Usage:**

```ts
type InMsg = { type: "subscribe"; topic: string } | { type: "ping"; ts: number };
type OutMsg = { type: "tick"; t: number; symbol: string };

function Ticker() {
  const { send, useSubscribe } = useWebSocketStream<InMsg, OutMsg>({ url: WS_URL });

  const [tick, setTick] = React.useState<OutMsg | null>(null);

  useSubscribe(
    o => o, // no transform
    v => setTick(v)
  );

  React.useEffect(() => {
    send({ type: "subscribe", topic: "AAPL" });
  }, [send]);

  if (!tick) return <div>…</div>;
  return <div>{tick.symbol}: {tick.t}</div>;
}
```

> For **AG Grid / table UIs**: project the stream with `bufferTime(200)` then `scan` into a list to avoid per-row state churn.

---

## Example: Accumulating Streamed Rows (AG Grid-friendly)

```ts
import { bufferTime, filter, mergeMap, scan } from "rxjs/operators";
import { Observable, from } from "rxjs";

type Trade = { id: string; symbol: string; price: number; qty: number; ts: number };
type OutMsg = { type: "trade"; payload: Trade };

function useTrades(url: string) {
  const { send, useSubscribe } = useWebSocketStream<unknown, OutMsg>({ url });
  const [rows, setRows] = React.useState<Trade[]>([]);

  useSubscribe(
    (o: Observable<OutMsg>) =>
      o.pipe(
        filter(m => m.type === "trade"),
        bufferTime(200),
        filter(batch => batch.length > 0),
        mergeMap(batch => from(batch.map(b => b.payload))),
        scan<Trade, Trade[]>(
          (acc, tr) => {
            const i = acc.findIndex(x => x.id === tr.id);
            if (i === -1) acc.push(tr);
            else acc[i] = tr;
            return acc;
          },
          []
        )
      ),
    setRows
  );

  return { rows, send };
}
```

---

## Authentication & Token Refresh

* **Query param** (`wss://.../stream?token=JWT`): simplest but rotates with reconnects.
* **Header via subprotocols**: depends on server support.
* **Message-based auth**: send `{type: "auth", token}` then server promotes connection.

If tokens expire:

```ts
import { switchMap, repeatWhen } from "rxjs/operators";
import { from, timer } from "rxjs";

const authed$ = from(refreshToken()).pipe(
  switchMap(token => webSocket({ url: `${WS_URL}?token=${token}` })),
  // when complete (server closes), wait and repeat:
  repeatWhen(() => timer(1000))
);
```

---

## JSON vs Protobuf/Binary

* Replace `serializer`/`deserializer`:

  * JSON: `JSON.stringify/parse`
  * Protobuf: `serializer: msg => MyMsg.encode(msg).finish()`; `deserializer: e => MyMsg.decode(new Uint8Array(e.data))`
* Set WS `binaryType = "arraybuffer"` if using binary.

---

## Error Handling

* Use **discriminated unions** from the server, e.g. `{ type: "error", code, message }`.
* In RxJS, handle at the **operator level** (e.g., `catchError`, `retryWhen`) rather than in React state, unless you need to show UI.

```ts
import { catchError, of } from "rxjs";

const safe$ = shared$.pipe(
  catchError(err => {
    // log/report
    return of({ type: "error", message: String(err) } as const);
  })
);
```

---

## Rendering Rate Control (Important!)

When the feed is very busy:

```ts
import { auditTime } from "rxjs/operators";

useSubscribe(
  o => o.pipe(auditTime(100)), // at most 10 FPS
  setValue
);
```

---

## Clean React Integration Checklist

* In `useEffect`, always **unsubscribe** on unmount.
* **One connection per app** is usually enough; **multicast** for consumers.
* **Do not** call `setState` on every packet; aggregate or throttle.
* Gate rendering behind **memoization** & **pure components**.

---

## Testing Streams

* Use **marble tests** (Vitest/Jest):

  * `rxjs-marbles` or `@rxjs/testing` test scheduler.
* For components, render with **React Testing Library**, mock the WS with a **Subject**.

---

## Quick Recipes

### 1) One-liner shared socket with backoff

```ts
const shared$ = defer(() => webSocket<OutMsg>({ url: WS_URL }))
  .pipe(withBackoff(), shareReplay({ bufferSize: 1, refCount: true }));
```

### 2) Topic stream

```ts
const topic$ = shared$.pipe(filter(m => m.topic === "EURUSD"));
```

### 3) UI-safe throttled render

```ts
const ui$ = topic$.pipe(auditTime(100));
```

### 4) Send helper

```ts
const send = (m: InMsg) => socket$.next(m);
```

### 5) Graceful close

```ts
// on logout:
socket$.complete();
```

---

## Common Pitfalls & Fixes

* **Multiple sockets created**: put creation in `defer()` + `shareReplay({refCount:true})` and ensure the hook/component doesn’t recreate it every render.
* **Memory leaks**: always unsubscribe in `useEffect` cleanup.
* **UI jank**: throttle/audit/buffer; avoid per-row setState; batch updates.
* **Retry storms**: add **exponential backoff** and a **max cap** if needed.
* **Missing types**: define clear `ClientMsg`/`ServerMsg` unions and **narrow** with type guards.

---

## Minimal End-to-End Example

```ts
// types.ts
export type InMsg =
  | { type: "subscribe"; topic: string }
  | { type: "unsubscribe"; topic: string }
  | { type: "ping"; ts: number };

export type OutMsg =
  | { type: "hello"; id: string }
  | { type: "tick"; topic: string; price: number; ts: number }
  | { type: "error"; message: string };

// ws.ts
import { webSocket } from "rxjs/webSocket";
import { defer } from "rxjs";
import { shareReplay } from "rxjs/operators";
import { withBackoff } from "./backoff";
import type { InMsg, OutMsg } from "./types";

export const ws$ = defer(() =>
  webSocket<InMsg | OutMsg>({
    url: "wss://example.com/quotes",
    serializer: v => JSON.stringify(v),
    deserializer: e => JSON.parse(e.data),
  })
).pipe(
  withBackoff(),
  shareReplay({ bufferSize: 1, refCount: true })
);

// Ticker.tsx
import * as React from "react";
import { ws$ } from "./ws";
import { filter, auditTime } from "rxjs/operators";
import { Subscription } from "rxjs";
import type { OutMsg, InMsg } from "./types";

export function Ticker({ topic }: { topic: string }) {
  const [price, setPrice] = React.useState<number | null>(null);

  React.useEffect(() => {
    const sub = new Subscription();

    // send subscribe (imperative)
    const sender = ws$.subscribe(); // ensure connection
    (ws$ as any).next?.({ type: "subscribe", topic } satisfies InMsg);
    sub.add(sender);

    // receive data (reactive)
    sub.add(
      (ws$ as any)
        .pipe(
          filter((m: OutMsg) => m.type === "tick" && m.topic === topic),
          auditTime(100)
        )
        .subscribe((m: OutMsg) => setPrice(m.price))
    );

    return () => {
      (ws$ as any).next?.({ type: "unsubscribe", topic } satisfies InMsg);
      sub.unsubscribe();
    };
  }, [topic]);

  return <div>{topic}: {price ?? "—"}</div>;
}
```