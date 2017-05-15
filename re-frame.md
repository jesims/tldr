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

### Our re-frame Standard ###

* For maintainability and flexibility, register all events using `reg-event-fx`
* All dom markup within the render function should be invoked by vector notation (not paren notation).

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
(defn my-handler-alt
  [{:keys [db]} [_ id]]
  {:db (dissoc-in db [:items id])
   :my-effect {:a "123"}})
```

### Invoking Render and UI Change ###

React is very efficient at it's rendering and only applies updates to the sections of the DOM that have changed.
A re-evaluation will occur whenever a ratom/reaction value has changed, and it has been de-referenced within a render function.

#### Form-1 Example ####

With a Form-1 notation, the entire form represents the renderable function. Any change to the arguments
(`args`) will cause a re-evaluation and UI update.

```clojure
(defn my-form1-renderable [& args]
  [:p (first args)]) ; This paragraph will change whenever args changes

(defn my-form1-renderable [sub]
  [:p @sub]) ; This paragraph will change whenever the subscription changes
```

#### Form-2 Example ####

With the higher order Form-2 notation, only the returned function is considered in the render cycle. Any changes
to the outer arguments (`args`) will not cause a UI update. 

```clojure
(defn my-form2-renderable [& args]
  (let [v (first args)]
    (fn []
      [:p v]))) ; This paragraph will never change

(defn my-form2-renderable [& args]
  (fn []
    [:p (first args]))) ; This paragraph will never change

(defn my-form2-renderable [sub]
  (let [v @sub]
    (fn []
      [:p v]))) ; This paragraph will never change as the subscription is de-referenced outside the render scope

(defn my-form2-renderable [sub]
  (fn []
    [:p @sub))) ; This paragraph will change whenever the subscription changes

(defn my-form2-renderable []
  (fn [sub]
    [:p v])) ; This paragraph will change whenever the subscription changes 
```

