# Asynchronous programming and Clojure

[Recently](https://www.meetup.com/fr-FR/Clojure-User-Group-Lyon/events/234989542/) I talked about how transducers could help composing asynchronous actions in Clojure.

The title of this post is a reference to [this one](https://alexn.org/blog/2017/01/30/asynchronous-programming-scala.html), which gives an opinionated state-of-the-art of asynchronous programming in Scala. The post is worth reading, it shows a good picture of how [difficult](https://www.infoq.com/presentations/Simple-Made-Easy) it can be to do simple things when you choose a complicated language. Obviously, you *don't need* a complicated language to make correct programs. You *may need* a complicated language if you want your programs to be provably consistent, but it won't give you any clue about its correctness. To prove its correctness, you have to find the pudding, and watch it used by a human (preferably, not yourself).

That said, I'm not here to start a language cock size contest. I like Scala and use it on a daily basis for great good. I also agree with the author in that the [Scala Future](http://docs.scala-lang.org/overviews/core/futures.html) is not optimal and that the [Monix Task](https://monix.io/docs/2x/eval/task.html) is a better default. My point is just to show that this programming style is *almost* available for free in standard Clojure.


## Clojure async primitives

Let's have a quick tour of what Clojure provides us.

### futures
Clojure has blocking futures. There is no such thing as a listenable Future in Clojure, and the best way to reify this idea remains [unclear](http://dev.clojure.org/display/design/Promises). Blocking code is what we want to avoid, so clojure futures won't help us.

### core.async
[core.async](https://github.com/clojure/core.async) is an implementation of [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes). It is a third-party library that is often presented as the official way to manage asynchronicity in Clojure. I won't dive into this topic, some have already done it before, surely better than I could.

Basically, core.async is about non-blocking queues, multiple waits and backpressure. If what you want to achieve is simple asynchronous action composition to avoid callback hell, there is a good chance that core.async would be overkill.

### agents
Agents are one of the most overlooked features of Clojure. Good explanations on about their rationale and applications are scarce. Surprisingly, even the [official documentation](https://clojure.org/reference/agents) is not very enlightening. Some recommend to [avoid them](https://rasterize.io/blog/clojure-the-good-parts.html). The first time I heard of agents, what I saw looked like a weird hack to allow side-effects inside STM transactions. I will try to explain why I now think they are a fundamental primitive, and I hope this rant will be a step towards partial support for agents in ClojureScript.

Agents should be your default weapon each time you have to manage side effects. The ability to manage side effect properly is fundamental in asynchronous programming because most asynchronous apis make extensive use of side effecting functions to generate events saying the job you requested is done, and holding the result of the job (which may be an error). This programming style is commonly referred to as "callback-based", and is well known for its inability to scale in complexity.


## Reactive process = control flow + behavior

The most common form is the 2-arity reducing function, used by the higher-order functions `reduce` its cousin `transduce`. However, `reduce-kv` takes 3-arity reducing functions, and the `map` transducers returns reducing functions of any arity.
A reducing function is any function such that the type of the return value is compatible with the type of the first argument.


### Agents implement control flow

First, let's define the `ps` function (short for **p**artial **s**end but also a common abbreviation for *process*, which can't be coincidence). Given an agent and a reducing function, `ps` will produce the side-effecting function that schedules the execution of a reduction step with the values you pass.
```clojure
(def ps (partial partial send))
```

Next, let's define the `!` function (say *bang*). Given a side-effecting function and optional arguments, `!` will apply the side-effecting function to the arguments and return the function. The sole feature of this function is to be a reducing function, actually it's *the* reducing function that performs the effects held by the accumulator. It will be useful any time we want to see an effect as a reduction.
```clojure
(defn !
  ([f] (f) f)
  ([f a] (f a) f)
  ([f a b] (f a b) f)
  ([f a b c] (f a b c) f)
  ([f a b c & ds] (apply f a b c ds) f))
```

We can now leverage agents to make side-effecting functions thread-safe.
```clojure
(def safe-println (ps (agent println) !))
```

We now have a function we can use in place of our good old println, and avoid overlapping writes to the output in multithreaded contexts, because the agent will ensure concurrent calls will be enqueued to be processed sequentially.


### Transducers in action

So far, the process we defined has no behaviour : it is just passing all messages it gets to another endpoint. If we want our process to apply a transformation to the stream of values, we need to define it in the reducing function we pass to `ps`. Fortunately, since 1.7, Clojure provides us with a very convenient way to [transform reducing functions](https://clojure.org/reference/transducers).

In fact, agents & transducers are complementary.
* Agents are all about control flow, and transducers are all about behavior
* Transducers are reducing function transformers, and reducing functions are the way you interact with agents
* Reducing functions produced by transducers may have side-effects, and agents are side-effect-friendly (unlike atoms or refs)

We can leverage transducers to go on improving our `println`. Let's make it look more like a real logger by adding timestamps and verbosity levels.
```clojure
(import '(java.time ZonedDateTime) '(java.time.format DateTimeFormatter))

(def timestamp #(.format (ZonedDateTime/now) DateTimeFormatter/ISO_INSTANT)

(def severity (zipmap [:debug :info :warn :error :fatal] (range)))

(defn logger [verbosity]
  (comp (map list)                                                                 ;; wrap values
        (filter (comp (fnil (partial <= (severity verbosity)) -1) severity first)) ;; take only severe enough
        (map #(cons (timestamp) %))                                                ;; add a timestamp
        (partial partial apply)))                                                  ;; unwrap values
```

Now that we have defined our logger behavior, we can instanciate it...
```clojure
(def log (ps (agent safe-println) ((logger :warn) !)))                             ;; log only when severity is warn or more
```

...and log some more or less interesting stuff :
```clojure
(log :info "incoming request")                                                     ;; nothing happens
(log :error "bad request format")                                                  ;; writes message with timestamp to stdout
```

## Context-aware transducers

So far, we are only able to send messages on output in reaction to an incoming message on input. This limitation is due to the nature of transducers as stream processors.
However, sometimes the result of a transformation may not be available immediately. We may also want to delay message production on purpose. In these situations, we need to be able to react to messages coming from another channel than the transducer input.
We also need to take care of concurrency, for our process may already be processing a message when the other is ready.

```clojure
(defn after [t f & args]
  (future (Thread/sleep t) (apply f args)))

(defn delay-all-1s [rf]
  (fn [r & args]
    (apply after 1000 send *agent* rf args)
    r))

(def delayed-println (ps (agent safe-println) (delay-all-1s !)))

(delayed-println 42)                                                        ;; will print 42, 1 second later
```

Let's look at delay-all-1s. It looks like a transducer, but it's not. It fails at beeing a transducer because it makes assumptions about the transducing context, that is, beeing run by an agent. If you pass it to `transduce`, it won't work as expected. However, as it stays compatible with transducers, you can compose them freely. You just have to remember to run it on an agent.
```clojure
(def delay-all-1s-inc (comp delay-all-1s (map inc)))

(def delayed-inc-println (ps (agent safe-println) (delay-all-1s-inc !)))

(delayed-inc-println 42)          ;; will print 43, 1 second later
```




```clojure
(defmacro task [& body]
  `(fn [rf#]
     (send *agent* #(rf# % (do ˜@body)))
     rf#))
(task (slow-inc 0))
```



> any respectable Scala type has flatMap

> any respectable Clojure type supports transducers

