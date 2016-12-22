# cluster-client
Sharing Connection among Multi-Process Nodejs

[![NPM version][npm-image]][npm-url]
[![build status][travis-image]][travis-url]
[![Test coverage][codecov-image]][codecov-url]
[![David deps][david-image]][david-url]
[![Known Vulnerabilities][snyk-image]][snyk-url]
[![npm download][download-image]][download-url]

[npm-image]: https://img.shields.io/npm/v/cluster-client.svg?style=flat-square
[npm-url]: https://npmjs.org/package/cluster-client
[travis-image]: https://img.shields.io/travis/node-modules/cluster-client.svg?style=flat-square
[travis-url]: https://travis-ci.org/node-modules/cluster-client
[codecov-image]: https://codecov.io/gh/node-modules/cluster-client/branch/master/graph/badge.svg
[codecov-url]: https://codecov.io/gh/node-modules/cluster-client
[david-image]: https://img.shields.io/david/node-modules/cluster-client.svg?style=flat-square
[david-url]: https://david-dm.org/node-modules/cluster-client
[snyk-image]: https://snyk.io/test/npm/cluster-client/badge.svg?style=flat-square
[snyk-url]: https://snyk.io/test/npm/cluster-client
[download-image]: https://img.shields.io/npm/dm/cluster-client.svg?style=flat-square
[download-url]: https://npmjs.org/package/cluster-client

As we know, each Node.js process runs in a single thread. Usually, we split a single process into multiple processes to take advantage of multi-core systems. On the other hand, it brings more system overhead, sush as maintaining more TCP connections between servers.

This module is designed to share connections among multi-process Nodejs.

## Theory

- Inspired by [Leader/Follower pattern](http://www.cs.wustl.edu/~schmidt/PDF/lf.pdf).
- Allow ONLY one process "the Leader" to communicate with server. Other processes "the Followers" act as "Proxy" client, and forward all requests to Leader.
- The Leader is selected by "Port Competition". Every process try to listen on a certain port (for example 7777), but ONLY one can occupy the port, then it becomes the Leader, the others become Followers.
- TCP socket connections are maintained between Leader and Followers. And I design a simple communication protocol to exchange data between them.
- If old Leader dies, one of processes will be selected as the new Leader.

## Diagram

normal (without using cluster client)
```js
+--------+   +--------+
| Client |   | Client |   ...
+--------+   +--------+
    |  \     /   |
    |    \ /     |
    |    / \     |
    |  /     \   |
+--------+   +--------+
| Server |   | Server |   ...
+--------+   +--------+

```

using cluster-client
```js
             +-------+
             | start |
             +---+---+
                 |
        +--------+---------+
      __| port competition |__
win /   +------------------+  \ lose
   /                           \
+--------+     tcp conn     +----------+
| Leader |<---------------->| Follower |
+--------+                  +----------+
    |
+--------+
| Client |
+--------+
    |  \
    |    \
    |      \
    |        \
+--------+   +--------+
| Server |   | Server |   ...
+--------+   +--------+

```

## Protocol

- Packet structure
```js
 0       1       2               4                                                              12
 +-------+-------+---------------+---------------------------------------------------------------+
 |version|req/res|    reserved   |                          request id                           |
 +-------------------------------+-------------------------------+-------------------------------+
 |           timeout             |   connection object length    |   application object length   |
 +-------------------------------+---------------------------------------------------------------+
 |         conn object (JSON format)  ...                    |            app object             |
 +-----------------------------------------------------------+                                   |
 |                                          ...                                                  |
 +-----------------------------------------------------------------------------------------------+
```
- Protocol Type
  - Register Channel
  - Subscribe/Publish
  - Invoke
- Sequence diagram

```js
 +----------+             +---------------+          +---------+ 
 | Follower |             |  local server |          |  Leader |
 +----------+             +---------------+          +---------+ 
      |     register channel     |       assign to        |
      + -----------------------> |  --------------------> |
      |                          |                        |
      |                                subscribe          |
      + ------------------------------------------------> |
      |       subscribe result                            |
      | <------------------------------------------------ +
      |                                                   |
      |                                 invoke            |
      + ------------------------------------------------> |
      |          invoke result                            |
      | <------------------------------------------------ +
      |                                                   |
```

## Install

```bash
$ npm install cluster-client --save
```

Node.js >= 4.0.0 required

## Usage

```js
'use strict';

const co = require('co');
const Base = require('sdk-base');
const cluster = require('cluster-client');

/**
 * Client Example
 */
class YourClient extends Base {
  constructor(options) {
    super(options);

    this.options = options;
    this.ready(true);
  }

  subscribe(reg, listener) {
    // subscribe logic
  }

  publish(reg) {
    // publish logic
  }

  * getData(id) {
    // invoke api
  }

  getDataCallback(id, cb) {
    // ...
  }

  getDataPromise(id) {
    // ...
  }
}

// create some client instances, but only one instance will connect to server
const client_1 = cluster(YourClient)
  .delegate('getData')
  .delegate('getDataCallback')
  .delegate('getDataPromise')
  .create({ foo: 'bar' });
const client_2 = cluster(YourClient)
  .delegate('getData')
  .delegate('getDataCallback')
  .delegate('getDataPromise')
  .create({ foo: 'bar' });
const client_3 = cluster(YourClient)
  .delegate('getData')
  .delegate('getDataCallback')
  .delegate('getDataPromise')
  .create({ foo: 'bar' });;

// subscribe information
client_1.subsribe('some thing', result => console.log(result));
client_2.subsribe('some thing', result => console.log(result));
client_3.subsribe('some thing', result => console.log(result));

// publish data
client_2.publish('some data');

// invoke method
client_3.getDataCallback('some thing', (err, val) => console.log(val));
client_2.getPromise('some thing').then(val => console.log(val));

co(function*() {
  const ret = yield client_1.getData('some thing');
  console.log(ret);
}).catch(err => console.error(err));
```

## API

- `delegate(from, to)`:
  create delegate method, `from` is the method name your want to create, and `to` have 3 possible values: subscribe, publish, and invoke, the default value is invoke 
- `override(name, value)`:
  override one property
- `create(…)`
  create the client instance


[MIT](LICENSE)