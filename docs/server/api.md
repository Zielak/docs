# Web-Socket Server

## Server

The [`Server`](#server) is responsible for providing the WebSocket server to enable communication between server and client.

### `constructor (options)`

#### `options.server`

The HTTP server to bind the WebSocket Server into. You may use [`express`](https://www.npmjs.com/package/express) for your server too.

```typescript fct_label="TypeScript"
// Colyseus + Express
import { Server } from "colyseus";
import { createServer } from "http";
import express from "express";
const port = process.env.port || 3000;

const app = express();
app.use(express.json());

const gameServer = new Server({
  server: createServer(app),
  express: app,
});

gameServer.listen(port);
```

```typescript fct_label="JavaScript"
// Colyseus + Express
const colyseus = require("colyseus");
const http = require("http");
const express = require("express");
const port = process.env.port || 3000;

const app = express();
app.use(express.json());

const gameServer = new colyseus.Server({
  server: http.createServer(app)
  express: app,
});

gameServer.listen(port);
```

```typescript fct_label="TypeScript (barebones)"
// Colyseus (barebones)
import { Server } from "colyseus";
const port = process.env.port || 3000;

const gameServer = new Server();
gameServer.listen(port);
```

```typescript fct_label="JavaScript (barebones)"
// Colyseus (barebones)
const colyseus = require("colyseus");
const port = process.env.port || 3000;

const gameServer = new colyseus.Server();
gameServer.listen(port);
```

#### `options.verifyClient`

This method happens before WebSocket handshake. If `verifyClient` is not set
then the handshake is automatically accepted.

- `info` (Object)
    - `origin` (String) The value in the Origin header indicated by the client.
    - `req` (http.IncomingMessage) The client HTTP GET request.
    - `secure` (Boolean) `true` if `req.connection.authorized` or `req.connection.encrypted` is set.

- `next` (Function) A callback that must be called by the user upon inspection of the `info` fields. Arguments in this callback are:
    - `result` (Boolean) Whether or not to accept the handshake.
    - `code` (Number) When `result` is `false` this field determines the HTTP error status code to be sent to the client.
    - `name` (String) When `result` is `false` this field determines the HTTP reason phrase.

```typescript fct_label="TypeScript"
import { Server } from "colyseus";

const gameServer = new Server({
  // ...

  verifyClient: function (info, next) {
    // validate 'info'
    //
    // - next(false) will reject the websocket handshake
    // - next(true) will accept the websocket handshake
  }
});
```

```typescript fct_label="JavaScript"
const colyseus = require("colyseus");

const gameServer = new colyseus.Server({
  // ...

  verifyClient: function (info, next) {
    // validate 'info'
    //
    // - next(false) will reject the websocket handshake
    // - next(true) will accept the websocket handshake
  }
});
```

#### `options.presence`

When scaling Colyseus through multiple processes / machines, you need to provide a presence server. Read more about [scalability](/scalability/), and the [`Presence API`](/server/presence/#api).

```typescript fct_label="TypeScript"
import { Server, RedisPresence } from "colyseus";

const gameServer = new Server({
  // ...
  presence: new RedisPresence()
});
```

```typescript fct_label="JavaScript"
const colyseus = require("colyseus");

const gameServer = new colyseus.Server({
  // ...
  presence: new colyseus.RedisPresence()
});
```

The currently available Presence servers are currently:

- `RedisPresence` (scales on a single server and multiple servers)

#### `options.gracefullyShutdown`

Register shutdown routine automatically. Default is `true`. If disabled, you
should call [`gracefullyShutdown()`](#gracefullyshutdown-exit-boolean) method
manually in your shutdown process.

### `define (name: string, handler: Room, options?: any)`

Define a new room handler.

**Parameters:**

- `name: string` - The public name of the room. You'll use this name when joining the room from the client-side.
- `handler: Room` - Reference to the `Room` handler class.
- `options?: any` - Custom options for room initialization.

```typescript
// Define "chat" room
gameServer.define("chat", ChatRoom);

// Define "battle" room
gameServer.define("battle", BattleRoom);

// Define "battle" room with custom options
gameServer.define("battle_woods", BattleRoom, { map: "woods" });
```

!!! Tip "Defining the same room handler multiple times"
    You may define the same room handler multiple times with different `options`. When [Room#onCreate()](/server/room/#oncreate-options) is called, the `options` will contain the merged values you specified on [Server#define()](/server/api/#define-name-string-handler-room-options-any) + the options provided when the room is created.

---

#### Matchmaking filters: `filterBy(options)`

**Parameters**

- `options: string[]` - a list of option names

Whenever a room is created by the `create()` or `findOrCreate()` methods, only the `options` defined by the `filterBy()` method are going to be stored internally, and used to filter out rooms in further `find()` or `findOrCreate()` calls.

**Example:** allowing different "game modes".

```typescript
gameServer.define("battle", BattleRoom)
  .filterBy(['mode']);
```

Whenever the room is created, the `mode` option is going to be stored internally.

```typescript
client.findOrCreate("battle", { mode: "duo" }).then(room => {/* ... */});
```

You can handle the provided option in the `onCreate()` and/or `onJoin()` to implement the requested feature inside your room implementation.

```typescript
class BattleRoom extends Room {
  onCreate(options) {
    if (options.mode === "duo") {
      // do something!
    }
  }
  onJoin(client, options) {
    if (options.mode === "duo") {
      // put this player into a team!
    }
  }
}
```

**Example:** filtering by built-in `maxClients`

The `maxClients` is an internal variable stored for matchmaking, and can be used for filtering too.

```typescript
gameServer.define("battle", BattleRoom)
  .filterBy(['maxClients']);
```

The client can then ask to join a room capable of handling a certain number of players.

```typescript
client.findOrCreate("battle", { maxClients: 10 }).then(room => {/* ... */});
client.findOrCreate("battle", { maxClients: 20 }).then(room => {/* ... */});
```

---

#### Matchmaking priority: `sortBy(options)`

You can also give a different priority for joining rooms depending on their information upon creation.

The `options` parameter is a key-value object containing the field name in the left, and the sorting direction in the right. Sorting direction can be one of these values: `-1`, `"desc"`, `"descending"`, `1`, `"asc"` or `"ascending"`.

**Example:** sorting by the built-in `clients`

The `clients` is an internal variable stored for matchmaking, which contains the current number of connected clients. On the example below, the rooms with the highest amount of clients connected will have priority. Use `-1`, `"desc"` or `"descending"` for descending order:

```typescript
gameServer.define("battle", BattleRoom)
  .sortBy({ clients: -1 });
```

To sort by the fewest amount of players, you can do the opposite. Use `1`, `"asc"` or `"ascending"` for ascending order:

```typescript
gameServer.define("battle", BattleRoom)
  .sortBy({ clients: 1 });
```

---

#### Listening to room instance events

The `define` method will return the registered handler instance, which you can listen to match-making events from outside the room instance scope. Such as:

- `"create"` - when a room has been created
- `"dispose"` - when a room has been disposed
- `"join"` - when a client join a room
- `"leave"` - when a client leave a room
- `"lock"` - when a room has been locked
- `"unlock"` - when a room has been unlocked

**Usage:**

```typescript
gameServer.define("chat", ChatRoom)
  .on("create", (room) => console.log("room created:", room.roomId))
  .on("dispose", (room) => console.log("room disposed:", room.roomId))
  .on("join", (room, client) => console.log(client.id, "joined", room.roomId))
  .on("leave", (room, client) => console.log(client.id, "left", room.roomId));
```

!!! Warning
    It's completely discouraged to manipulate a room's state through these events. Use the [abstract methods](/server/room/#abstract-methods) in your room handler instead.

### `attach (options: any)`

> You usually do not need to call this. Use it only if you have a very specific reason to do so.

Attaches or creates the WebSocket server.

- `options.server`: The HTTP server to attach the WebSocket server on.
- `options.ws`: An existing WebSocket server to be re-used.

```javascript fct_label="Express"
import express from "express";
import { Server } from "colyseus";

const app = new express();
const gameServer = new Server();

gameServer.attach({ server: app });
```

```javascript fct_label="http.createServer"
import http from "http";
import { Server } from "colyseus";

const httpServer = http.createServer();
const gameServer = new Server();

gameServer.attach({ server: httpServer });
```

```javascript fct_label="WebSocket.Server"
import http from "http";
import express from "express";
import ws from "ws";
import { Server } from "colyseus";

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({
    // your custom WebSocket.Server setup.
});

const gameServer = new Server();
gameServer.attach({ ws: wss });
```


### `listen (port: number)`

Binds the WebSocket server into the specified port.

### `onShutdown (callback: Function)`

Register a callback that should be called before the process shut down. See [graceful shutdown](/server/graceful-shutdown/) for more details.

### `gracefullyShutdown (exit: boolean)`

Shutdown all rooms and clean-up its cached data. Returns a promise that fulfils
whenever the clean-up has been complete.

This method is called automatically unless `gracefullyShutdown: false` has been provided on `Server` constructor.
