# Fluxible

[![npm version](https://badge.fury.io/js/fluxible.svg)](http://badge.fury.io/js/fluxible)
[![Build Status](https://travis-ci.org/yahoo/fluxible.svg?branch=master)](https://travis-ci.org/yahoo/fluxible)
[![Dependency Status](https://david-dm.org/yahoo/fluxible.svg)](https://david-dm.org/yahoo/fluxible)
[![devDependency Status](https://david-dm.org/yahoo/fluxible/dev-status.svg)](https://david-dm.org/yahoo/fluxible#info=devDependencies)
[![Coverage Status](https://coveralls.io/repos/yahoo/fluxible/badge.png?branch=master)](https://coveralls.io/r/yahoo/fluxible?branch=master)
[![Gitter chat](https://badges.gitter.im/gitterHQ/gitter.png)](https://gitter.im/yahoo/fluxible)

Pluggable container for isomorphic [flux](https://github.com/facebook/flux) applications that provides interfaces that are common throughout the Flux architecture and restricts usage of these APIs to only the parts that need them to enforce the unidirectional flow.

## Install

`npm install --save fluxible`

## Usage

```js
var AppComponent = require('./components/Application.jsx'); // Top level React component
var Fluxible = require('fluxible');
var app = new Fluxible({
    appComponent: AppComponent // optional top level component
});

// Per request/session
var context = app.createContext();
var loadPageAction = require('./actions/loadPage');
context.executeAction(loadPageAction, {/*payload*/}, function (err) {
    if (err) throw err;
    var element = AppComponent({
        // Allow the component to access only certain methods of this session context
        context: context.getComponentContext();
    });
    // OR since the appComponent was passed into the constructor:
    // var element = context.createElement({});

    var html = React.renderToString(element);

    var appState = app.dehydrate(context);
    // Expose appState to the client
});
```

For a more extensive example of usage both on the server and the client, see [flux-examples](https://github.com/yahoo/flux-examples).

### Dehydration/Rehydration

Fluxible uses reserved methods throughout called `rehydrate` and `dehydrate` which are responsible for taking a snapshot of server-side state so that it can be sent to the browser and rehydrated back to the same state in the browser. This naming scheme also extends to [dispatchr](https://github.com/yahoo/dispatchr) which takes care of dehydrating/rehydrating the store instances.

There are two kinds of state within fluxible:

 * **Application State**: Settings and data that are registered on server start
 * **Context State**: Settings and data that are created per context/request

Application level rehydrate method is allowed asynchronous operation in case it needs to load JavaScript or data on demand.

### Context Types

Within a context, Fluxible creates interfaces providing access to only certain parts of the system. These are broken down as such:

 * **Action Context**: interface accessible by action creator methods. Passed as first parameter to all action creators.
 * **Component Context**: interface accessible by React components. Should be passed as prop to top level React component and then propagated to child components that require acess to it.
 * **Store Context**: interface accessible by stores. Passed as first parameter to all stores. See [dispatchr docs](https://github.com/yahoo/dispatchr#constructor-1)

### Creating Plugins

Plugins allow you to extend the interface of each context type. Here, we'll give components access to the `getFoo()` function:

```js
var Fluxible = require('fluxible');
var app = new Fluxible();

app.plug({
    // Required unique name property
    name: 'TestPlugin',
    // Called after context creation to dynamically create a context plugin
    plugContext: function (options) {
        // `options` is the same as what is passed into `createContext(options)`
        var foo = options.foo;
        // Returns a context plugin
        return {
            // Method called to allow modification of the component context
            plugComponentContext: function (componentContext) {
                componentContext.getFoo = function () {
                    return foo;
                };
            },
            //plugActionContext: function (actionContext) {}
            //plugStoreContext: function (storeContext) {}

            // Allows context plugin settings to be persisted between server and client. Called on server
            // to send data down to the client
            dehydrate: function () {
                return {
                    foo: foo
                };
            },
            // Called on client to rehydrate the context plugin settings
            rehydrate: function (state) {
                foo = state.foo;
            }
        };
    },
    // Allows dehydration of application plugin settings
    dehydrate: function () { return {}; },
    // Allows rehydration of application plugin settings
    rehydrate: function (state) {}
});

var context = app.createContext({
    foo: 'bar'
});

context.getComponentContext().getFoo(); // returns 'bar'
// or this.props.context.getFoo() from a React component
```

Example plugins:
 * [fluxible-plugin-fetchr](https://github.com/yahoo/fluxible-plugin-fetchr) - Polymorphic RESTful services
 * [fluxible-plugin-routr](https://github.com/yahoo/fluxible-plugin-routr) - Routing behavior

## Mixin

The mixin (accessible via `require('fluxible').Mixin`) uses React's context to provide access to the component context from within a component. This prevents you from having to pass the context to every component via props. This requires that you pass the component context as the context to React:

```js
var FluxibleMixin = require('fluxible').Mixin;
var Component = React.createClass({
    mixins: [FluxibleMixin],
    getInitialState: function () {
        return this.getStore(FooStore).getState();
    }
});

React.withContext(context.getComponentContext(), function () {
    var html = React.renderToString(<Component />);
});
```

The mixin can also be used to statically list store dependencies and listen to them automatically in componentDidMount. This is done by adding a static property `storeListeners` in your component.

You can do this with an array, which will default all store listeners to call the `onChange` method:

```js
var FluxibleMixin = require('fluxible').Mixin;
var MockStore = require('./stores/MockStore'); // Your store
var Component = React.createClass({
    mixins: [FluxibleMixin],
    statics: {
        storeListeners: [MockStore]
    },
    onChange: function () {
        done();
    },
});
```

Or you can be more explicit with which function to call for each store by using a hash:

```js
var FluxibleMixin = require('fluxible').Mixin;
var MockStore = require('./stores/MockStore'); // Your store
var Component = React.createClass({
    mixins: [FluxibleMixin],
    statics: {
        storeListeners: {
            onMockStoreChange: [MockStore]
        }
    },
    onMockStoreChange: function () {
        done();
    },
});
```

This prevents boilerplate for listening to stores in `componentDidMount` and unlistening in `componentWillUnmount`.

## Helper Utilities

Fluxible also exports [dispatcher's store utilities](https://github.com/yahoo/dispatchr#helper-utilities) so that you do not need to have an additional dependency on dispatchr. They are available by using `require('fluxible/utils/BaseStore')` and `require('fluxible/utils/createStore')`.

## API
[//]: # (API_START)
### Fluxible

Instantiated once across all requests, this holds settings and interfaces that are used across all requests/sessions.

#### Constructor

Creates a new application instance with the following parameters:

 * `options`: An object containing the application settings
 * `options.appComponent` (optional): Stores your top level React component for access using getAppComponent()

#### createContext(contextOptions)

Creates a new FluxibleContext instance passing the `contextOptions` into the constructor. Also iterates over the plugins calling `plugContext` on the plugin if it exists in order to allow dynamic modification of the context.

#### plug(plugin)

Allows custom application wide settings to be shared between server and client. Also allows dynamically plugging the FluxibleContext instance each time it is created by implementing a `plugContext` function that receives the context options.

#### getPlugin(pluginName)

Provides access to get a plugged plugin by name.

#### registerStore(store)

Passthrough to [dispatchr's registerStore function](https://github.com/yahoo/dispatchr#registerstorestoreclass)

#### getAppComponent()

Provides access to the `options.appComponent` that was passed to the constructor. This is useful if you want to create the application in a file that is shared both server and client but then need to access the top level component in server and client specific files.

#### dehydrate(context)

Returns a serializable object containing the state of the Fluxible and passed FluxibleContext instances. This is useful for serializing the state of the application to send it to the client. This will also call any plugins which contain a dehydrate method.

#### rehydrate(state)

Takes an object representing the state of the Fluxible and FluxibleContext instances (usually retrieved from dehydrate) to rehydrate them to the same state as they were on the server. This will also call any plugins which contain a rehydrate method.

### FluxibleContext

Instantiated once per request/session, this provides isolation of data so that it is not shared between requests on the server side.

#### Constructor

Creates a new context instance with the following parameters:

 * `options`: An object containing the context settings
 * `options.app`: Provides access to the application level functions and settings

#### createElement(props)

Instantiates the app level React component (if provided in the constructor) with the given props with an additional `context` key containing a ComponentContext. This is the same as the following assuming AppComponent is your top level React component:

```js
AppComponent({
    context: context.getComponentContext();
});
```

#### executeAction(action, payload, callback)

This is the entry point into an application's execution. The initial action is what begins the flux flow: action dispatches events to stores and stores update their data structures. On the server, we wait for the initial action to finish and then we're ready to render using React. On the client, the components are already rendered and are waiting for store change events.

Parameters:

 * `action`: A function that takes three parameters: `actionContext`, `payload`, `done`
 * `payload`: the action payload
 * `done`: the callback to call when the action has been completed

 ```js
 var action = function (actionContext, payload, done) {
     // do stuff
     done();
 };
 context.executeAction(action, {}, function (err) {
     // action has completed
 });
 ```

#### plug(plugin)

Allows custom context settings to be shared between server and client. Also allows dynamically plugging the ActionContext, ComponentContext, and StoreContext to provide additional methods.

#### getActionContext()

Generates a context interface providing access to only the functions that should be called from actions. By default: `dispatch`, `executeAction`, and `getStore`.

This action context object is used every time an `executeAction` method is called and is passed as the first parameter to the action.

#### getComponentContext()

Generates a context interface providing access to only the functions that should be called from components. By default: `executeAction`, `getStore`. `executeAction` does not allow passing a callback from components so that it enforces actions to be send and forget.

This context interface should be passed in to your top level React component and then sent down to children as needed. These components will now have access to listen to store instances, execute actions, and access any methods added to the component context by plugins.

#### getStoreContext()

Generates a context interface providing access to only the functions that should be called from stores. By default, this is empty. See [store constructor interface](https://github.com/yahoo/dispatchr#constructor) for how to access this from stores.

#### dehydrate()

Returns a serializable object containing the state of the FluxibleContext and its Dispatchr instance. This is useful for serializing the state of the current context to send it to the client. This will also call any plugins whose plugContext method returns an object containing a dehydrate method.

#### rehydrate(state)

Takes an object representing the state of the FluxibleContext and Dispatchr instances (usually retrieved from dehydrate) to rehydrate them to the same state as they were on the server. This will also call any plugins whose plugContext method returns an object containing a rehydrate method.

[//]: # (API_STOP)

## License

This software is free to use under the Yahoo! Inc. BSD license.
See the [LICENSE file][] for license text and copyright information.

[LICENSE file]: https://github.com/yahoo/fluxible/blob/master/LICENSE.md
