
# Lesson 42 — Real-Time React: WebSockets, Server-Sent Events, Reconnection, and Render Storms

So far, most of our server communication has looked like:

```text
USER
 ↓
request
 ↓
server
 ↓
response
```

That's a **request/response** model.

Real-time applications introduce something different:

```text
SERVER
  │
  │ events can arrive whenever they happen
  ▼
CLIENT
```

Examples:

```text
chat messages
notifications
live prices
collaborative editing
online/offline presence
delivery tracking
live dashboards
multiplayer state
```

Now the hard part isn't just:

> "How do I receive a message?"

It's:

> **How do I integrate a continuously changing external data source into React without making the UI unstable or wasteful?**

This is where your previous lessons on:

```text
useEffect
useSyncExternalStore
Context
reducers
TanStack Query
optimistic state
render performance
```

all become useful together.

---

# 1. Polling vs real-time

Start with the simplest approach.

You can poll:

```js
setInterval(() => {
  fetch("/api/messages");
}, 5000);
```

Architecture:

```text
every 5 seconds
 ↓
request
 ↓
server
 ↓
response
```

This is easy, but wasteful.

If nothing changes:

```text
request
 ↓
same data
```

You still paid for the request.

And if something changes immediately after your last poll:

```text
t = 0   poll
t = 1   message arrives
t = 2
t = 3
t = 4
t = 5   next poll
```

The UI doesn't learn about the message until the next poll.

---

# 2. Push-based communication

Real-time systems instead keep a connection open.

Conceptually:

```text
client
  │
  ├──────── connection ────────► server
  │
  │◄──── event ────────────────
  │
  │◄──── event ────────────────
  │
  │◄──── event ────────────────
  │
```

The server can send new information whenever necessary.

Two important browser technologies are:

```text
WebSocket
Server-Sent Events (SSE)
```

---

# 3. WebSocket

A WebSocket gives you a persistent, bidirectional connection.

That means:

```text
client → server
server → client
```

at any time.

Example:

```js
const socket =
  new WebSocket("wss://example.com/socket");
```

Then:

```js
socket.addEventListener(
  "message",
  event => {
    console.log(event.data);
  }
);
```

You can also send:

```js
socket.send(
  JSON.stringify({
    type: "typing",
    conversationId: 42
  })
);
```

So:

```text
WebSocket
=
two-way live channel
```

---

# 4. Server-Sent Events

SSE is different.

With:

```js
const source =
  new EventSource("/events");
```

the server can continuously send events to the browser.

But the communication direction is primarily:

```text
server → client
```

So:

```text
WebSocket
→ bidirectional

SSE
→ server pushes events
```

For a notification stream:

```text
server
 ↓
notification
 ↓
browser
```

SSE can be a very natural fit.

---

# 5. Choosing between them

Think about the data flow.

### Chat

You need:

```text
client → server
send message
```

and:

```text
server → client
new message
```

WebSocket is a natural fit.

### Live notification feed

You mostly need:

```text
server → client
new notification
```

SSE can be a natural fit.

### Live stock-like updates

If the browser mostly receives updates:

```text
server → client
```

SSE can work well.

### Collaborative editing

You often need:

```text
client ↔ server
```

WebSocket is generally more appropriate.

The technology should follow the communication model.

---

# 6. The first React mistake: socket inside render

This is bad:

```jsx
function Chat() {
  const socket =
    new WebSocket("wss://example.com/chat");

  return <div>Chat</div>;
}
```

Why?

Every render creates a new connection.

Remember:

```text
render
→ should be pure
```

But:

```js
new WebSocket(...)
```

is an external side effect.

So you get:

```text
render #1
 ↓
connection #1

render #2
 ↓
connection #2

render #3
 ↓
connection #3
```

Now the application has multiple sockets.

This is exactly the kind of bug `useEffect` exists to prevent.

---

# 7. Basic WebSocket integration with `useEffect`

A simple component:

```jsx
import { useEffect, useState } from "react";

function Chat() {
  const [messages, setMessages] =
    useState([]);

  useEffect(() => {
    const socket =
      new WebSocket(
        "wss://example.com/chat"
      );

    socket.addEventListener(
      "message",
      event => {
        const message =
          JSON.parse(event.data);

        setMessages(messages => [
          ...messages,
          message
        ]);
      }
    );

    return () => {
      socket.close();
    };
  }, []);

  return (
    <ul>
      {messages.map(message => (
        <li key={message.id}>
          {message.text}
        </li>
      ))}
    </ul>
  );
}
```

This is the simplest correct pattern.

---

# 8. Execution trace

Mount:

```text
Chat mounts
 ↓
effect runs
 ↓
WebSocket created
 ↓
event listener attached
```

Then server sends:

```json
{
  "id": 1,
  "text": "Hello"
}
```

Browser receives:

```text
message event
 ↓
JSON.parse
 ↓
setMessages(...)
 ↓
React update
 ↓
render
 ↓
new message appears
```

When component unmounts:

```text
cleanup
 ↓
socket.close()
```

That's the lifecycle.

---

# 9. Why functional state updates matter here

Imagine events arrive quickly:

```text
message A
message B
message C
```

You might write:

```js
setMessages([
  ...messages,
  message
]);
```

But the event handler can hold an older state snapshot.

Instead:

```js
setMessages(messages => [
  ...messages,
  message
]);
```

says:

> "Take whatever the latest state is when React processes this update."

This is exactly the functional update concept you learned in Lesson 2.

Real-time systems are one of the places where that knowledge becomes especially useful.

---

# 10. But this simple solution has a scaling problem

Suppose:

```text
1 message/sec
```

Fine.

Now:

```text
100 messages/sec
```

You potentially have:

```text
100 state updates/sec
```

If every event causes:

```text
setMessages(...)
```

React may perform a lot of rendering work.

Now imagine:

```text
500 events/sec
```

Your application could spend more time processing incoming events than rendering useful UI.

This is a **render storm**.

---

# 11. What is a render storm?

A render storm is the practical situation where:

```text
external events
     ↓
too many state updates
     ↓
too many renders
     ↓
too much CPU
     ↓
UI becomes sluggish
```

For example:

```text
WebSocket
  ↓
event
  ↓
setState
  ↓
render
  ↓
event
  ↓
setState
  ↓
render
  ↓
event
  ↓
setState
  ↓
render
```

The problem is not that React can't receive events.

The problem is that we're mapping **every external event directly to a visible UI render**.

---

# 12. Not every event needs to become a render

This is a powerful real-time principle:

> **The event stream and the UI update frequency do not have to be identical.**

Suppose a server sends:

```text
position:
10.1
10.2
10.3
10.4
10.5
...
```

at 120 updates/sec.

Maybe the UI only needs to display:

```text
60 frames/sec
```

Or perhaps even:

```text
15 updates/sec
```

So:

```text
event frequency
≠
render frequency
```

This distinction is fundamental.

---

# 13. Buffering events

Instead of:

```text
event
 ↓
setState
 ↓
render
```

for every message:

```text
event
 ↓
buffer
```

Then periodically:

```text
buffer
 ↓
one state update
 ↓
one render
```

Conceptually:

```text
events:
A B C D E F G
        ↓
      buffer
        ↓
   one UI update
        ↓
      render
```

This is **batching/coalescing**.

---

# 14. Why React's automatic batching isn't the whole solution

React can batch some state updates occurring in the same turn.

But suppose events arrive:

```text
t = 0 ms
t = 15 ms
t = 30 ms
t = 45 ms
...
```

Those are separate asynchronous events.

You shouldn't assume:

```text
100 events
=
1 render
```

React's batching is not an unlimited "render once per burst" mechanism.

The application may need its own event aggregation strategy.

---

# 15. A simple batching approach

For example:

```jsx
const bufferRef = useRef([]);
const [messages, setMessages] =
  useState([]);

useEffect(() => {
  const socket =
    new WebSocket(URL);

  socket.addEventListener(
    "message",
    event => {
      bufferRef.current.push(
        JSON.parse(event.data)
      );
    }
  );

  const interval =
    setInterval(() => {
      if (bufferRef.current.length === 0) {
        return;
      }

      const newMessages =
        bufferRef.current;

      bufferRef.current = [];

      setMessages(messages => [
        ...messages,
        ...newMessages
      ]);
    }, 100);

  return () => {
    socket.close();
    clearInterval(interval);
  };
}, []);
```

Now the flow becomes:

```text
many socket events
       ↓
ref buffer
       ↓
every 100ms
       ↓
single React update
```

This can dramatically reduce render frequency.

---

# 16. Why use a `ref` for the buffer?

Because:

```js
bufferRef.current.push(...)
```

doesn't cause a render.

That's exactly what refs are good at:

```text
persistent mutable storage
+
no render
```

You learned this in Lesson 9.

So:

```text
high-frequency transient data
→ ref

UI state that must be displayed
→ state
```

This is a very useful rule.

---

# 17. But now we have another problem

Suppose the incoming events aren't just messages.

Imagine:

```text
presence events
typing events
cursor events
price ticks
```

The stream itself is conceptually application state.

Maybe the websocket connection shouldn't belong to one component.

Instead:

```text
WebSocket
   ↓
external store
   ↓
many React components
```

This is where Lesson 33 becomes highly relevant.

---

# 18. Real-time data as an external store

Imagine a chat store outside React:

```js
let messages = [];

const listeners =
  new Set();

function getSnapshot() {
  return messages;
}

function subscribe(listener) {
  listeners.add(listener);

  return () => {
    listeners.delete(listener);
  };
}
```

When a socket event arrives:

```js
function onMessage(message) {
  messages = [
    ...messages,
    message
  ];

  listeners.forEach(listener =>
    listener()
  );
}
```

React can subscribe through:

```js
useSyncExternalStore(
  subscribe,
  getSnapshot
);
```

Now:

```text
WebSocket
   ↓
external store
   ↓
useSyncExternalStore
   ↓
React
```

This is often a much cleaner architecture for shared real-time state.

---

# 19. Why this can be better

Suppose you have:

```text
ChatList
UnreadBadge
NotificationPanel
TypingIndicator
Header
```

All want data from the same socket connection.

If each component does:

```js
useEffect(() => {
  new WebSocket(...)
}, []);
```

you might accidentally create:

```text
5 components
 ↓
5 sockets
```

That's usually not what you want.

Instead:

```text
one connection
      ↓
one external store
      ↓
multiple React subscribers
```

Architecture:

```text
                 WebSocket
                     │
                     ▼
              shared store
             /     |      \
            /      |       \
      ChatList   Badge   Indicator
```

Much cleaner.

---

# 20. Centralized connection ownership

A real-time manager might look conceptually like:

```js
class ChatStore {
  socket = null;
  messages = [];
  listeners = new Set();

  connect() {
    if (this.socket) {
      return;
    }

    this.socket =
      new WebSocket(URL);

    this.socket.onmessage =
      event => {
        this.messages = [
          ...this.messages,
          JSON.parse(event.data)
        ];

        this.emit();
      };
  }

  emit() {
    for (const listener of this.listeners) {
      listener();
    }
  }

  subscribe(listener) {
    this.listeners.add(listener);

    return () => {
      this.listeners.delete(listener);
    };
  }

  getSnapshot() {
    return this.messages;
  }
}
```

Then React:

```js
const messages =
  useSyncExternalStore(
    listener =>
      store.subscribe(listener),
    () =>
      store.getSnapshot()
  );
```

Now connection lifecycle is not tied directly to a single component.

---

# 21. This architecture also helps with duplicate subscriptions

Imagine:

```text
ChatList
UnreadBadge
Notifications
```

all subscribe to the same store.

One socket.

Three React consumers.

That's:

```text
1 connection
3 subscriptions
```

instead of:

```text
3 connections
3 subscriptions
```

Huge difference at scale.

---

# 22. The store must still obey snapshot rules

Remember the critical `useSyncExternalStore` rule:

If data hasn't changed, return the same snapshot identity.

Bad:

```js
getSnapshot() {
  return [...messages];
}
```

This creates a new array every time.

React sees:

```text
old snapshot !== new snapshot
```

even if the messages haven't changed.

That can cause endless update behavior or unnecessary work.

Better:

```js
getSnapshot() {
  return messages;
}
```

where `messages` itself is replaced only when the store changes.

Again:

```text
immutable update
+
stable snapshot
```

makes external-store integration much easier.

---

# 23. Reconnection

Real-time connections are not permanent.

The network can disappear.

You might get:

```text
WebSocket
   ↓
OPEN
   ↓
network lost
   ↓
CLOSED
```

Now the application needs:

```text
reconnect
```

A naive approach:

```js
socket.onclose = () => {
  connect();
};
```

can be dangerous.

Why?

Suppose the server is down.

You'll get:

```text
connect
 ↓
fail
 ↓
connect
 ↓
fail
 ↓
connect
 ↓
fail
```

very quickly.

That's a **reconnect storm**.

---

# 24. Exponential backoff

A common solution is increasing the delay:

```text
attempt 1 → wait 1s
attempt 2 → wait 2s
attempt 3 → wait 4s
attempt 4 → wait 8s
attempt 5 → wait 16s
```

Usually you cap it:

```text
maximum = 30s
```

So:

```text
delay =
min(base × 2^attempt, max)
```

You often also add **jitter**, meaning a random small variation, so a huge number of clients don't all reconnect at exactly the same moment.

Conceptually:

```text
backoff
+
jitter
=
less synchronized reconnect pressure
```

This is a distributed-systems problem appearing in your frontend.

---

# 25. Don't reconnect when the browser is offline

You already learned:

```text
navigator.onLine
online
offline
```

and `useSyncExternalStore` for browser external state.

That's useful here.

If:

```text
browser offline
```

don't aggressively reconnect.

Instead:

```text
offline
 ↓
wait
 ↓
browser becomes online
 ↓
attempt reconnect
```

This is one of those cases where two seemingly unrelated lessons fit perfectly together.

---

# 26. Connection state should be modeled explicitly

Instead of:

```js
const [connected, setConnected] =
  useState(false);
```

consider the actual state machine:

```text
idle
connecting
open
closing
closed
retrying
failed
```

Even if you simplify it, the important concept is:

```text
connection state
is not
message state
```

Don't mix:

```js
{
  messages,
  connected,
  retryCount,
  error
}
```

into one giant ambiguous variable if the state transitions become complicated.

A reducer or external-store state machine can be much clearer.

---

# 27. Real-time state as a state machine

For a connection:

```text
DISCONNECTED
     │
     │ connect
     ▼
CONNECTING
     │
     │ success
     ▼
CONNECTED
     │
     │ network failure
     ▼
RECONNECTING
     │
     │ success
     └──────────────► CONNECTED
```

After repeated failures:

```text
RECONNECTING
     │
     │ max retries
     ▼
FAILED
```

This is much easier to reason about than:

```text
if (!socket) ...
if (closed) ...
if (retrying) ...
if (!ready) ...
```

scattered throughout components.

---

# 28. Don't put the socket protocol inside UI components

Bad:

```jsx
function Dashboard() {
  // 200 lines of:
  // socket setup
  // reconnect
  // parse messages
  // deduplication
  // authentication
  // heartbeats
  // buffering
  // state updates
}
```

The component should ideally consume a clean abstraction:

```js
const dashboard =
  useLiveDashboard();
```

while a lower layer manages:

```text
connection
protocol
reconnect
buffering
deduplication
store
```

This is the same principle behind custom hooks:

> Share stateful logic rather than duplicating it.

---

# 29. Custom hook on top of a real-time store

For example:

```js
function useMessages() {
  return useSyncExternalStore(
    subscribe,
    getSnapshot
  );
}
```

Then:

```jsx
function ChatList() {
  const messages =
    useMessages();

  return ...
}
```

And:

```jsx
function UnreadBadge() {
  const messages =
    useMessages();

  return ...
}
```

The components don't know:

```text
WebSocket
reconnect
EventSource
buffer
protocol
```

They only know:

```text
messages
```

That's good abstraction.

---

# 30. Ordering problems

Now we're entering real distributed-system territory.

Suppose messages arrive:

```text
A
B
C
```

but due to network behavior your client processes:

```text
A
C
B
```

Your UI could show them in the wrong order.

Therefore many real-time protocols include:

```text
sequence number
timestamp
version
cursor
event ID
```

For example:

```json
{
  "id": "evt-102",
  "sequence": 102,
  "type": "message",
  "payload": {...}
}
```

You can track:

```text
lastSeen = 101
```

and receive:

```text
102
```

Good.

If next you get:

```text
104
```

you know:

```text
103 is missing
```

Now you may need a recovery mechanism.

---

# 31. Real-time systems need synchronization, not just streaming

This is a profound distinction.

A socket saying:

```text
"here is a new event"
```

doesn't guarantee you have the complete state.

Imagine:

```text
connected
 ↓
receive event 1
 ↓
network drops
 ↓
events 2, 3, 4 happen
 ↓
reconnect
```

Your client only knows:

```text
event 1
```

The server knows:

```text
1, 2, 3, 4
```

So after reconnect you need:

```text
resynchronize
```

possibly by:

```text
refetch snapshot
+
resume event stream
```

This is far more robust than assuming every connection receives every event forever.

---

# 32. Snapshot + event stream

A strong real-time architecture often looks like:

```text
initial snapshot
      ↓
current state
      ↓
subscribe to events
      ↓
apply events
      ↓
disconnect
      ↓
reconnect
      ↓
resync snapshot or missing events
```

This is conceptually:

```text
snapshot
+
incremental updates
```

rather than:

```text
events forever
```

This is one of the most important production patterns for live data.

---

# 33. TanStack Query + WebSocket

Now connect real-time systems to server state.

Suppose:

```text
useQuery(
  ["messages", roomId]
)
```

has:

```text
[A, B, C]
```

Then WebSocket receives:

```text
new message D
```

You can update the query cache:

```js
queryClient.setQueryData(
  ["messages", roomId],
  messages => [
    ...messages,
    newMessage
  ]
);
```

Architecture:

```text
initial HTTP query
       ↓
TanStack Query cache
       ↓
WebSocket events
       ↓
cache update
       ↓
React consumers
```

This is a very strong pattern.

---

# 34. Why use both HTTP and WebSocket?

Because they solve different things.

HTTP:

```text
give me the current snapshot
```

WebSocket:

```text
tell me what changes after that
```

So:

```text
HTTP
→ initial state

WebSocket
→ incremental updates
```

That's often easier to reason about than trying to reconstruct the entire state solely from an event stream.

---

# 35. Example

Initial:

```text
GET /rooms/42/messages

[A, B, C]
```

Store:

```text
messages = [A, B, C]
```

Then WebSocket:

```text
D
```

Cache:

```text
[A, B, C, D]
```

Then:

```text
E
```

Cache:

```text
[A, B, C, D, E]
```

Network drops.

On reconnect:

```text
GET /rooms/42/messages?since=E
```

or:

```text
GET current snapshot
```

depending on the server protocol.

This restores consistency.

---

# 36. Deduplication

Suppose the server sends:

```text
event 100
```

and after reconnect sends it again.

Without deduplication:

```text
message A
message A
```

You have duplicate UI.

So event IDs matter.

You may maintain:

```js
seenEventIds
```

or use the message/entity ID itself.

The general rule:

> **Real-time consumers should be designed to tolerate repeated delivery whenever the protocol can produce it.**

This is another distributed-systems principle appearing in UI architecture.

---

# 37. Optimistic update + real-time confirmation

Now combine:

```text
useOptimistic
+
WebSocket
```

Suppose user sends:

```text
"Hello"
```

Client immediately shows:

```text
Hello
```

with temporary ID:

```text
temp-123
```

Server processes it and broadcasts:

```json
{
  "id": "msg-981",
  "clientMessageId": "temp-123",
  "text": "Hello"
}
```

Client can reconcile:

```text
temp-123
   ↓
msg-981
```

instead of showing two messages.

This is a very common pattern in real-time applications.

---

# 38. Why client-generated IDs are useful

Suppose the client creates:

```text
clientMessageId = "abc-123"
```

and sends:

```json
{
  "clientMessageId": "abc-123",
  "text": "Hello"
}
```

The server responds:

```json
{
  "id": 987,
  "clientMessageId": "abc-123",
  "text": "Hello"
}
```

Now the client knows:

```text
temporary message
      ↕
server message
```

This helps reconcile optimistic UI and server truth.

---

# 39. Real-time typing indicators

Typing indicators are a perfect example of data that should **not** necessarily enter long-term server state.

Suppose:

```text
Alice is typing
```

This is ephemeral.

You probably don't want:

```text
database row:
AliceTyping = true
```

for every keystroke.

Instead:

```text
WebSocket event
 ↓
transient client state
```

and remove it after a timeout.

This gives you another state classification:

```text
persistent server state
vs
ephemeral real-time state
```

That distinction is extremely important.

---

# 40. Not every server event belongs in TanStack Query

This is another important design point.

Use TanStack Query for data that behaves like:

```text
server-owned resource
```

such as:

```text
messages
orders
projects
notifications
```

But something like:

```text
user is typing
cursor moved
mouse position
presence heartbeat
```

may be better represented as:

```text
ephemeral real-time state
```

and handled by a dedicated external store or local state.

Don't put every socket packet into a query cache simply because the query cache exists.

---

# 41. High-frequency events should often bypass normal React state

Examples:

```text
mouse cursor
pointer position
audio levels
game state
live telemetry
```

Suppose:

```text
120 events/sec
```

and you do:

```js
setPosition(event.position);
```

for every event.

That means React participates in all 120 updates.

Sometimes that's fine.

Sometimes it isn't.

A better architecture might be:

```text
WebSocket
 ↓
ref / external store
 ↓
requestAnimationFrame
 ↓
single UI update per frame
```

Now:

```text
120 incoming events
≈
60 visual updates
```

or fewer.

---

# 42. `requestAnimationFrame`

The browser gives you:

```js
requestAnimationFrame(...)
```

which schedules work around the browser's next visual frame.

Conceptually:

```text
events:
1
2
3
4
5

before next frame
   ↓
coalesce latest state

frame
   ↓
render latest position
```

This is especially useful for:

```text
dragging
cursor synchronization
canvas
animations
live telemetry
```

The critical idea is:

> For visual data, synchronize to the display frame rather than blindly rendering for every incoming packet.

---

# 43. Example: latest-value wins

Suppose incoming prices are:

```text
100
101
102
103
104
```

and your UI only needs the current value.

You don't need to render every intermediate number.

Store:

```js
latestPriceRef.current = 104;
```

Then periodically:

```text
render 104
```

This is **coalescing**.

The event stream is:

```text
100 → 101 → 102 → 103 → 104
```

The UI cares about:

```text
104
```

not every historical intermediate value.

---

# 44. Real-time architecture needs to classify data

This is one of the most useful things to do before writing code.

Ask:

### Is this a durable resource?

```text
orders
messages
tasks
projects
```

→ server state / query cache

### Is this ephemeral?

```text
typing
presence
cursor
```

→ local/external transient state

### Is this a high-frequency stream?

```text
telemetry
position
price ticks
```

→ buffer/coalesce/rAF/external store

### Is this an interaction?

```text
send
edit
delete
```

→ mutation / Action / Server Function

This classification prevents architecture from becoming tangled.

---

# 45. WebSocket cleanup

This deserves emphasis.

Bad:

```jsx
useEffect(() => {
  const socket = new WebSocket(URL);

  socket.onmessage = handleMessage;
}, []);
```

No cleanup.

Now navigating away could leave:

```text
old socket
```

alive.

Then the component comes back:

```text
new socket
```

You now have duplicates.

Correct:

```jsx
return () => {
  socket.close();
};
```

The same synchronization principle from ordinary Effects applies to real-time connections.

---

# 46. Strict Mode becomes useful here

In development, Strict Mode may intentionally run effect setup/cleanup cycles in a way that exposes missing cleanup.

Suppose:

```text
effect
 ↓
connect
 ↓
cleanup
 ↓
connect again
```

If you wrote correct cleanup:

```text
close socket
```

everything behaves sensibly.

If not:

```text
connect
connect
```

and you notice duplicate messages.

Strict Mode can therefore expose real resource-lifecycle bugs.

It isn't the root problem.

Missing cleanup is.

---

# 47. Authentication and WebSockets

A real-time connection is still a security boundary.

You might need:

```text
authentication
authorization
```

before allowing the subscription.

Just because the user is on:

```text
/room/42
```

doesn't mean they're authorized for:

```text
room 42 events
```

The server must enforce that.

The same rule from Server Functions applies:

```text
client intent
≠
permission
```

---

# 48. Don't put secrets in the browser

For example, don't do:

```js
new WebSocket(
  `wss://...?apiKey=SUPER_SECRET`
);
```

if that key is meant to remain secret.

Anything sent to the browser is potentially observable by the browser user.

Use appropriate authentication mechanisms and short-lived credentials/tokens where necessary, depending on your backend architecture.

---

# 49. Heartbeats and dead connections

Sometimes a TCP/WebSocket connection can appear open while the application has effectively stopped receiving useful traffic.

Protocols may therefore use:

```text
ping
pong
heartbeat
```

to verify liveness.

Conceptually:

```text
client → ping
server → pong
```

If too many expected responses are missing:

```text
connection considered unhealthy
 ↓
close
 ↓
reconnect
```

This is an infrastructure concern, not a React concern.

React should consume the resulting connection state rather than implementing the entire transport protocol inside a component.

---

# 50. Server-Sent Events have a useful property

SSE uses the browser's `EventSource`.

Example:

```js
const source =
  new EventSource("/events");

source.onmessage = event => {
  const data =
    JSON.parse(event.data);
};
```

Because the browser manages the event-stream protocol for you, SSE can be simpler than implementing a WebSocket client for one-way server updates.

And the server can send named events:

```text
event: notification
data: {...}
```

Then:

```js
source.addEventListener(
  "notification",
  event => {
    ...
  }
);
```

So:

```text
SSE
→ browser-native streaming server events
```

can be a nice fit for some applications.

---

# 51. Reconnection with SSE

`EventSource` has built-in reconnection behavior in the browser.

But you still have to think about:

```text
authorization
server state recovery
duplicate events
stale state
connection lifecycle
```

The browser reconnecting doesn't automatically guarantee the application is synchronized.

Again:

```text
transport reconnection
≠
application state synchronization
```

That's an important distinction.

---

# 52. React should not own the socket protocol

Suppose your server sends:

```json
{
  "type": "message.created",
  "version": 17,
  "payload": {...}
}
```

Your transport layer should ideally handle:

```text
parse
validation
versioning
reconnect
deduplication
ordering
```

and expose a clean domain model:

```js
{
  type: "messageCreated",
  message
}
```

Then React consumes:

```text
domain events
```

rather than raw transport packets.

This creates a clean separation:

```text
transport
 ↓
domain state
 ↓
React
```

---

# 53. Message validation matters

Remember your TypeScript lesson:

```text
TypeScript
≠
runtime validation
```

A WebSocket message comes from outside your application runtime.

This:

```js
const message =
  JSON.parse(event.data);
```

doesn't prove it has the shape you expect.

You should conceptually do:

```text
untrusted event
 ↓
parse
 ↓
validate
 ↓
domain object
 ↓
store
 ↓
React
```

That's especially important for real-time systems because malformed data can arrive at arbitrary times.

---

# 54. TypeScript can help model events beautifully

For example:

```ts
type ServerEvent =
  | {
      type: "message.created";
      payload: Message;
    }
  | {
      type: "message.deleted";
      payload: {
        id: string;
      };
    }
  | {
      type: "user.typing";
      payload: {
        userId: string;
        roomId: string;
      };
    };
```

Then:

```ts
function handleEvent(event: ServerEvent) {
  switch (event.type) {
    case "message.created":
      // event.payload is Message

      break;

    case "message.deleted":
      // event.payload has id

      break;

    case "user.typing":
      // event.payload has userId/roomId

      break;
  }
}
```

This uses the discriminated-union concepts you learned earlier.

Real-time protocols are one of the best places to use them.

---

# 55. State updates should be domain-aware

Don't do:

```js
setState(event);
```

for every packet.

Instead:

```js
switch (event.type) {
  case "message.created":
    addMessage(event.payload);
    break;

  case "message.deleted":
    removeMessage(event.payload.id);
    break;

  case "user.typing":
    updateTyping(event.payload);
    break;
}
```

Now the store has meaningful domain transitions.

This resembles the reducer model:

```text
event
 ↓
state transition
 ↓
new state
```

That's not accidental.

Real-time state is often essentially an event-driven state machine.

---

# 56. Event sourcing — a useful conceptual connection

You don't necessarily need to build event sourcing, but the concept is useful.

Imagine:

```text
initial state
+
event 1
+
event 2
+
event 3
```

can reconstruct current state.

For example:

```text
cart initially empty

ADD item A
ADD item B
REMOVE item A
```

results in:

```text
[B]
```

This is conceptually:

```text
state
+
events
→
new state
```

That's very close to reducer thinking:

```js
(state, action) =>
  nextState
```

So reducers are a surprisingly good mental bridge to real-time event processing.

---

# 57. But reducer and event stream are not identical

A reducer is:

```text
local deterministic state transition
```

A server event stream is:

```text
external asynchronous input
```

The pattern can be:

```text
external event
 ↓
validate
 ↓
dispatch/reduce
 ↓
state transition
```

But the event source and reducer responsibilities should remain separate.

---

# 58. Example: external store with event reducer

Conceptually:

```js
let state = {
  messages: [],
  typingUsers: new Set()
};

function reduce(state, event) {
  switch (event.type) {
    case "message.created":
      return {
        ...state,
        messages: [
          ...state.messages,
          event.payload
        ]
      };

    case "message.deleted":
      return {
        ...state,
        messages:
          state.messages.filter(
            m => m.id !== event.payload.id
          )
      };

    default:
      return state;
  }
}
```

Then:

```js
socket.onmessage = event => {
  const parsed = parseAndValidate(event);

  state = reduce(
    state,
    parsed
  );

  emit();
};
```

And React subscribes to that store.

This gives you:

```text
socket
 ↓
event
 ↓
reduce
 ↓
external state
 ↓
useSyncExternalStore
 ↓
React
```

That's an elegant architecture for complex real-time applications.

---

# 59. Real-time + optimistic mutation

Let's put everything together for a chat send operation.

Client:

```text
user clicks Send
 ↓
optimistic message appears
 ↓
Server Function / HTTP mutation
 ↓
server stores message
 ↓
server broadcasts canonical message
 ↓
client receives event
 ↓
replace temporary message
 ↓
final UI
```

More precisely:

```text
                  SEND
                   │
                   ▼
             optimistic UI
             temp-id = 123
                   │
                   ▼
             server mutation
                   │
                   ▼
             database write
                   │
                   ▼
             broadcast event
                   │
                   ▼
              WebSocket
                   │
                   ▼
            canonical event
            id = 987
                   │
                   ▼
             reconcile temp
                   │
                   ▼
                UI
```

This is a highly realistic modern React architecture.

---

# 60. What if the mutation fails?

Then:

```text
optimistic message
 ↓
server rejects
 ↓
remove/mark failed
```

The client needs to reconcile:

```text
expected state
vs
server reality
```

That is exactly the same optimistic-state concept from Lesson 34.

---

# 61. What if the server is faster than your mutation response?

Interesting case.

You send:

```text
message X
```

Server writes it and broadcasts:

```text
message X
```

Then your original request resolves.

You might process:

```text
broadcast
```

before:

```text
mutation response
```

So event ordering across channels can differ.

For example:

```text
WebSocket
 ↓
message-created

HTTP
 ↓
request-success
```

The two transport paths have independent timing.

Therefore your application shouldn't assume:

```text
HTTP response always arrives first
```

or:

```text
WebSocket event always arrives first
```

This is another reason domain IDs and reconciliation rules matter.

---

# 62. Real-time systems are about eventual convergence

This is perhaps the deepest idea in this lesson.

Your client may temporarily have:

```text
state A
```

Server:

```text
state B
```

Optimistic state:

```text
state C
```

Then events arrive.

The architecture's job is to make the client eventually converge toward:

```text
server truth
```

So:

```text
optimistic UI
    ↓
mutation
    ↓
server truth
    ↓
event propagation
    ↓
client convergence
```

This is much more accurate than imagining real-time UI as:

```text
server pushed data
→ setState
```

---

# 63. Performance and real-time architecture are inseparable

Remember the previous lesson.

Suppose server sends:

```text
1,000 events/sec
```

You need to decide:

```text
Which events matter?
Which should be aggregated?
Which should be dropped?
Which should be stored?
Which should trigger React?
Which should update only external state?
Which should affect visible UI?
```

This is a data-flow architecture question.

Not simply:

```text
"How do I make WebSocket work?"
```

---

# 64. A practical classification

|Data|Typical approach|
|---|---|
|Chat messages|Query/cache + WebSocket|
|Notifications|Query/cache + event stream|
|Typing indicator|Ephemeral local/external state|
|Cursor position|External mutable state + frame-based rendering|
|Live metrics|External store + coalescing|
|CRUD mutations|Action/Server Function/HTTP|
|Connection status|External store/browser state|
|Initial snapshot|HTTP/query/loader/server component|

This isn't a rigid rulebook; it is a useful starting architecture.

---

# 65. A complete production-style diagram

```text
                         SERVER
                           │
                  ┌────────┴────────┐
                  │                 │
               HTTP/API         WebSocket/SSE
                  │                 │
                  │                 │
                  ▼                 ▼
              initial data       events
                  │                 │
                  └────────┬────────┘
                           ▼
                    client data layer
                           │
                 ┌─────────┴─────────┐
                 │                   │
          durable server state   ephemeral state
                 │                   │
          TanStack Query       external store/ref
                 │                   │
                 └─────────┬─────────┘
                           ▼
                         React
                           │
                 ┌─────────┼─────────┐
                 │         │         │
               state     Suspense   UI
                 │
             transitions
```

That is a much more realistic mental model of a real-time React app.

---

# 66. The most important mistakes to avoid

### Mistake 1

Creating a WebSocket during render.

```text
render
→ new WebSocket()
```

Never do that.

---

### Mistake 2

Creating one socket per component.

```text
5 components
→ 5 connections
```

Usually centralize connection ownership.

---

### Mistake 3

Putting every event directly into React state.

```text
event
→ setState
```

For high-frequency streams, buffer/coalesce.

---

### Mistake 4

Assuming reconnect means synchronized.

```text
reconnect
≠
state recovery
```

Resynchronize deliberately.

---

### Mistake 5

Treating event delivery as exactly once.

Design for duplicates/missing events where the transport/protocol can produce them.

---

### Mistake 6

Trusting event payloads.

Network data is untrusted input.

Validate it.

---

### Mistake 7

Assuming the client controls permissions.

Authorization belongs on the server.

---

# 67. The React-specific mental model

The cleanest architecture often looks like:

```text
EXTERNAL SYSTEM
     │
     │ websocket / SSE
     ▼
connection manager
     │
     ▼
event validation
     │
     ▼
domain store
     │
     ▼
useSyncExternalStore
     │
     ▼
React
```

Then:

```text
React mutation
     │
     ▼
Server Function / API
     │
     ▼
server
     │
     ▼
broadcast event
     │
     ▼
domain store
     │
     ▼
React
```

This creates a cycle:

```text
USER
 ↓
mutation
 ↓
SERVER
 ↓
event
 ↓
STORE
 ↓
REACT
```

rather than independently updating five unrelated pieces of client state.

---

# 68. The deepest connection to Lesson 33

We learned:

```text
useSyncExternalStore
```

because some state lives outside React.

A WebSocket is exactly that kind of system.

The socket has its own lifecycle:

```text
open
message
close
error
reconnect
```

React doesn't own that lifecycle.

Therefore:

```text
WebSocket
→ external system

useSyncExternalStore
→ React's safe subscription mechanism
```

That's one of the strongest real-world use cases for the hook.

---

# 69. The deepest connection to the performance lesson

We learned:

```text
render count
≠
performance
```

Now real-time systems make this obvious.

You can have:

```text
1,000 events
```

but only:

```text
10 meaningful UI updates
```

by aggregating them.

So:

```text
event frequency
→ transport problem

render frequency
→ React problem

visual update frequency
→ browser/UI problem
```

They are separate dimensions.

---

# 70. Final mental model

Remember this:

```text
                  REAL-TIME SYSTEM

             ┌─────────────────────┐
             │   WebSocket / SSE   │
             └──────────┬──────────┘
                        │
                     events
                        │
                        ▼
                validate / decode
                        │
                        ▼
                  domain store
                        │
               ┌────────┴────────┐
               │                 │
         high-frequency       durable state
               │                 │
          buffer/coalesce      query cache
          ref/external store       │
               │                 │
               └────────┬────────┘
                        ▼
               useSyncExternalStore
                        │
                        ▼
                      React
                        │
                ┌───────┴────────┐
                │                │
              render         interaction
                                  │
                                  ▼
                         Server Function/API
                                  │
                                  ▼
                               SERVER
```

The key distinction is:

> **React should not own the real-time transport. React should consume the state produced by the real-time transport.**

That single idea leads to much cleaner architecture.

---

# What to remember

```text
WebSocket
→ bidirectional persistent connection

SSE
→ server → browser event stream

useEffect
→ establish/cleanup external connection

useRef
→ buffer high-frequency transient data without rendering

useSyncExternalStore
→ subscribe React to shared external real-time state

TanStack Query
→ durable server-state cache

WebSocket/SSE
→ incremental live updates

Snapshot + event stream
→ initial truth + subsequent changes

Reconnect
→ transport recovery

Resync
→ application-state recovery

Backoff + jitter
→ avoid reconnect storms

Deduplication
→ tolerate repeated events

Sequence IDs
→ detect missing/out-of-order events

Coalescing
→ many events → fewer UI updates

requestAnimationFrame
→ synchronize visual updates with browser frames

optimistic state
→ temporary expected result

server event
→ canonical truth

reconciliation
→ make temporary and canonical states converge
```

And the single most important principle from this lesson:

> **An event stream can be high-frequency, unreliable, duplicated, delayed, or reordered. Your React UI should be built on top of a deliberate state model rather than treating every incoming packet as "just setState."**

The next natural topic is **production React architecture**: how to structure a large application by domains/features, where routing/data/UI logic should live, how to separate server/client/shared code, how to avoid giant hooks and god-components, and how all the patterns we've learned fit into an actual project.