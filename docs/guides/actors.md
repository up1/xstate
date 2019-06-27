# Actors <Badge text="4.6+"/>

[:rocket: Quick Reference](#quick-reference)

The [Actor model](https://en.wikipedia.org/wiki/Actor_model) is a mathematical model of message-based computation that simplifies how multiple "entities" (or "actors") communicate with each other. Actors communicate by sending messages (events) to each other. An actor's local state is private, unless it wishes to share it with another actor, by sending it as an event.

When an actor receives an event, three things can happen:

- A finite number of messages can be **sent** to other actors
- A finite number of new actors can be created (or **spawned**)
- The actor's local state may change (determined by its **behavior**)

State machines and statecharts work very well with the actor model, as they are event-based models of behavior and logic. Remember: when a state machine transitions due to an event, the next state contains:

- The next `value` and `context` (an actor's local state)
- The next `actions` to be executed (potentially newly spawned actors or messages sent to other actors)

> Think of actors like dynamic services in XState.

Actors can be considered a dynamic variant of [invoked services](./communication.md) (same internal implementation!) with two important differences:

- They can be _spawned_ at any time (as an action)
- They can be _stopped_ at any time (as an action)

## Actor API

An actor (as implemented in XState) has an interface of:

- An `id` property, which uniquely identifies the actor in the local system
- A `.send(...)` method, which is used to send events to this actor

They may have optional methods:

- A `.stop()` method which stops the actor and performs any necessary cleanup
- A `.subscribe(...)` method for actors that are [Observable](https://github.com/tc39/proposal-observable).

All the existing invoked service patterns fit this interface:

- [Invoked promises](./communication.md#invoking-promises) are actors that ignore any received events and send at most one event back to the parent
- [Invoked callbacks](./communication.md#invoking-callbacks) are actors that can send events to the parent (first `callback` argument), receive events (second `onReceive` argument), and act on them
- [Invoked machines](./communication.md#invoking-machines) are actors that can send events to the parent (`sendParent(...)` action) or other actors it has references to (`send(...)` action), receive events, act on them (state transitions and actions), spawn new actors (`spawn(...)` function), and stop actors.
- [Invoked observables](./communication.md#invoking-observables) are actors whose emitted values represent events to be sent back to the parent.

## Spawning actors

Just as in Actor-model-based languages like [Akka](https://doc.akka.io/docs/akka/current/guide/introduction.html) or [Erlang](http://www.erlang.org/docs), actors are spawned and referenced in `context` (as the result of an `assign(...)` action).

1. Import the `spawn` function from `'xstate'`
2. In an `assign(...)` action, create a new actor reference with `spawn(...)`

The `spawn(...)` function creates an **actor reference** by providing 1 or 2 arguments:

- `entity` - the (reactive) value or machine that represents the behavior of the actor. Possible types for `entity`:
  - [Machine](./communication.md#invoking-machines)
  - [Promise](./communication.md#invoking-promises)
  - [Callback](./communication.md#invoking-callbacks)
  - Observable
- `name` (optional) - a string uniquely identifying the actor. This should be unique for all spawned actors and invoked services.

```js {13-14}
import { Machine, spawn } from 'xstate';
import { todoMachine } from './todoMachine';

const todosMachine = Machine({
  // ...
  on: {
    'NEW_TODO.ADD': {
      actions: assign({
        todos: (context, event) => [
          ...context.todos,
          {
            todo: event.todo,
            // add a new todoMachine actor with a unique name
            ref: spawn(todoMachine, `todo-${event.id}`)
          }
        ]
      })
    }
    // ...
  }
});
```

If you do not provide a `name` argument to `spawn(...)`, a unique name will be automatically generated. This name will be nondeterministic :warning:.

::: tip
Treat `const actorRef = spawn(someMachine)` as just a normal value in `context`. You can place this `actorRef` anywhere within `context`, based on your logic requirements. As long as it's within an assignment function in `assign(...)`, it will be scoped to the service from where it was spawned.
:::

::: warning
Do not call `spawn(...)` outside of an assignment function. This will produce an orphaned actor (without a parent) which will have no effect.

```js
// ❌ Never call spawn(...) externally
const someActorRef = spawn(someMachine);

// ❌ spawn(...) is not an action creator
{
  actions: spawn(someMachine);
}

// ❌ Do not assign spawn(...) outside of an assignment function
{
  actions: assign({
    // remember: this is called immediately, before a service starts
    someActorRef: spawn(someMachine)
  });
}

// ✅ Assign spawn(...) inside an assignment function
{
  actions: assign({
    someActorRef: () => spawn(someMachine)
  });
}
```

:::

Different types of values can be spawned as actors.

## Sending events to actors

With the [`send()` action](./actions.md#send-action), events can be sent to actors via a [target expression](./actions.md#send-targets):

```js {13}
const machine = Machine({
  // ...
  states: {
    active: {
      entry: assign({
        someRef: () => spawn(someMachine)
      }),
      on: {
        SOME_EVENT: {
          // Use a target expression to send an event
          // to the actor reference
          actions: send('PING', {
            to: context => context.someRef
          })
        }
      }
    }
  }
});
```

## Spawning Promises

Just like [invoking promises](./communication.md#invoking-promises), promises can be spawned as actors. The event sent back to the machine will be a `'done.invoke.<ID>'` action with the promise response as the `data` property in the payload:

```js {11}
// Returns a promise
const fetchData = query => {
  return fetch(`http://example.com?query=${event.query}`).then(data =>
    data.json()
  );
};

// ...
{
  actions: assign({
    ref: (_, event) => spawn(fetchData(event.query))
  });
}
// ...
```

::: tip
It is not recommended to spawn promise actors, as [invoking promises](./communication.md#invoking-promises) is a better pattern for this, since they are dependent on state (self-cancelling) and have more predictable behavior.
:::

## Spawning Callbacks

Just like [invoking callbacks](./communication.md#invoking-callbacks), callbacks can be spawned as actors. This example models a counter-interval actor that increments its own count every second, but can also react to `{ type: 'INC' }` events.

```js {22}
const counterInterval = (callback, receive) => {
  let count = 0;

  const intervalId = setInterval(() => {
    callback({ type: 'COUNT.UPDATE', count });
    count++;
  }, 1000);

  receive(event => {
    if (event.type === 'INC') {
      count++;
    }
  });

  return () => { clearInterval(intervalId); }
}

const machine = Machine({
  // ...
  {
    actions: assign({
      counterRef: () => spawn(counterInterval)
    })
  }
  // ...
});
```

Events can then be sent to the actor:

```js {5-7}
const machine = Machine({
  // ...
  on: {
    'COUNTER.INC': {
      actions: send('INC', {
        to: context => context.ref
      })
    }
  }
  // ...
});
```

## Spawning Observables

Just like [invoking observables](./communication.md#invoking-observables), observables can be spawned as actors:

```js {22}
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

const createCounterObservable = (ms) => interval(ms)
  .pipe(map(count => ({ type: 'COUNT.UPDATE', count })))

const machine = Machine({
  context: { ms: 1000 },
  // ...
  {
    actions: assign({
      counterRef: ({ ms }) => spawn(createCounterObservable(ms))
    })
  }
  // ...
  on: {
    'COUNT.UPDATE': { /* ... */ }
  }
});
```

## Spawning Machines

Machines are the most effective way to use actors, since they offer the most capabilities. Spawning machines is just like [invoking machines](./communication.md#invoking-machines), where a `machine` is passed into `spawn(machine)`:

```js {13,26,30-32}
const remoteMachine = Machine({
  id: 'remote',
  initial: 'offline',
  states: {
    offline: {
      on: {
        WAKE: 'online'
      }
    },
    online: {
      after {
        1000: {
          actions: sendParent('REMOTE.ONLINE')
        }
      }
    }
  }
});

const parentMachine = Machine({
  id: 'parent',
  initial: 'waiting',
  states: {
    waiting: {
      entry: assign({
        localOne: () => spawn(remoteMachine)
      }),
      on: {
        'LOCAL.WAKE': {
          actions: send('WAKE', {
            to: context => context.localOne
          })
        },
        'REMOTE.ONLINE': 'connected'
      }
    },
    connected: {}
  }
});

const parentService = interpret(parentMachine)
  .onTransition(state => console.log(state.value))
  .start();

parentService.send('LOCAL.WAKE');
// => 'waiting'
// ... after 1000ms
// => 'connected'
```

### Syncing and Reading State <Badge text="4.6.1+"/>

One of the main tenets of the Actor model is that actor state is _private_ and _local_ - it is never shared unless the actor chooses to share it, via message passing. Sticking with this model, an actor can _notify_ its parent whenever its state changes by sending it a special "update" event with its latest state. In other words, parent actors can subscribe to their child actors' states.

To do this, set `{ sync: true }` as an option to `spawn(...)`:

```js {4}
// ...
{
  actions: assign({
    someRef: () => spawn(todoMachine, { sync: true })
  });
}
// ...
```

This will automatically subscribe the machine to the spawned child machine's state, which is kept updated in `ref.state`:

```js
someService.onTransition(state => {
  const { someRef } = state.context;

  console.log(someRef.state);
  // => State {
  //   value: ...,
  //   context: ...
  // }
});
```

By default, `sync` is set to `false`; that is, `ref.state === undefined`.

::: warning
Prefer sending events to the parent explicitly (`sendParent(...)`) rather than subscribing to every state change. Syncing with spawned machines can result in "chatty" event logs, since every update from the child results in a new `"xstate.update"` event sent from the child to the parent. Here is an example alternative pattern:

```js {9-12}
// Child machine
// ...
on: {
  CHANGE: {
    actions: assign({ value: (_, event) => event.value })
  },
  SAVE: {
    // Only notify parent of changes on SAVE event
    actions: sendParent({
      type: 'UPDATE_FROM_CHILD',
      data: context
    })
  }
}
// ...
```

:::

## Quick Reference

**Import `spawn`** to spawn actors:

```js
import { spawn } from 'xstate';
```

**Spawn actors** in `assign` action creators:

```js
// ...
{
  actions: assign({
    someRef: (context, event) => spawn(someMachine)
  });
}
// ...
```

**Spawn different types** of actors:

```js
// ...
{
  actions: assign({
    // From a promise
    promiseRef: (context, event) =>
      spawn(
        new Promise((resolve, reject) => {
          // ...
        }, 'my-promise')
      ),

    // From a callback
    callbackRef: (context, event) =>
      spawn((callback, receive) => {
        // send to parent
        callback('SOME_EVENT');

        // receive from parent
        receive(event => {
          // handle event
        });

        // disposal
        return () => {
          /* do cleanup here */
        };
      }),

    // From an observable
    observableRef: (context, event) => spawn(someEvent$),

    // From a machine
    machineRef: (context, event) =>
      spawn(
        Machine({
          // ...
        })
      )
  });
}
// ...
```

**Sync state** with an actor: <Badge text="4.6.1+"/>

```js
// ...
{
  actions: assign({
    someRef: () => spawn(someMachine, { spawn: true })
  });
}
// ...
```

**Reading synced state** from an actor: <Badge text="4.6.1+"/>

```js
service.onTransition(state => {
  const { someRef } = state.context;

  someRef.state;
  // => State { ... }
});
```

**Send event to actor** with `send` action creator:

```js
// ...
{
  actions: send('SOME_EVENT', {
    to: context => context.someRef
  });
}
// ...
```

**Send event from actor** to parent with `sendParent` action creator:

```js
// ...
{
  actions: sendParent('ANOTHER_EVENT');
}
// ...
```

**Reference actors** from `context`:

```js
someService.onTransition(state => {
  const { someRef } = state.context;

  console.log(someRef);
  // => { id: ..., send: ... }
});
```