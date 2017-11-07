# chai-kefir

Chai plugin for asserting on Kefir Observables.

[![Build Status](https://travis-ci.org/kefirjs/chai-kefir.svg?branch=master)](https://travis-ci.org/kefirjs/chai-kefir)

---

## How to Install

Install with npm:

```bash
npm i --save-dev chai-kefir
```

Then register the plugin with chai in your test files:

```js
import chaiKefir from 'chai-kefir';
import { use } from 'chai';

use(chaiKefir);
```

Now the `chai-kefir` assertions are available on the chai Assertion.

## How to Use

At the top of your tests, import `chai-kefir` and `kefir and register Kefir:

```js
import Kefir from 'kefir';
import { use } from 'chai';
import chaiKefir from 'chai-kefir';
```

If you're not using ESModules, make sure you grab the `default` property:

```js
const Kefir = require('kefir');
const { use } = require('chai');
const chaiKefir = require('chai-kefir').default;
```

At the top of your test file, use the exported factory function to create the plugin and register it with `chai`:

```js
const { plugin, activate, send, stream, prop, pool } = chaiKefir(Kefir);
use(plugin);
```

All of the exported functions enable you to interact with Kefir Observables without needing to directly connect them to real or mock sources.

---

# API

## `Factory: (Kefir) => PluginHelpers`

The default export is a factory function that takes the application's Kefir instance returns an object of plugin helpers. Those helpers are documented below.

## `PluginHelpers`

### `plugin: (chai, utils) => void`

The `plugin` function registers `chai-kefir`'s assertions with `chai`. This function should be passed to `chai`'s `use` function.

### `activate: (obs: Kefir.Observable) => void`

`activate` is a simple helper function to turn a stream on.

### `deactivate: (obs: Kefir.Observable) => void`

`deactivate` is a simple helper function to turn a stream off. It can turn off streams that were activated with `activate`. Streams turned on through other means (direct call to `on{Value|Error|End|Any}`, use of `observe`, etc.) need to be deactivated through their complementary mechanisms.

### `send: (obs: Kefir.Observable, values: Array<Event<T>>) => obs`

`send` is a helper function for emitting values into a given observable. Note that the second parameter is an array of values to emit from the observable. The `Event` is generated by the `value`, `error`, and `end` functions. For all three of these functions, the optional `options` object is not needed.

### `value: (value, options: ?{ current }) => Event<Value>`
### `error: (error, options: ?{ current }) => Event<Error>`
### `end: (options: ?{ current }) => Event<End>`

`value` and `error` take a value or error and an optional `options` object and return an `Event` object that can be passed to `send`, `emit`, or `emitInTime`. `end` does not take this value, as the `end` event in Kefir does not send a value with it.

When passing to `send`, the `options` object is ignored. `options` is used by `emit` and `emitInTime` (both described below) to determine whether the event should be treated as a `Kefir.Property`'s current event, error, or end.

### `stream: () => Kefir.Stream`
### `prop: () => Kefir.Property`
### `pool: () => Kefir.Pool`

`stream`, `prop`, and `pool` are helper functions to create empty streams, properties, and pools. These can be used as mock sources to send values into. They have no other behavior.

## Assertions

### `observable`

Asserts whether the expected value is a `Kefir.Observable`. For the other assertions, we recommend chaining off `observable`. `property` below requires it; the rest should for consistency. 

```js
expect(obs).to.be.an.observable();
```

### `property`

Asserts whether the expected value is a `Kefir.Property`. Must be chained with `observable`.

```js
expect(obs).to.be.an.observable.property();
```

### `stream`

Asserts whether the expected value is a `Kefir.Stream`.

```js
expect(obs).to.be.an.observable.stream();
``` 

### `pool`

Asserts whether the expected value is a `Kefir.Pool`.

```js
expect(obs).to.be.an.observable.pool();
```

### `active`

Asserts whether the expected value is an observable that is active.

```js
expect(obs).to.be.an.active.observable();
```

### `emit`

Asserts whether the provided observable emits the expected values synchronously. `emit` takes an array of values to match against and expects them to deep equal the values in the correct order.

Accepts an optional callback to be called after the observable is activated. This is because values emitted into the observable before it's passed to `chai` will not be emitted into the assertion, unless it's a Property.

```js
expect(obs).to.emit([value(1), error(new Error('whoops!')), end()], () => {
    send(obs, [value(1), error(new Error('whoops!')), end()]);
});
```

If `obs` is a `Kefir.Property` with a current value, the expected values should get the options object with `current: true`. Note that given how Properties work, only the last value is current.

```js
send(obs, [value(1)]);
send(obs, [value(2)]);
expect(obs).to.emit([value(2, { current: true }), end()], () => {
    send(obs, [end()]);
});
```

These rules also apply to `emitInTime`.

### `emitInTime`

Assets whether the provided emits the values correctly over time. Uses `lolex` behind the scenes to take over JavaScripts timers, allowing you to assert against the times the values are emitted. The expected value should be an array of tuples, where the first value is the time and the second is the value emitted.

```js
const expected = [
    [0, value(1)],
    [10, error(new Error('whoops!'))],
    [20, end()]
]
```

Accepts a callback which is passed both a simple `tick` function as well as the full `lolex` `clock`. `tick` advances the internal timer by the provided ms. `clock` is documented [here][clock].

```js
expect(obs).to.emit(expected, (tick, clock) => {
    send(obs, [value(1)]);
    tick(10);
    send(obs, [error(new Error('whoops!'))]);
    tick(10);
    send(obs, [end()]);
});
```

`emitInTime` also accepts an optional configuration object after the callback. That object takes the following options:

* `reverseSimultaneous: bool`: Indicates whether callbacks scheduled for the same time should be called in reverse. This is an advanced use case to check if your implementation handles a common browser bug. See this [issue][timer-issue] for more information. This is handled correctly by Kefir's built-in methods, so unless you're using timers in your implementation, this mostly isn't necessary.

  [clock]: https://github.com/sinonjs/lolex/#var-id--clocksettimeoutcallback-timeout
  [timer-issue]: https://github.com/sinonjs/lolex/issues/24
