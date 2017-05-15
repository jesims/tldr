# re-frame

[re-frame](https://github.com/Day8/re-frame) is a pattern for writing SPAs using [Reagent](http://reagent-project.github.io/) in [ClojureScript](https://clojurescript.org/)

## The re-frame loop

re-frame is a loop of 6 steps. The first three steps are responsible for application state, the last three steps are responsible for the view. Steps 1 and 4 are handled by re-frame. As a developer your job is to implement steps 1, 2, 4, 5.

### Summary

1. an event is triggered 
1. which gets get handled by event handlers that produce effects
1. effect handlers make a changes to the db
1. triggering subscription functions to rerun
1. triggers view function to rerun
1. which result in an updated DOM

A typical development process would look like:

1. Design your app's data model
1. Write and register event handlers to handle events that change the application state, returning fx
1. Write and register subscription functions which return data for the view
1. Write Reagent view functions that subscribe to the subscriptions, returning Reagent hiccup

### 1. Event Dispatch

An event is sent when something happens like the click a button or an event from a websocket.

A button component could dispatch an on-click event using an anonymous function literal like this:

```clojure
(defn delete-item
  [id]
  [re-com/button
    :label "Delete this item!"
    :on-click #(dispatch [:delete id])])
```

Dispatch emits an event `:delete` with the id.

See [1st Domino - Event Dispatch](https://github.com/Day8/re-frame/blob/master/README.md#1st-domino---event-dispatch) and [Code for Domino 1](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-1)

### 2. Event Handling

Register and create an event handler.
 
To register a handler, which typically occurs on application startup:

```clojure
(re-frame.core/reg-event-fx 
  :delete
  (fn my-delete-handler
    [fx event] 
    (let [db (:db fx)                              ; fx is a coeffects map which holds the current db and other effects
          id (second event)]                       ; event is a vector containing the event name and any additional values from the dispatch
      (assoc fx :db (dissoc-in db [:items id]))))) ; remove the value in the db and return the fx
```

The effects argument contains an entire world view of your application, not just the database as in this example.

re-frame puts all the application state into **ONE** place, the app-db. app-db is a reagent/atom. You should only put data in one place, the app-db.

The effects map that is returned from the handler function does not need to contain :db. It could contain any combination of [effects](https://github.com/Day8/re-frame/blob/master/docs/External-Resources.md#effect-and-coeffect-handlers)

Handler functions are pure (side-effect free). If you need to change the world in some way, return an effects for that change. See the effects section below.

See [2nd Domino - Event Handling](https://github.com/Day8/re-frame/blob/master/README.md#2nd-domino---event-handling) and [Code For Domino 2](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-2)

## 3. Effect Handling

The effect handler function returns an effects map. Each key in the effects map is an effect, with the values supplying the details of that effect. This example has only one key `:db`, so there is only one effect (replace the db with the key's value).

The value of `:db` in the effects map is the new app-db state.

The effects map can have any number of effects. re-frame ships with [default effect handlers](https://github.com/Day8/re-frame/blob/master/docs/Effects.md#builtin-effect-handlers), but you can add your [own](https://github.com/Day8/re-frame-http-fx).

See [3rd Domino - Effect Handling](https://github.com/Day8/re-frame/blob/master/README.md#3rd-domino---effect-handlig) and [Code For Domino 3](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-3)

### 4. Subscriptions

When the db is `reset!`, the subscription functions are called, computing the data to use in the view.

Subscriptions are registered when the application is started:
```clojure
(re-frame.core/reg-sub 
  :item-ids         ; the name of the subscription
  (fn item-ids
    [db]            ; db is the current app state
    (:items db)))   ; list of ids
```

See [Domino 4 - Query](https://github.com/Day8/re-frame/blob/master/README.md#domino-4---query) and [Code For Domino 4](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-4)

### 5. View

A view subscribes to a subscription is called to re-compute DOM.

An example of a view that does this:

```clojure
(defn my-subscribed-view
  []
  (let [items (re-frame.core/subscribe [:item-ids])]
    (fn []
      [:div (map delete-item @items)]))) ; subscriptions always return a reaction, so they need to be dereferenced
```

`items` is sourced from the db via the `:item-ids` subscription from Step 4. `delete-item` is our view from Step 1.

See [Domino 5 - View](https://github.com/Day8/re-frame/blob/master/README.md#domino-5---view) and [Code For Domino 5](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-5)

### 6. DOM

A virtual DOM (hiccup structure returned by the view function `my-subscribed-view`) is made real by re-frame through Reagent and React. You don't need to do anything in this step, except understand where it sits in the flow.

See [Domino 6 - DOM](https://github.com/Day8/re-frame/blob/master/README.md#domino-6---dom) and [Code for Domino 6](https://github.com/Day8/re-frame/blob/master/README.md#code-for-domino-6)

## Some important additions

### Our re-frame Standard

* For flexibility, register all events using `reg-event-fx`
* All hiccup components should be invoked by vector notation (not paren notation)

### Effects

When your application needs to create a side-effect *(change the world in some way - call an API or write to local storage, etc)* then you might be tempted to put it in a handler function. Don't.

Handler functions should remain pure.

Pure functions:

* given the same input will always return the same output
* creates no side-effects *(does not change the world in any way)*
* relies on no external state *(gets everything it needs from it's arguments)

This is important because pure functions:

* reduce cognitive load because developers only have to reason locally
* testing is simpler - data in, data out
* allow re-frame developers to replay events

To keep event handler functions pure, we define our own side effects.

Register the effect handler, typically at startup

```clojure
(reg-fx :my-effect 
  (fn my-effects-handler-fn
    [values]                                ; values is a map
    (change-the-world-in-some-way values)))
```

And then we can use it in an event handler

```clojure
(defn my-handler
  [{:keys [db]} [_ id]]
  {:db (dissoc-in db [:items id])
   :my-effect {:a "123"}})
```

### Invoking Render and UI Change

React is very efficient at it's rendering and only applies updates to the parts of the DOM that have changed.

A reagent re-evaluation will occur whenever a reagent atom or subscription reaction value has changed, and has been de-referenced within a render function.

#### Form-1 Example

With a Form-1 notation, the entire form represents the render function. Any change to the arguments
(`args`) will cause a re-evaluation and UI update.

```clojure
(defn my-form1-renderable [& args]
  [:p (first args)]) ; This paragraph will change whenever args changes

(defn my-form1-renderable [sub-or-atom]
  [:p @sub-or-atom]) ; This paragraph will change whenever the subscription or atom changes
```

#### Form-2 Example

With the higher order Form-2 notation, only the returned function is considered in the render cycle. Any changes
to the outer arguments (`args`) will not cause a UI update. 

```clojure
(defn my-form2-renderable [& args]
  (let [v (first args)]
    (fn []
      [:p v]))) ; This paragraph will never change

(defn my-form2-renderable [& args]
  (fn []
    [:p (first args)])) ; This paragraph will never change

(defn my-form2-renderable [sub]
  (let [v @sub]
    (fn []
      [:p v]))) ; This paragraph will never change as the subscription is de-referenced outside the render 

(defn my-form2-renderable [sub]
  (fn []
    [:p @sub])) ; This paragraph will change whenever the subscription changes

(defn my-form2-renderable []
  (fn [sub]
    [:p @sub])) ; This paragraph will change whenever the subscription changes 
```

