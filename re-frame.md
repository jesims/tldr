# tldr;

## re-frame ##

re-frame is a pattern for writing SPAs in ClojureScript.

re-frame is a loop of 6 steps. The first three steps are responsible for application state, the last three steps are responsible for the view. Steps 1 and 4 are handled by re-frame. As a developer your job is to implement steps 1, 2, 4, 5.

### 1. Event Dispatch ###

An event is sent when something happens - click a button or receive an event from a websocket:

A re-com/button component would dispatch an on-click event using an anonymous function literal like this:

```clojure
(defn my-delete-button
  []
  [re-com/button
    :label "Delete this item!"
	:on-click #(dispatch [:my-delete-event "123"])])
```

Dispatch emits an event ':my-delete-event'

### 2. Event Handling ###

Register and create an event handler. 

To register a handler, which typically occurs on application startup:

```clojure
(re-frame.core/reg-event-fx :my-delete-event my-delete-handler)
```

And then create a handler for that event:

```clojure
(defn my-delete-handler
  [fx event]                           ;;fx is a coeffects map which holds the current state of the world.
  (let [id (second event)              ;;event is a vector containing the event name and any additional values from the dispatch
        db (:db fx)]
    {:db (dissoc-in db [:items id]}     ;;remove the value in the db and return a new fx
  
```

The coeffects argument contains an entire world view of your application, not just the database as in this example.

re-frame puts all the application state into **ONE** place, the app-db. app-db is a reagent/atom. You should only put data in one place, the app-db.

You need to create a spec schema for your app-db.

The effects map that is returned from the handler function does not need to contain :db. It could contain any combination of [effects](https://github.com/Day8/re-frame/blob/master/docs/External-Resources.md#effect-and-coeffect-handlers)

Handler functions should be side-effect free. If you need to change the world in some way, return a coeffect for that change. See the effects section below.

### 3. Effect Handling ###

The effect handler function `my-delete-handler` returns an *effects* map. Each key in the effects map is an effect, with the values supplying the details of that effect. This example has only one key `:db` , so there is only one effect *(replace the app-db with the key's value)*.

The value of `:db` in the effects map is the new app-db state.

The effects map can have any number of effects. re-frame ships with [default effect handlers](https://github.com/Day8/re-frame/blob/master/docs/Effects.md#builtin-effect-handlers), but you can add your [own](https://github.com/Day8/re-frame-http-fx).

### 4. Query ###

When a new version of app state has been computed and installed, a query function over this app state is reactively called, computing the data to use in the view.

The query function is called a *subscription*

```clojure
(defn my-query-fn
  [db _]                           ;; db is the current app state
  (:items db))                     ;; just a list of ids in this example
```

Query functions are registered as subscriptions when the application is started

```clojure
(re-frame.core/reg-sub :my-query my-query-fn)
```

When a query for :my-query is required, `my-query-fn` will be called to handle it.

### 5. View ###

The query function re-computes a (new) value for a view.

A view that subscribes to the query is called reactively to re-compute DOM.

An example of a view that does this:

```clojure
(defn my-subscribed-view
  []
  (let [items (subscribe [:my-query])]        ;; items is whatever my-query-fn returns
    [:div (map items @items]))                ;; subscriptions always return a ratom, so they need to be dereferenced
```

Here, `items` is sourced from the app-state (via the `my-query-fn`) by a subscription to the registered `:my-query' function from Step 4.

### 6. Dom ###

The computed DOM (hiccup structure returned by the view function `my-subscribed-view`) is made real by re-frame through Reagent / React. You don't need to do anything in this step except understand where it sits in the flow.

### 3-4-5-6 Summary ###

* a change to the app state...
* triggers query functions to rerun...
* triggers view function to rerun...
* which result in new DOM

A typical development process would look like:

1. Design your app's information model and write a spec schema for it
2. Write and register event handlers to handle events and change the application state
3. Write and register query functions which return data for the view
4. Write Reagent view functions that subscribe to the query functions

## Some important additions ##

### Effects ###

When your application needs to create a side-effect *(change the world in some way - call an API or write to local storage, etc)* then you might be tempted to put it in a handler function. Don't.

Handler functions should remain pure.

Pure functions:

* given the same input will always return the same output
* creates no side-effects *(does not change the world in any way)*
* relies on no external state *(gets everything it needs from it's arguments)

This is important because pure functions:

* reduce congnitive load because developers only have to reason locally
* testing is simpler - data in, data out
* allow re-frame developers to replay events

To keep event handler functions pure we can define our own side effects.

First, we create an effect handler

```clojure
(defn my-effects-handler-fn
  [values]                                  ;; values is a map
  (change-the-world-in-some-way values))
```

Then we register it, typically at startup

```clojure
(reg-fx :my-effect my-effects-handler-fn)
```

And then we can use it in an event handler

```clojure
(defn my-handler
  [fx event]
  (let [id (second event)
        db (:db fx)]
    {:db (dissoc-in db [:items id]
	 :my-effect {:a "123"}}
  
```
