# lil-http-terminator 🦾

Gracefully terminates HTTP(S) server.

This module was forked from the amazing [http-terminator](https://github.com/gajus/http-terminator). The important changes:

- Zero dependencies. The original `http-terminator` brings in more than 10 sub-dependencies.
- Removed TypeScript and a dozen of supporting files, configurations, etc. No more code transpilation.
- Simpler API. Now you do `require("lil-http-terminator")({ server });` to get a terminator object.

## Behaviour

When you call [`server.close()`](https://nodejs.org/api/http.html#http_server_close_callback), it stops the server from accepting new connections, but it keeps the existing connections open indefinitely. This can result in your server hanging indefinitely due to keep-alive connections or because of the ongoing requests that do not produce a response. Therefore, in order to close the server, you must track creation of all connections and terminate them yourself.

lil-http-terminator implements the logic for tracking all connections and their termination upon a timeout. lil-http-terminator also ensures graceful communication of the server intention to shutdown to any clients that are currently receiving response from this server.

## API

```js
const HttpTerminator = require("lil-http-terminator");

const terminator = HttpTerminator({
    server, // required. The node.js http server object instance
    
    // optional
    gracefulTerminationTimeout: 1000, // optional, default is 1000
    logger: console, // optional, default is `global.console`. If termination goes wild the module might log about it.
}) 

// Do not call server.close(); Instead call this:
await terminator.terminate();
```

## Usage

Use the terminator when node.js process is shutting down.

```js
const http = require("http");

const server = http.createServer();

const httpTerminator = require("lil-http-terminator")({
  server
});

async function shutdown(signal) {
    console.log(`Received ${signal}. Shutting down.`)
    await httpTerminator.terminate();
    process.exit(0);
}

process.on("SIGTERM", shutdown); // used by K8s, AWS ECS, etc.
process.on("SIGINT", shutdown); // Atom, VSCode, WebStorm or Terminal Ctrl+C
```

## Alternative libraries

There are several alternative libraries that implement comparable functionality, e.g.

- https://github.com/gajus/http-terminator (origin of this module)
- https://github.com/hunterloftis/stoppable
- https://github.com/thedillonb/http-shutdown
- https://github.com/tellnes/http-close
- https://github.com/sebhildebrandt/http-graceful-shutdown

The main benefit of `lil-http-terminator` is that:

- it does not have any dependencies
- it does not monkey-patch Node.js API
- it immediately destroys all sockets without an attached HTTP request
- it allows graceful timeout to sockets with ongoing HTTP requests
- it properly handles HTTPS connections
- it informs connections using keep-alive that server is shutting down by setting a `connection: close` header
- it does not terminate the Node.js process

## FAQ

### What is the use case for lil-http-terminator?

To gracefully terminate a HTTP server.

We say that a service is gracefully terminated when service stops accepting new clients, but allows time to complete the existing requests.

There are several reasons to terminate services gracefully:

- Terminating a service gracefully ensures that the client experience is not affected (assuming the service is load-balanced).
- If your application is stateful, then when services are not terminated gracefully, you are risking data corruption.
- Forcing termination of the service with a timeout ensures timely termination of the service (otherwise the service can remain hanging indefinitely).
