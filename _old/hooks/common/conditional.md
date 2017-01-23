# Running hooks conditionally

There are times when you may want to run a hook conditionally,
perhaps depending on the provider, the user authorization,
if the user created the record, etc.

A custom service may be designed to always be called with the `create` method,
with a data value specifying the action the service is to perform.
Certain actions may require authentication or authorization,
while others do not.

- [Workflows](#workflows)
- [Conditional hooks](#conditional-hooks)
    - [iffElse](#iffelse)
    - [iff](#iff)
    - [else](#else)
    - [when](#when)
    - [unless](#unless)
- [Predicate functions](#predicate-functions)
    - [every](#every)
    - [some](#some)
    - [isNot](#isnot)
    - [isProvider](#isprovider)
- [Other](#other)
    - [combine](#combine)

## Workflows

These features add decision making capabilities to your hooks.

However they may also be used to create *workflows* or *processes*.
You may, for example, have a metric service for sending time series data to a metrics cluster
and want to call it from different places to send different key/value pairs.

You could create `sendMetric` containing the hooks and conditional logic
necessary, and include it with the hooks where its needed. 

## Conditional hooks

These hooks conditionally run a set of other hooks,
based on the result of a [predicate function](#predicate-functions).

### iffElse
`iffElse(predicate: boolean|Promise|function, trueHooks: [], falseHooks: []): HookFunc`

Resolve the predicate to a boolean.
Run the first set of hooks sequentially if the result is truthy,
the second set otherwise.

- Used as a `before` or `after` hook.
- Predicate may be a sync or async function.
- Hooks to run may be sync, Promises or callbacks.
- `feathers-hooks` catches any errors thrown in the predicate or hook.

```javascript
const { iffElse, populate, serialize } = require('feathers-hooks-common');
app.service('purchaseOrders').after({
  create: iffElse(() => { ... },
    [populate(poAccting), serialize( ... )],
    [populate(poReceiving), serialize( ... )]
  )
});
```

Options

- `predicate` [required] - Determines if hookFuncs should be run or not.
If a function, `predicate` is called with the hook as its param.
It returns either a boolean or a Promise that evaluates to a boolean
- `trueHooks` [optional] - Zero or more hook functions run when `predicate` is truthy.
- `falseHooks` [optional] - Zero or more hook functions run when `predicate` is false.

### iff
`iff(predicate: boolean|Promise|function, ...hookFuncs: HookFunc[]): HookFunc`

Resolve the predicate to a boolean.
Run the hooks sequentially if the result is truthy.

- Used as a `before` or `after` hook.
- Predicate may be a sync or async function.
- Hooks to run may be sync, Promises or callbacks.
- `feathers-hooks` catches any errors thrown in the predicate or hook.

```javascript
const hooks = require('feathers-hooks-common');
const isNotAdmin = adminRole => hook => hook.params.user.roles.indexOf(adminRole || 'admin') === -1;

app.service('workOrders').after({
  // async predicate and hook
  create: hooks.iff(
    () => new Promise((resolve, reject) => { ... }),
    hooks.populate('user', { field: 'authorisedByUserId', service: 'users' })
  )
});

app.service('workOrders').after({
  // sync predicate and hook
  find: [ hooks.iff(isNotAdmin(), hooks.remove('budget')) ]
});
```

Options

- `predicate` [required] - Determines if hookFuncs should be run or not.
If a function, `predicate` is called with the hook as its param.
It returns either a boolean or a Promise that evaluates to a boolean
- `hookFuncs` [optional] - Zero or more hook functions.
They may include other conditional hooks.

### else
`if(...).else(...hookFuncs: HookFunc[]): HookFunc`

`iff().else()` is similar to `iff` and `iffElse`.
It's syntax is more suitable for writing nested conditional hooks.
If the predicate in the `iff()` is falsey, run the hooks in `else()` sequentially.

- Used as a `before` or `after` hook.
- Hooks to run may be sync, Promises or callbacks.
- `feathers-hooks` catches any errors thrown in the predicate or hook.

```javascript
service.before({
  create:
    hooks.iff(isServer,
      hookA,
      hooks.iff(isProvider('rest'), hook1, hook2, hook3)
        .else(hook4, hook5),
      hookB
    )
      .else(
        hooks.iff(hook => hook.path === 'users', hook6, hook7)
      )
});
```

Options

- `hookFuncs` [optional] - Zero or more hook functions.
They may include other conditional hooks.

### when

An alias for `iff`.

### unless
`unless(predicate: boolean|Promise|function, ...hookFuncs: HookFunc[]): HookFunc`

Resolve the predicate to a boolean.
Run the hooks sequentially if the result is falsey.

- Used as a `before` or `after` hook.
- Predicate may be a sync or async function.
- Hooks to run may be sync, Promises or callbacks.
- `feathers-hooks` catches any errors thrown in the predicate or hook.

```javascript
service.before({
  create:
    unless(isServer,
      hookA,
      unless(isProvider('rest'), hook1, hook2, hook3),
      hookB
    )
});
```

Options

- `predicate` [required] - Determines if hookFuncs should be run or not.
If a function, `predicate` is called with the hook as its param.
It returns either a boolean or a Promise that evaluates to a boolean.
- `hookFuncs` [optional] - Zero or more hook functions.
They may include other conditional hook functions.

## Predicate functions

Predicate functions are used with [conditional hooks](#conditional-hooks).
They return truthy or falsey values

### every
`every(...hookFuncs: HookFunc[]): boolean`

Run hook functions in parallel.
Return `true` if every hook function returned a truthy value.

- Used as a predicate function with [conditional hooks](#conditional-hooks).
- The current `hook` is passed to all the hook functions, and they are run in parallel.
- Hooks to run may be sync or Promises only.
- `feathers-hooks` catches any errors thrown in the predicate.

```javascript
service.before({
  create: hooks.iff(hooks.every(hook1, hook2, ...), hookA, hookB, ...)
});
```
```javascript
hooks.every(hook1, hook2, ...).call(this, currentHook)
  .then(bool => { ... });
```

Options

- `hookFuncs` [required] Functions which take the current hook as a param and return a boolean result.

### some
`some(...hookFuncs: HookFunc[]): boolean`

Run hook functions in parallel.
Return `true` if any hook function returned a truthy value.

- Used as a predicate function with [conditional hooks](#conditional-hooks).
- The current `hook` is passed to all the hook functions, and they are run in parallel.
- Hooks to run may be sync or Promises only.
- `feathers-hooks` catches any errors thrown in the predicate.

```javascript
service.before({
  create: hooks.iff(hooks.some(hook1, hook2, ...), hookA, hookB, ...)
});
```
```javascript
hooks.some(hook1, hook2, ...).call(this, currentHook)
  .then(bool => { ... });
```

Options

- `hookFuncs` [required] Functions which take the current hook as a param and return a boolean result.

### isNot
`isNot(predicate: boolean|Promise|function)`

Negate the `predicate`.

- Used as a predicate with [conditional hooks](#conditional-hooks).
- Predicate may be a sync or async function.
- `feathers-hooks` catches any errors thrown in the predicate.

```javascript
import hooks, { iff, isNot, isProvider } from 'feathers-hooks-common';
const isRequestor = () => hook => new Promise(resolve, reject) => ... );

app.service('workOrders').after({
  iff(isNot(isRequestor()), hooks.remove( ... ))
});
```

Options

- `predicate` [required] - A function which returns either a boolean or a Promise that resolves to a boolean.


### isProvider
`isProvider(provider: string, providers?: string[])`

Check which transport called the service method.
All providers ([REST](../rest/readme.md), [Socket.io](../real-time/socket-io.md) and [Primus](../real-time/primus.md)) set the `params.provider` property which is what `isProvider` checks for.

 - Used as a predicate function with [conditional hooks](#conditional-hooks).

```javascript
import hooks, { iff, isProvider } from 'feathers-hooks-common';

app.service('users').after({
  iff(isProvider('external'), hooks.remove( ... ))
});
```

Options

- `provider` [required] - The transport that you want this hook to run for. Options are:
  - `server` - Run the hook if the server called the service method.
  - `external` - Run the hook if any transport other than the server called the service method.
  - `socketio` - Run the hook if the Socket.IO provider called the service method.
  - `primus` - If the Primus provider.
  - `rest` - If the REST provider.
- `providers` [optional] - Other transports that you want this hook to run for.
  
## Other
  
### combine
`combine(...hooks: HookFunc[])`

Sequentially execute multiple hooks.
`combine` is usually used [within custom hook functions](./utils-hooks.md#combine).

```javascript
service.before({
  create: hooks.combine(hook1, hook2, hook3) // same as [hook1, hook2, hook3]
});
```

Options

- `hooks` [optional] - The hooks to run.