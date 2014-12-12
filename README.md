## Status 

Alpha. Incomplete. But getting close.  

How to do subscriptions?  Macro which hides the `reaction`? How to dispose of the reaction.

## re-frame

re-frame is a tiny [reagent] framework for writing  [SPAs] using ClojureScript.

It proposes a pattern for structuring an app, and provides a small library
implementing one version of this pattern.

In another context, re-frame might be called an MVC framework, except
it is instead a functional RACES framework - Reactive-Atom Component Event Subscription
(I love the smell of acronym in the morning).

## Claims

Nothing about re-frame is the slightest bit original or clever. 
You'll find no ingenious use of functional zippers, transducers or core.async. 
This is a good thing (although, for the record, one day I'd love to develop
something original and clever).

Using re-frame, you will be able to break your application code into distinct pieces. 
Each of these pieces can be easily described, understood and tested independently. 
These pieces will (mostly) be pure functions.

At small scale, any framework seems like pesky overhead. The 
explanatory examples in here are small scale, so you'll need to
squint a little to see the benefit.

## Core Beliefs

First, above all we believe in the one true [Dan Holmsand] (creator of reagent), 
and his divine instrument the `ratom`.  We genuflect towards Sweden once a day. 

Second, we believe that [FRP] is a honking great idea.  We think you only really 
"get" Reagent once you view it as an [FRP] library, and not simply a 
ReactJS wrapper. To put that another way, we think that Reagent at its best is closer in 
nature to [Hoplon] or [Elm] than it is [OM]. This wasn't obvious to us initially - we
knew we liked reagent, but it took a while for the penny to drop as to why.

Finally, we believe in one way data flow.  We don't like read/write `cursors` which
allow for the two way flow of data. re-frame does implement two data way flow, but it 
uses two, one-way flows to do it.

If you aren't familiar with FRP, or even if you think you are, I'd recomend reading [this FRP backgrounder](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754 before you go any further.

## The Parts

To explain re-frame, we'll now incrementally 
develop a diagram.  We'll explain each part as it is added.

<blockquote class="twitter-tweet" lang="en"><p>Well-formed Data at rest is as close to perfection in programming as it gets. All the crap that had to happen to put it there however...</p>&mdash; Fogus (@fogus) <a href="https://twitter.com/fogus/status/454582953067438080">April 11, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

##### The Big Ratom 

Our re-frame diagram starts with the "well formed data at rest" bit: 
```
app-db
```

re-frame recomends that you put your data into one place (probably one dirty great
big atom) which we'll call `app-db`. Structure the data in that place, of course. 

Now, this advice is not the slightest bit controversial for 'real' databases, right? 
You'd happily put all your well formed data into Postgres or mysql. But within a running application (in memory), it is different. If you have
background in OO, this data-in-one-place is a hard one to swallow.  You've
spent your life breaking systems into pieces, organised around behaviour and trying
to hide the data.  I still wake up in a sweat some nights thinking about all
that clojure data lying around exposed and passive.

But, as @Fogus said above, data is the easy bit. 

From here on, we'll assume `app-db` is one of these: 
```
(def app-db  (reagent/atom {}))    ;; a reagent atom, containing a map
```

Although it is a reagent atom (ratom), I'd encourage you to actively think about
it as an (in-memory) database. 
It will contain structured data (perhaps with a formal [Prismatic Schema] spec).
You will need to query that data. You will perform CRUD 
and other transformations on it. You'll often want to transact on this
database atomically, etc.  So "in-memory database"
seems a more useful paradigm than plain old atom.

Finally, a clarification:  `app-db` doesn't actually have to be a reagent/atom containing
a map. In theory, re-frame
imposes no requirement here.  It could be a [datascript] database.  But, as you'll see, it
would have to be a "reactive datastore" of some description (an 
"observable" datastore -- one that can tell you when it has changed).

##### The Magic Bit

Reagent provides a `ratom` (reagent atom) and a `reaction`. These are the two key building blocks.

`ratoms` are like normal ClojureScript atoms. You can `swap!` and `reset!` them, `watch` them, etc. 

`reactions` act a bit like functions. Its a macro which wraps some `computation` (some forms?) and returns a `ratom` containing the result of that` computation`.  

The magic bit is that `reaction` will automatically rerun the `computation` whenever the computation's "inputs" change, and it will `reset!` the originally returned `ratom` to the newly conputed value. 

Go on, re-read that paragraph again, then look at this ...

```clojure
(ns example1
  (:require-macros [reagent.ratom :refer [reaction]])
  (:require [reagent.core   :as    r]))
    
(def db-app  (r/atom {:a 1}))                 ;; our base ratom

(def ratom2  (reaction {:b (:a @db-app)}))    ;; reaction wraps a computation 
(def ratom3  (reaction (cond = (:a @db-app)   ;; reaction wraps another computation
                             0 "World"
                             1 "Hello")))

;; notice that both the computations above involve dereferencing db-app
;; notice also that both reactions above return a ratom. 

(println @ratom2)    ;; ==>  {:b 1}       ;; a computed result, involving @db-app
(println @ratom3)    ;; ==> "Hello"       ;; a computed result, involving @db-app

(reset!  db-app  {:a 0})        ;; this change to db-app, triggers recomputation 
                                ;; both ratom2 and ratom3 will get new values.

(println @ratom2)    ;; ==>  {:b 0}    ;; ratom2 is result of {:b (:a @db-app)}
(println @ratom3)    ;; ==> "World"    ;; ratom3 is automatically updated too.
    
;; cleanup 
(dispose ratom2)
(dispose ratom3)
```

So, `reaction` wraps a computation, and returns a `ratom`.  Whenever the "inputs" to the computation change, the computation is rerun and the returned ratom is `reset!` to the new value. The "inputs" to the computation are any ratoms which are dereferenced duration execution of the computation.

While the mechanics are different, this is similar in intent to `lift' in [Elm] and `defc=` in [hoplon].

So, in FRP terms, a `reaction` will produce a "stream" of values, accessible via the ratom it returns. 

The way that reagent harnesses these two building blocks is a delight. 

Okay, that was all important background information.  Let's get back with the diagram.

### The Components

Extending the diagram a bit, we introduce `components`:
```
db-app  -->  components  --> hiccup
```
When using reagent, you write one or more `components`.  Think about
`components` as `pure functions` - data in, hiccup out.  `hiccup` is
ClojureScript data structures which represent DOM.

Here's a trivial component:
```
(defn greet
   []
   [:div "Hello ratoms and recactions"])

(greet n)                
;; ==>  [:div "Hello ratoms and recactions"] 
```

You'll notice that our component is a regular clojure function, nothing special. In this case, it takes no paramters and it returns a vector (hiccup). 

Here is a slightly more interesting component: 
```
(defn greet
   [name]                       ;; 'name' is a ratom, contains a string
   [:div "Hello "  @name])      ;; dereference name here to get out the value it contains 
   
;; create a ratom, containing a string
(def n (reagent/atom "re-frame"))  

;; call our `component` function
(greet n)                   
;; ==>  [:div "Hello " "re-frame"] 
```

Okay, so have we got it that components are:  data in, hiccup out ?

Good, let's introduce a `reaction`:
```
(defn greet
   [name]                       ;; name is a ratom
   [:div "Hello "  @name])      ;; dereference name here, to get out the value it contains 
   
(def n (reagent/atom "re-frame"))

;; The computation '(greet n)' produces hiccup which is stored into 'hiccup-ratom'
(def hiccup-ratom  (reaction (greet n)))    ;; notice the use of reaction

;; what is the result of the initial computation ?
(println @hiccup-ratom)
;; ==>  [:div "Hello " "re-frame"] 

;; now change the ratom which is dereferenced in the computation
(reset! n "blah")            ;;    change n to a new value

;; the computaton '(greet n)'  has been rerun, and 'hiccup-ratom' has an updated value
(println @hiccup-ratom)
;; ==>   [:div "Hello " "blah"] 
```

So, as `n` changes value, `hiccup-ratom` changes value. In fact, we could view a series of changes to `n` as producing a "stream" of changes to `hiccup-ratom` (over time). 

If you understand the **concept** of re-computation, then we're there.

Truth injection time.  I haven't been completely straight with you, so we could just focus on the **concepts**.  Here's the reality -- reagent runs `reactions` (re-computations) via requestAnnimationFrame, which is, say, 16ms in the future, or after the current thread of processing finishes, which ever is the greater. So if you were to actually run the lines of code above one after the other,  you might not see the re-computation done immediately after `n` gets reset!, unless the animation frame has run. Not that this bit of annoying truth really matters much.  All you need is the concept.  

On with my lies and distortions ...

A `component` like `greet` is a bit like the templates you'd find in frameworks
like Django or Rails or Mustache -- it maps data to HTML -- except for two massive differences: 
- you have available the full power of ClojureScript (you are just generating a clojure datastructure). The downside tradeoff is that these are not "designer friendly" HTML templates.
- these components are reactive.  When their "inputs" change, they
  are automatically rerun, producing new hiccup. reagent adroitly shields you from
  the details, but `components` are wrapped by a `reaction`.

Summary: when the stream of data flowing into a `component` changes, the `component` is re-computed, producing a "stream" of output hiccup, which, as we'll see below, is turned into DOM and stitched into the GUI. Reagent largely looks after this part of the "flow" for us. 

### ReactJS

The complete data flow from data to DOM is:

```
app-db  -->  components --> Hiccup --> Reagent --> VDOM  -->  ReactJS  --> DOM
```

Best to imagine this process as a pipeline of 3 functions.  Each
function takes data from the
previous step, and produces data for the next step.  In the next
diagram, the three functions are marked. The unmarked nodes are data,
produced by one step, becoming the input to the next step.  hiccup,
VDOM and DOM are all various forms of HTML markup (in our world that's data).

```
app-db  -->  components --> hiccup --> Reagent --> VDOM  -->  ReactJS  --> DOM
               f1                      f2                     f3
```

The combined three-function pipeline should be seen as a pure
function `P` which takes  `app-db` as a parameter, and returns DOM.
```
app-db  -->  P  --> DOM
```

This arrangement is:

1.  Easy to test.  We put data into the ratom, allow the DOM to be rendered, and check that the right divs are in place.  This would typically be done via phantomjs. 
2.  Easily understood.  Generally, components can be understood in isolation.  In almost all cases, a component is genuinely a pure fucntion, or is conceptually a pure function. 
 
The whole thing might be a multistep process, but we only have to bother ourselves with the writing of the `components`.  Reagent/ReactJS looks after the rest of the pipeline.

If the `app-db` changes, the DOM changes.  The DOM is a function of app-db (the state of the app). 

But wait ... we do have to do some work to kick off the flow correctly ...

### Subscriptions

Back to the first part of our diagram:
```
app-db -->  components --> hiccup
```

How does the flow begin. How does data flow from the `app-db` to the components?  

We want our components to `subscribe` to changes in `app-db`. 

XXX A subscription is a `reaction` ....  (reaction  (get-in [:some :path] @app-db))
XXX needs `identical?` check for efficiency ... only propogate when value has changed
XXX Talk about registration of subscription handlers. 
XXX need to invoke  (dispose XXXX)

components tend to be organised into a heirarchy and often data is flowing from parent to child compoentns. 

But at certain points, for example at the root components, something has to 'subscribe' to `app-db` 


### Event Flow

The data flow from `app-db` to the DOM is the first half of the story. We now need to consider the 2nd part of the story: the data flow in the opposite direction.

In response to user interaction, a DOM will generate
events like "clicked delete button on item 42" or
"unticked the checkbox for 'send me spam'".

These events have to "handled".  The code doing this handling might
 mutate the `app-db`, or requrest more data from thet server, or POST somewhere, etc. 

An app will have many handlers, and collectively
they represent the **control layer of the application**.

The backward data flow of events happens via a conveyor belt:

```
app-db  -->  components  --> Hiccup  --> Reagent  --> VDOM  -->  ReactJS  --> DOM
  ^                                                                            |
  |                                                                            v
  handlers <-------------------  events  ---------------------------------------
                           a "conveyor belt" takes events
                           from the DOM to the handlers
```


Generally, when the user manipulates the GUI, the state of the application changes. In our case, 
that means the `app-db` will change.  After all, it **is** the state.  And the DOM presented to the user is a function of that state. So that's the cycle.  GUI events cause `app-db` change, which then causes a rerender, and the users sees something different.

So handlers, which look after events, are the part of the system which does `app-db` mutation. You
could almost imagine them as a "stored procedure" in a
database. Almost. Stretching it?  We do like our in-memory
database analogies.

### What are events?

Events are data. You choose the format.

Our implementation chooses a vector format. For example:

    [:delete-item 42]

The first item in the vector identifies the event and
the rest of the vector is the optional parameters -- in this cse, the id (42) of the item to delete.

Here are some other example events:
```clojure
    [:set-spam-wanted false]
    [[:complicated :multi :part :key] "a parameter" "another one"  45.6]
```

### Dispatching Events

Events start in the DOM.  They are `dispatched`. 

For example, a button component might look like this: 
```clojure
    (defn yes-button
        []
        [:div  {:class "button-class"
                :on-click  #(dispatch [:yes-button-clicked])}
                "Yes"])
```

Notice the `on-click` handler:
```clojure
    #(dispatch [:yes-button-clicked])
```

With re-frame, we try to keep the DOM as passive as possible.  It is simply a rendering of `app-db`.  So that "on-click" is a simple as we can make it. 

There is a signle `dispatch` function in the entire app, and it takes only one paramter, the event.

Let's update our diagram to show dispatch:
```
app-db  -->  components  --> Hiccup  --> Reagent  --> VDOM  -->  ReactJS  --> DOM
  ^                                                                            |
  |                                                                            v
  handlers <-------------------------------------  (dispatch [event-id  other stuff])
```

### Event Handlers 

Collectively, event handlers provide the control logic in the applications. 

The job of many event handlers is to change the `app-db` in some way. Add an item here, or delete that one there. So often CRUD but sometimes much more.

Even though handlers appear to be about `app-db` mutation, re-frame requires them to be pure fucntions. 
```
   (state-in-app-db, event-vector) -> new-state
```   
re-frame passes to an event handler two paramters: the current state of `app-db` plus the event, and the job of a handler to to return a modified version of the state  (which re-frame will then put back into the `app-db`).


```
(defn handle-delete
    [state [_ item-id]]          ;; notice how event vector is destructured -- 2nd parameter
    (dissoc-in state [:some :path item-id]))     ;; return a modified version of 'state'
```

Because handlers are pure functions, and because they generally only have to handle one situation, they tend to be easy to test and understand.  

### Routing

`dispatch` has to call the right handler. 

XXXX handlers have to be registered 

----------  The End

In our implementation `dispatch` and `router` are merged. 

`dispatch` is the conveyor belt, and it could be implemtned in many ways:
- it could push events into a core.asyc channel. 

`router` could be implemented as:
- a multimethod, and find the right event handler by inspection of `first` on the event vectory. 




[SPAs]:http://en.wikipedia.org/wiki/Single-page_application
[reagent]:http://reagent-project.github.io/
[Dan Holmsand]:https://github.com/holmsand
[Hiccup]:https://github.com/weavejester/hiccup
[FRP]:https://gist.github.com/staltz/868e7e9bc2a7b8c1f754
[Elm]:http://elm-lang.org/
[OM]:https://github.com/swannodette/om
[Prismatic Schema]:https://github.com/Prismatic/schema
[InterViews]:http://www.softwarepreservation.org/projects/c_plus_plus/library/index.html#InterViews
[datascript]:https://github.com/tonsky/datascript
[Hoplon]:http://hoplon.io/
[Pedestal App]:https://github.com/pedestal/pedestal-app