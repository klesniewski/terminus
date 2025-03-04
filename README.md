# terminus

[![Join Slack](https://img.shields.io/badge/Join%20us%20on-Slack-e01563.svg)](https://godaddy-oss.slack.com/)
[![Build Status](https://github.com/godaddy/terminus/actions/workflows/cicd.yml/badge.svg)](https://github.com/godaddy/terminus/actions/workflows/cicd.yml/badge.svg)

Adds graceful shutdown and Kubernetes readiness / liveness checks for any HTTP applications.

## Installation

Install via npm:

```console
npm i @godaddy/terminus --save
```

## Usage

```javascript
const http = require('http');
const { createTerminus } = require('@godaddy/terminus');

function onSignal () {
  console.log('server is starting cleanup');
  return Promise.all([
    // your clean logic, like closing database connections
  ]);
}

function onShutdown () {
  console.log('cleanup finished, server is shutting down');
}

function healthCheck ({ state }) {
  // `state.isShuttingDown` (boolean) shows whether the server is shutting down or not
  return Promise.resolve(
    // optionally include a resolve value to be included as
    // info in the health check response
  )
}

const server = http.createServer((request, response) => {
  response.end(
    `<html>
      <body>
        <h1>Hello, World!</h1>
       </body>
     </html>`
   );
})

const options = {
  // health check options
  healthChecks: {
    '/healthcheck': healthCheck,    // a function accepting a state and returning a promise indicating service health,
    verbatim: true,                 // [optional = false] use object returned from /healthcheck verbatim in response,
    __unsafeExposeStackTraces: true // [optional = false] return stack traces in error response if healthchecks throw errors
  },
  caseInsensitive,                  // [optional] whether given health checks routes are case insensitive (defaults to false)

  statusOk,                         // [optional = 200] status to be returned for successful healthchecks
  statusOkResponse,                 // [optional = { status: 'ok' }] status response to be returned for successful healthchecks
  statusError,                      // [optional = 503] status to be returned for unsuccessful healthchecks
  statusErrorResponse,              // [optional = { status: 'error' }] status response to be returned for unsuccessful healthchecks

  // cleanup options
  timeout: 1000,                    // [optional = 1000] number of milliseconds before forceful exiting
  signal,                           // [optional = 'SIGTERM'] what signal to listen for relative to shutdown
  signals,                          // [optional = []] array of signals to listen for relative to shutdown
  useExit0,                         // [optional = false] instead of sending the received signal again without beeing catched, the process will exit(0)
  sendFailuresDuringShutdown,       // [optional = true] whether or not to send failure (503) during shutdown
  beforeShutdown,                   // [optional] called before the HTTP server starts its shutdown
  onSignal,                         // [optional] cleanup function, returning a promise (used to be onSigterm)
  onShutdown,                       // [optional] called right before exiting
  onSendFailureDuringShutdown,      // [optional] called before sending each 503 during shutdowns

  // both
  logger                            // [optional] logger function to be called with errors. Example logger call: ('error happened during shutdown', error). See terminus.js for more details.
};

createTerminus(server, options);

server.listen(PORT || 3000);
```

Upon receiving the signal, Terminus will:

1. Set the state to shutting down.
   You can use it to start failing readiness probe.
2. Execute and wait for `beforeShutdown`.
   Here you can log the fact that shutdown has started.
   You can also delay here shutting down HTTP server, so that the service is taken out from load balancer first.
3. Stop HTTP server and wait for its complete shutdown.
   The server will immediately stop accepting new connections.
   All existing idle connections get terminated with sending a `FIN` packet.
   Terminus will wait for `timeout` for in-flight requests to complete, after which the remaining requests will be shut down with a `FIN` packet.
4. Execute and wait for `onSignal`.
   By now there are no more in-flight requests and you can clean up your resources.
   For example, shut down your database connections.
5. Execute and wait for `onShutdown`.
   Here you can log successful shutdown and flush your logs.
6. Exit with code `0` or kill the process, depending on `useExit0` setting.

If any of the steps above throws, Terminus will log it and exit the process with code `1`.

With `healthChecks` you can register multiple health checks, so you can have a separate liveness and readiness probe implementations.

Note, that when `sendFailuredDuringShutdown` is set to `true` (default) and Terminus enters shutting down state, it will return failure without calling provided health check implementation.

### With custom error messages

```js
const http = require('http');
const { createTerminus, HealthCheckError } = require('@godaddy/terminus');

createTerminus(server, {
  healthChecks: {
    '/healthcheck': async function () {
      const errors = []
      return Promise.all([
        // all your health checks goes here
      ].map(p => p.catch((error) => {
        // silently collecting all the errors
        errors.push(error)
        return undefined
      }))).then(() => {
        if (errors.length) {
          throw new HealthCheckError('healthcheck failed', errors)
        }
      })
    }
  }
});
```

### With custom headers
```js
const http = require("http");
const express = require("express");
const { createTerminus, HealthCheckError } = require('@godaddy/terminus');
const app = express();

app.get("/", (req, res) => {
  res.send("ok");
});

const server = http.createServer(app);

function healthCheck({ state }) {
  return Promise.resolve();
}

const options = {
  healthChecks: {
    "/healthcheck": healthCheck,
    verbatim: true,
    __unsafeExposeStackTraces: true,
  },
  headers: {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "OPTIONS, POST, GET",
  },
};

terminus.createTerminus(server, options);

server.listen(3000);
```

### With express

```javascript
const http = require('http');
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('ok');
});

const server = http.createServer(app);

const options = {
  // opts
};

createTerminus(server, options);

server.listen(PORT || 3000);
```

### With koa

```javascript
const http = require('http');
const Koa = require('koa');
const app = new Koa();

const server = http.createServer(app.callback());

const options = {
  // opts
};

createTerminus(server, options);

server.listen(PORT || 3000);
```

### With cluster (and eg. express)

If you want to use (`cluster`)[https://nodejs.org/api/cluster.html] to use more than one CPU, you need to use `terminus` per worker.
This is heavily inspired by https://medium.com/@gaurav.lahoti/graceful-shutdown-of-node-js-workers-dd58bbff9e30.

See `example/express.cluster.js`.

## How to set Terminus up with Kubernetes?

When Kubernetes or a user deletes a Pod, Kubernetes will notify it and wait for `gracePeriod` seconds before killing it.

During that time window (30 seconds by default), the Pod is in the `terminating` state and will be removed from any Services by a controller.
The Pod itself needs to catch the `SIGTERM` signal and start failing any readiness probes.

> If the ingress controller you use route via the Service, it is not an issue for your case. At the time of this writing, we use the nginx ingress controller which routes traffic directly to the Pods.

During this time, it is possible that load-balancers (like the nginx ingress controller) don't remove the Pods "in time", and when the Pod dies, it kills live connections.

To make sure you don't lose any connections, we recommend delaying the shutdown with the number of milliseconds that's defined by the readiness probe in your deployment configuration.
To help with this, terminus exposes an option called `beforeShutdown` that takes any Promise-returning function.

Also it makes sense to use the `useExit0 = true` option to signal Kubernetes that the container exited gracefully.
Otherwise APM's will send you alerts, in some cases.

```javascript
function beforeShutdown () {
  // given your readiness probes run every 5 second
  // may be worth using a bigger number so you won't
  // run into any race conditions
  return new Promise(resolve => {
    setTimeout(resolve, 5000)
  })
}
createTerminus(server, {
  beforeShutdown,
  useExit0: true
})
```

[Learn more](https://github.com/kubernetes/contrib/issues/1140#issuecomment-231641402)

## Limited Windows support

Due to inherent platform limitations, `terminus` has limited support for Windows.
You can expect `SIGINT` to work, as well as `SIGBREAK` and to some extent `SIGHUP`.
However `SIGTERM` will never work on Windows because killing a process in the task manager is unconditional, i.e., there's no way for an application to detect or prevent it.
Here's some relevant documentation from [`libuv`](https://github.com/libuv/libuv) to learn more about what `SIGINT`, `SIGBREAK` etc. signify and what's supported on Windows - [http://docs.libuv.org/en/v1.x/signal.html]([http://docs.libuv.org/en/v1.x/signal.html).
Also, see [https://nodejs.org/api/process.html#process_signal_events](https://nodejs.org/api/process.html#process_signal_events).
