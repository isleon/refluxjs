# RefluxJS

A simple library for unidirectional dataflow architecture inspired by ReactJS [Flux](http://facebook.github.io/react/blog/2014/05/06/flux.html).

[![Build Status](https://travis-ci.org/spoike/refluxjs.svg?branch=master)](https://travis-ci.org/spoike/refluxjs)

You can read an overview of Flux [here](http://facebook.github.io/react/docs/flux-overview.html), however the gist of it is to introduce a more functional programming style architecture by eschewing MVC like pattern and adopting a single data flow pattern.

```
╔═════════╗       ╔════════╗       ╔═════════════════╗
║ Actions ║──────>║ Stores ║──────>║ View Components ║
╚═════════╝       ╚════════╝       ╚═════════════════╝
     ^                                      │
     └──────────────────────────────────────┘

```

The pattern is composed of actions and data stores, where actions initiate new data to pass through data stores before coming back to the view components again. If a view component has an event that needs to make a change in the application's data stores, they need to do so by signalling to the stores through the actions available.

## Comparing RefluxJS with Facebook Flux

The goal of the refluxjs project is to get this architecture easily up and running in your web application, both client-side or server-side. There are some differences between how this project works and how Facebook's proposed Flux architecture works:

You can read more in this [blog post about React Flux vs Reflux](http://spoike.ghost.io/deconstructing-reactjss-flux/).

### Similarities with Flux

Some concepts are still in Reflux in comparison with Flux:

* There are actions
* There are data stores
* The data flow is unidirectional

### Differences with Flux

Reflux has refactored Flux to be a bit more dynamic and be more FRP friendly:

* The singleton dispatcher is removed in favor for letting every action act as dispatcher instead.
* Because actions are listenable, the stores may listen to them. Stores don't need to have a big switch statements that does static type checking (of action types) with strings
* Stores may listen to other stores, i.e. it is possible to create stores that can *aggregate data further*, similar to a map/reduce.
* `waitFor` is replaced in favor to handle *serial* and *parallel* data flows:
 * **Aggregate data stores** (mentioned above) may listen to other stores in *serial*
 * **Joins** for joining listeners in *parallel*
* *Action creators* are not needed because RefluxJS actions are functions that will pass on the payload they receive to anyone listening to them

## Examples

You can find some example projects at these locations:

* [Todo Example Project](https://github.com/spoike/refluxjs-todo) - [http://spoike.github.io/refluxjs-todo/](http://spoike.github.io/refluxjs-todo/)

## Installation

You can currently install the package as a npm package or bower.

### NPM

The following command installs reflux as an npm package:

    npm install reflux

### Bower

The following command installs reflux as a bower component that can be used in the browser:

    bower install reflux

## Usage

For a full example check the [`test/index.js`](test/index.js) file.

### Creating actions

Create an action by calling `Reflux.createAction`.

```javascript
var statusUpdate = Reflux.createAction();
```

An action is a functor that can be invoked like any function.

```javascript
statusUpdate(); // Invokes the action statusUpdate
```

It is as simple as that. There is also a convenience function for creating multiple actions.

```javascript
var Actions = Reflux.createActions([
    "statusUpdate",
    "statusEdited",
    "statusAdded"
  ]);

// Actions object now contains the actions
// with the names given in the array above
// that may be invoked as usual

Actions.statusUpdate();
```

#### Action hooks

There are a couple of hooks avaiable for each action.

* `preEmit` - Is called before the action emits an event. It receives the arguments from the action invocation.

* `shouldEmit` - Is called after `preEmit` and before the action emits an event. By default it returns `true` which will let the action emit the event. You may override this if you need to check the arguments that the action receives and see if it needs to emit the event.

Example usage:

```javascript
Actions.statusUpdate.preEmit = function() { console.log(arguments); };
Actions.statusUpdate.shouldEmit = function(value) {
    return value > 0;
};

Actions.statusUpdate(0);
Actions.statusUpdate(1);
// Should output: 1
```

### Creating data stores

Create a data store much like ReactJS's own `React.createClass` by passing a definition object to `Reflux.createStore`. You may set up all action listeners in the `init` function and register them by calling the store's own `listenTo` function.

```javascript
// Creates a DataStore
var statusStore = Reflux.createStore({

    // Initial setup
    init: function() {

        // Register statusUpdate action
        this.listenTo(statusUpdate, this.output);
    },

    // Callback
    output: function(flag) {
        var status = flag ? 'ONLINE' : 'OFFLINE';

        // Pass on to listeners
        this.trigger(status);
    }

});
```

In the above example, whenever the action is called, the store's `output` callback will be called with whatever parameters was sent in the action. E.g. if the action is called as `statusUpdate(true)` then the flag argument in `output` function is `true`.

### Listening to changes in data store

In your component, register to listen to changes in your data store like this:

```javascript
// Fairly simple view component that outputs to console
function ConsoleComponent() {

    // Registers a console logging callback to the statusStore updates
    statusStore.listen(function(status) {
        console.log('status: ', status);
    });
};

var consoleComponent = new ConsoleComponent();
```

Invoke actions as if they were functions:

```javascript
statusUpdate(true);
statusUpdate(false);
```

With the setup above this will output the following in the console:

```
status:  ONLINE
status:  OFFLINE
```

#### React component example

Register your component to listen for changes in your data stores, preferably in the `componentDidMount` [lifecycle method](http://facebook.github.io/react/docs/component-specs.html) and unregister in the `componentWillUnmount`, like this:

```javascript
var Status = React.createClass({
    initialize: function() { },
    onStatusChange: function(status) {
        this.setState({
            currentStatus: status
        });
    },
    componentDidMount: function() {
        this.unsubscribe = statusStore.listen(this.onStatusChange);
    },
    componentWillUnmount: function() {
        this.unsubscribe();
    },
    render: function() {
        // render specifics
    }
});
```

#### Convenience mixin for React

You always need to unsubscribe components from observed actions and stores upon
unmounting. To simplify this process you can use [mixins in React](http://facebook.github.io/react/docs/reusable-components.html#mixins). There is a convenience mixin available at `Reflux.ListenerMixin`.

```javascript
var Status = React.createClass({
    mixins: [Reflux.ListenerMixin],
    onStatusChange: function(status) {
        this.setState({
            currentStatus: status
        });
    },
    componentDidMount: function() {
        this.listenTo(statusStore, this.onStatusChange);
    },
    render: function() {
        // render specifics
    }
});
```

The mixin provides the `listenTo` method for the React component, that works much like the one found in the Reflux's stores, and handles the listeners during mount and unmount for you.

### Listening to changes in other data stores (aggregate data stores)

A store may listen to another store's change, making it possible to safely chain stores for aggregated data without affecting other parts of the application. A store may listen to other stores using the same `listenTo` function as with actions:

```javascript
// Creates a DataStore that listens to statusStore
var statusHistoryStore = Reflux.createStore({
    init: function() {

        // Register statusStore's changes
        this.listenTo(statusStore, this.output);

        this.history = [];
    },

    // Callback
    output: function(statusString) {
        this.history.push({
            date: new Date(),
            status: statusString
        });
        // Pass the data on to listeners
        this.trigger(this.history);
    }

});
```
## Advanced usage

### Switching EventEmitter

Don't like to use the EventEmitter provided? You can switch to another one, such as NodeJS's own like this:

```javascript
// Do this before creating actions or stores

Reflux.setEventEmitter(require('events').EventEmitter);
```

### Switching nextTick

Whenever action functors are called, they return immediately through the use of `setTimeout` (`nextTick` function) internally.

You may switch out for your favorite `setTimeout`, `nextTick`, `setImmediate`, et al implementation:

```javascript

// node.js env
Reflux.nextTick(process.nextTick);
```

For better alternative to `setTimeout`, you may opt to use the [`setImmediate` polyfill](https://github.com/YuzuJS/setImmediate).


### Joining parallel listeners with composed listenables

Reflux makes it easy to listen to actions and stores that emit events in parallel. You can use this feature to compose and share listenable objects (composed listenables) among several stores.

```javascript
var theTide = Reflux.all(waveAction, timeStore);

var clockStore = Reflux.createStore({
    init: function() {
        this.listenTo(theTide, this.theTideCallback);
    },
    theTideCallback: function(waveActionArgs, timeStoreArgs) {
      // ...
    }
});

// node.js environment
if (process.env.DEVELOPMENT) {
    theTide.listenTo(console.log.bind(console), window);
}
```

`Reflux.all` always passes the last arguments that a listenable emitted to your callback, discarding subsequent emits. Arguments are passed in order. This means that the first argument which the callback receives, is the set of arguments which was emitted by the first listenable that was passed to `Reflux.all` and so on for the other arguments.

#### Comparison with Flux's `waitFor()`

The `Reflux.all` functionality is similar to Flux's `waitFor()`, but differs in a few aspects:

* Composed listenables may be reused by other stores
* Composed listenables always emit asynchronously
* Actions and stores may emit multiple times before the composed listenable (`theTide` in the example above) emits
* Action and store callbacks are not executed in a single *synchronous* iteration

### Sending default data with the listenTo function

The `listenTo` function provided by the `Store` and the `ListenerMixin` has a third parameter that accepts a callback. This callback will be invoked when the listener is registered with whatever the `getDefaultData` is returning.

```javascript
var exampleStore = Reflux.createStore({
    init: function() {},
    getDefaultData: function() {
        return "the initial data";
    }
});

// Anything that will listen to the example store
this.listenTo(exampleStore, onChangeCallback, initialCallback)

// initialCallback will be invoked immediately with "the initial data" as first argument
```

## Colophon

[List of contributors](https://github.com/spoike/reflux/graphs/contributors) is available on Github.

This project is licensed under [BSD 3-Clause License](http://opensource.org/licenses/BSD-3-Clause). Copyright (c) 2014, Mikael Brassman.

For more information about the license for this particular project [read the LICENSE.md file](LICENSE.md).

This project uses [eventemitter3](https://github.com/3rd-Eden/EventEmitter3), is currently MIT licensed and [has it's license information here](https://github.com/3rd-Eden/EventEmitter3/blob/master/LICENSE).
