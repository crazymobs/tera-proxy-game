# tera-proxy-game

Hosts a TCP proxy server to read, modify, and inject network data between a TERA game client and server. This modular system built on event-based hooks allows for easy creation and usage of script mods.

This document is divided into two section. Most readers are likely viewing this for documentation on writing modules, so the [module reference](#module-reference) will come first. The latter half, the [API reference](#api-reference), details the classes exported by `require('tera-proxy-game')`, which is not needed for module authors.

# Module Reference

A module loaded through `dispatch.load(name)` is instantiated similarly to:

```js
const YourModule = require(name);
modules[name] = new YourModule(dispatch);
```

Thus, a loadable module's export must be a constructible function with at least one parameter, which will be an instance of `DispatchWrapper`. Since this function is called with `new`, the context of `this` will be unique to each connection to the proxy allowing support for multiple clients.

For examples, see [TODO](#...).

## `DispatchWrapper`

An instance of `DispatchWrapper` is created for each module loaded by `Dispatch`. It keeps track of a module's name and has a handful of methods which mostly forward parameters to the base `Dispatch` instance. It is often named `dispatch` for conciseness and legacy reasons, which might be confusing, but in most cases as a module author you won't need to interact with the base `Dispatch` object.

### Properties

#### `base`

Points to the base `Dispatch` instance. Useful if you want to track something per-connection given an instance of a `DispatchWrapper`, where you can make a `WeakMap` with `dispatch.base` as a key (see [`slash`](https://github.com/baldera-mods/slash/blob/master/index.js) for an example).

### Methods

#### `hook(name, version, [options], callback)`

Adds a hook for a packet.

`name` will usually be the name of the message being watched for, such as `"S_LOGIN"`, but it can also be `"*"` to catch all messages. You may also use lowerCamelCase for names, removing all underscores and using lowercase unless the character was originally preceded by an underscore (such as `"sLogin"`).

`version` should be an integer representing the desired packet version as given by the corresponding [`tera-data`] definition file, or it can be `"*"` to use the latest version. Alternatively, `"raw"` can be used to create a raw hook, which does not perform any parsing and instead passes the callback the raw (decrypted) data buffer.

`options` is an optional object with the following optional properties:

 * `order`: A lower number causes this hook to run before hooks with a higher `order`. This defaults to 0, so you can imagine negative values running closer to the source side and positive values running closer to the destination side. This is helpful for modules that way want to examine packets before another module modifies or silences them, or filter to what the receiving side will see by running later than most other hooks.

 * `type`: One of `"real"` (default), `"fake"`, or `"all"`. Packets created and sent through `Dispatch` are first routed back through the handler, so you can, for instance, examine only "fake" (generated) packets by making a hook with `type: 'fake'`. Just be extra sure that any packets generated in a `fake` or `all` hook do not generate more in an infinite loop.

`callback` is the callback function for the hook, which receives:
 * For a normal hook (version is not `"raw"`),
   * `event`: An object containing the parsed message data (see [`tera-data`]).
   * `fake`: `true` if this packet was generated through the proxy, `false` otherwise.
   * Return value is `true` if `event` is modified, or `false` to stop and silence the message. Other return values are ignored. **If you change `event` but do not return `true`, the changes will not be saved.**
 * For a `raw` hook,
   * `code`: The opcode of the message as an integer.
   * `data`: The `Buffer` of the raw message data.
   * `fromServer`: `true` if the message was sent by the server, `false` otherwise.
   * `fake`: `true` if this packet was generated through the proxy, `false` otherwise.
   * Return value is a `Buffer` of the modified message data to use, or `false` to stop and silence the message. Other return values are ignored. Note that unlike for normal hooks, modifications to the original buffer will be saved and used *regardless of whether you return `true`*.

Returns an object representing the hook. Its properties are not set in stone and are not meant to be changed, so do not depend on them. If you have a use case where you absolutely need to do something with the properties, please submit a GitHub issue describing it.

 [`tera-data`]: <https://github.com/meishuu/tera-data>

#### `unhook(hook)`

Removes a hook. It will no longer be called. Pass in the result of a call to `Dispatch#hook()`.

#### `toClient(buffer)`
#### `toClient(name, version, data)`
#### `toServer(buffer)`
#### `toServer(name, version, data)`

Constructs and sends a packet to either the TERA client or server.

If `buffer` is used, it will simply be sent as-is (before encryption).

If `data` is used, it must be an object (to be serialized by [`tera-data-parser`](https://github.com/meishuu/tera-data-parser-js)). `name` must be the packet name and `version` should be the version number as given by the `tera-data` definition file, or `"*"` for the latest version.

#### `load(name, [from, [required, [args...]]])`

Load the module referenced by `name` using `from.require()`. You will likely want to pass the `module` from the calling context in order to emulate a `require()` from there; otherwise, it will default to loading the module as if `require()` were called from inside `tera-proxy-game`. See the [module.require documentation](https://nodejs.org/api/modules.html#modules_module_require_id) for more details.

Example usage:

```js
dispatch.load('logger', module);
```

If the second parameter is a function instead of a reference to `module`, it will use that function as the module constructor (thus, it *cannot* be an arrow function). If this is done, it is recommended to pass a `name` that won't conflict with any loadable modules.

```js
dispatch.load('<core>', function coreModule(dispatch) {
  // ...
});
```

If `required` (default `true`) is falsy, any errors thrown from attempting to load the module will be caught and printed, and no value will be returned.

Any subsequent arguments, if given, will be passed to the module's constructor after the one and only `dispatch` parameter.

Returns the instantiated module if successful.

#### `unload(name)`

Unloads the module referenced by `name`, calling the `destructor()` method on the module if it exists. This will also automatically remove all hooks associated with the module.

Returns `true` if successful, `false` otherwise.

## `Dispatch`

An instance of `Dispatch` is created for every connection to the proxy game server. You'll typically only need to access this if you are writing for the proxy, not for a module.

### Properties

#### `connection`

### Methods

#### `load(name, [from, [args...]])`

See `DispatchWrapper#load()`. Note that this version does not have the `required` parameter.

#### `unload(name)`
#### `hook(name, version, [options], callback)`
#### `unhook(hook)`

Same as the methods of the same name from `DispatchWrapper`. You generally shouldn't use this `hook()` since it will be missing module information and requires an explicit `unhook()`.

#### `write(outgoing, buffer)`
#### `write(outgoing, name, version, data)`

Sends data to one of the associated connections.

`outgoing` should be `true` to send to the server, otherwise it will be sent to the client.

For the rest of the parameters, see `DispatchWrapper#toClient()`.

#### `handle(data, fromServer, [fake])`

Passes the (usually parsed) data all registered hooks.

Return value is the raw data buffer, or `false` if silenced by any hook.

#### `reset()`

Unloads all modules and removes all hooks.

# API Reference

**Note: If you are a module author, you most likely do not need anything from this section and may stop reading.**

`tera-proxy-game` exposes three classes, each of which can be accessed as a property of the same name on the exported object. These allow for setting up the proxy game server and hooking up a client to it.

## Example
```js
const net = require('net');
const { Connection, RealClient } = require('tera-proxy-game');

// Create a regular TCP server.
const server = net.createServer((socket) => {
  // Disable packet buffering to minimize latency.
  socket.setNoDelay(true);

  // Set up a connection handler.
  const connection = new Connection();

  // Set up a client handler to handle communication between the client and proxy.
  const client = new RealClient(connection, socket);

  // Connect this proxy to some TERA server.
  const proxied = connection.connect(client, { host: '208.67.49.92', port: 10001 });

  // Load the `logger` module.
  connection.dispatch.load('logger');
});

// Listen on localhost:9247.
server.listen(9247, '127.0.0.1', () => {
  const { address, port } = server.address();
  console.log(`listening on ${address}:${port}`);
});
```

## `Connection`

This class handles encryption and packet buffering for a socket to a real game server.

### `new Connection()`

Sets up a connection handler. This also initializes an instance of `Dispatch` unique to this `Connection`.

### Properties

#### `dispatch`

An intsance of `Dispatch`.

### Methods

#### `connect(client, options)`

Start a connection between the real game server and `client`, either a `RealClient` or a `FakeClient`.

`options` will be passed directly to [`net.connect`](https://nodejs.org/api/net.html#net_net_connect_options_connectlistener).

Returns the `net.Socket` to the real game server.

#### `close()`

Ends the connection and calls `Dispatch#reset()`.

## `RealClient`

A `RealClient` is for legitimate game clients or otherwise clients driven by outside means. It has its own packet buffer and encryption state to appear transparent to the client while still allowing `Dispatch` to read, modify, and inject packets.

### `new RealClient(connection, socket)`

`connection` must be an instance of `Connection`.

`socket` must be a TCP socket (see [`net.Socket`](https://nodejs.org/api/net.html#net_class_net_socket)). You most likely want to call `socket.setNoDelay(true)` to disable buffering, or else the client connection will have inconsistent latency.

## `FakeClient`

A `FakeClient` is used for headless clients / bots. It does not require an incoming TCP connection and it is expected that you will simulate an actual client through `Dispatch` (see [`tera-discord-relay`](https://github.com/meishuu/tera-discord-relay/blob/master/tera/index.js)).

### `new FakeClient(connection, [keys])`

`connection` must be an instance of `Connection`.

`keys`, if provided, must be an array of two `Buffer`s, each of length 128. If not provided, they will be generated [randomly](https://nodejs.org/api/crypto.html#crypto_crypto_randombytes_size_callback). These keys are used to set up encryption for the connection.

### Events

A `FakeClient` inherits from [`EventEmitter`](https://nodejs.org/api/events.html#events_class_eventemitter) and emits the following events:

#### `connect`

Emitted *after* the initial handshake, as soon as both the client and server have shared their encryption keys.

#### `timeout`
#### `error`

These two events are forwarded from the `net.Socket` to the real game server.

#### `close`

Emitted when `FakeClient#close()` is called.
