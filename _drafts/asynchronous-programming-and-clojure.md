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


## Process = control flow + behavior

Agents & Transducers are complementary.
* Agents are all about control flow, and transducers are all about behavior
* Transducers are reducing function transformers, and reducing functions are the way you interact with agents
* Reducing functions produced by transducers may have side-effects, and agents are side-effect-friendly


```clojure
(def ps (partial partial send))
```

> any respectable Scala type has flatMap

> any respectable Clojure type supports transducers