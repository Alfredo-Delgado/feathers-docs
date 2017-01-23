# Feathers Client + Primus

Primus works very similar to [Socket.io](socket-io.md) but supports a number of different real-time libraries. [Once configured on the server](../real-time/primus.md) service methods and events will be available through a Primus socket connection. 

## Establishing the connection

In the browser, the connection can be established by loading the client from `primus/primus.js` and instantiating a new `Primus` instance. Unlike HTTP calls, websockets do not have a cross-origin restriction in the browser so it is possible to connect to any Feathers server. See below for platform specific examples.

> **ProTip**: The socket connection URL has to point to the server root which is where Feathers will set up Primus.

## Client options

The Primus configuration (`primus(connection [, options])`) can take settings which currently support:

- `timeout` (default: 5000ms) - The time after which a method call fails and times out. This usually happens when calling a service or service method that does not exist.

## Browser Usage

Using [the Feathers client](feathers.md), the `feathers-primus/client` module can be configured to use the Primus connection:

```html
<script type="text/javascript" src="primus/primus.js"></script>
<script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/core-js/2.1.4/core.min.js"></script>
<script type="text/javascript" src="//unpkg.com/feathers-client@^1.0.0/dist/feathers.js"></script>
<script type="text/javascript">
  var primus = new Primus('http://api.my-feathers-server.com');
  var app = feathers()
    .configure(feathers.hooks())
    .configure(feathers.primus(primus));

  var messageService = app.service('messages');
  
  messageService.on('created', function(message) {
    console.log('Someone created a message', message);
  });
  
  messageService.create({
    text: 'Message from client'
  });
</script>
```

## Server Usage

This sets up Primus as a stand alone client in NodeJS using the websocket transport. If you are integrating with a separate server instance you can also connect without needing to require `primus-emitter`. You can read more in the [Primus documentation](https://github.com/primus/primus#connecting-from-the-server).

```bash
$ npm install feathers feathers-primus feathers-hooks primus primus-emitter ws
```

```js
const feathers = require('feathers');
const primus = require('feathers-primus/client');
const Primus = require('primus');
const Emitter = require('primus-emitter');
const hooks = require('feathers-hooks');
const Socket = Primus.createSocket({
  transformer: 'websockets',
  plugin: {
    'emitter': Emitter
  }
});
const socket = new Socket('http://api.feathersjs.com');
const app = feathers()
  .configure(hooks())
  // Configure and change the default timeout to one second
  .configure(primus(socket, { timeout: 1000 }));

// Get the message service that uses a websocket connection
const messageService = app.service('messages');

messageService.on('created', message => console.log('Someone created a message', message));
```

## React Native Usage

TODO (EK): Add some of the specific React Native things we needed to change to properly support websockets. I'm pretty sure this doesn't completely work so it's still a WIP. PR's welcome if you want to try primus out on React Native!

```bash
$ npm install feathers feathers-primus feathers-hooks primus
```

```js
import React from 'react-native';
import hooks from 'feathers-hooks';
import {client as feathers} from 'feathers';
import {client as primus} from 'feathers-primus';

let Socket = primus.socket;

const socket = new Socket('http://api.feathersjs.com');
const app = feathers()
  .configure(hooks())
  .configure(primus(socket));

// Get the message service that uses a websocket connection
const messageService = app.service('messages');

messageService.on('created', message => console.log('Someone created a message', message));
```
