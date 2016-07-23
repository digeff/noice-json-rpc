# Noice Json Rpc
[![Build Status](https://img.shields.io/travis/nojvek/noice-json-rpc/master.svg)](https://travis-ci.org/nojvek/noice-json-rpc)
[![Coverage Status](https://img.shields.io/coveralls/nojvek/noice-json-rpc/master.svg)](https://coveralls.io/github/nojvek/noice-json-rpc?branch=master)
[![npm version](https://img.shields.io/npm/v/noice-json-rpc.svg)](https://www.npmjs.com/package/noice-json-rpc)
[![npm total downloads](https://img.shields.io/npm/dt/noice-json-rpc.svg?maxAge=2592000)](https://www.npmjs.com/package/noice-json-rpc)
[![licence](https://img.shields.io/npm/l/express.svg?maxAge=2592000)](https://github.com/nojvek/noice-json-rpc)
[![issues open](https://img.shields.io/github/issues/nojvek/noice-json-rpc.svg)](https://github.com/nojvek/noice-json-rpc/issues)

Client and Server helpers to implement a clean function based Api for Json Rpc.

Noice Json Rpc takes in a websocket like object. It calls `send(msg:str)` function and expects messages to come from `on('message', handler)`. It works out of the box with WebSockets but it can also work with stdin/stdout, worker threads, iframes or any other mechanism in which strings can be sent or received. 

Its only dependency is `events.EventEmitter`.

Using ES6 proxies it exposes a clean client-server api. Since its written in TypeScript, the api object can be cast to work off an interface specific to the domain. e.g [chrome-remote-debug-protocol](https://github.com/nojvek/chrome-remote-debug-protocol) 

## [Example](tests/example.ts)

```js
import * as WebSocket from 'ws'
import WebSocketServer = WebSocket.Server
import * as rpc from 'noice-json-rpc'

async function setupClient() {
    try {
        const api = new rpc.Client(new WebSocket("ws://localhost:8080"), {logConsole: true}).api()

        await Promise.all([
            api.Runtime.enable(),
            api.Debugger.enable(),
            api.Profiler.enable(),
            api.Runtime.run(),
        ])

        await api.Profiler.start()
        await new Promise((resolve) => api.Runtime.onExecutionContextDestroyed(resolve)); // Wait for event
        await api.Profiler.stop()

    } catch (e) {
        console.error(e)
    }
}

function setupServer() {
    const wssServer = new WebSocketServer({port: 8080});
    const api = new rpc.Server(wssServer).api();

    const enable = () => {}

    api.Debugger.expose({enable})
    api.Profiler.expose({enable})
    api.Runtime.expose({
        enable,
        run() {}
    })
    api.Profiler.expose({
        enable,
        start() {
            setTimeout(() => {
                api.Runtime.emitExecutionContextDestroyed()
            }, 1000)
        },
        stop() {
            return {data: "noice!"}
        }
    })

}

setupServer()
setupClient()
```

Output

```
Client > {"id":1,"method":"Runtime.enable"}
Client > {"id":2,"method":"Debugger.enable"}
Client > {"id":3,"method":"Profiler.enable"}
Client > {"id":4,"method":"Runtime.run"}
Client < {"id":1,"result":{}}
Client < {"id":2,"result":{}}
Client < {"id":3,"result":{}}
Client < {"id":4,"result":{}}
Client > {"id":5,"method":"Profiler.start"}
Client < {"id":5,"result":{}}
Client < {"method":"Runtime.executionContextDestroyed"}
Client > {"id":6,"method":"Profiler.stop"}
Client < {"id":6,"result":{"data":"noice!"}}
```
