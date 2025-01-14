<a href="https://www.taoensso.com" title="More stuff by @ptaoussanis at www.taoensso.com">
<img src="https://www.taoensso.com/taoensso-open-source.png" alt="Taoensso open-source" width="350"/></a>

**[CHANGELOG][]** | [API][] | current [Break Version][]:

```clojure
[com.taoensso/carmine "3.2.0"] ; See CHANGELOG for details
```

> See [here][backers] if to help support my open-source work, thanks! - [Peter Taoussanis][Taoensso.com]

# Carmine: Clojure [Redis][] client & message queue

Redis and Clojure are individually awesome, and **even better together**.

Carmine is a mature Redis client for Clojure that offers an idiomatic Clojure API with plenty of **speed**, **power**, and **ease of use**.

## Features
 * Fast, simple **all-Clojure** library.
 * Fully documented [API][] with support for the **latest Redis commands**.
 * Production-ready **connection pooling**.
 * Auto **de/serialization** of Clojure data types via [Nippy][].
 * Fast, simple **message queue** API.
 * Fast, simple **distributed lock** API.
 * Support for **Lua scripting**, **Pub/Sub**, etc.
 * Support for custom **reply parsing**.
 * Includes **Ring session-store**.

An major upcoming (v4) release will include support for RESP3, Redis Sentinel, Redis Cluster, and more.

## Quickstart

Add the necessary dependency to your project:

```clojure
Leiningen: [com.taoensso/carmine "3.2.0"] ; or
deps.edn:   com.taoensso/carmine {:mvn/version "3.2.0"}
```

And setup your namespace imports:

```clojure
(ns my-app
  (:require [taoensso.carmine :as car :refer [wcar]]))
```

You'll usually want to define a single **connection pool**, and one **connection spec** for each of your Redis servers, example:

```clojure
(defonce my-conn-pool   (car/connection-pool {})) ; Create a new stateful pool
(def     my-conn-spec-1 {:uri "redis://redistogo:pass@panga.redistogo.com:9475/"})

(def my-wcar-opts
  {:pool my-conn-pool
   :spec my-conn-spec-1})
```

This `my-wcar-opts` can then be provided to Carmine's `wcar` ("with Carmine") API:

```clojure
(wcar my-wcar-opts (car/ping)) ; => "PONG"
```

`wcar` is the main entry-point to Carmine's API, see its [docstring](http://ptaoussanis.github.io/carmine/taoensso.carmine.html#var-wcar) for more info on pool and spec opts, etc.

You can create a `wcar` partial for convenience:

```clojure
(defmacro wcar* [& body] `(car/wcar ~my-wcar-opts ~@body))
```

### Command pipelines

Calling multiple Redis commands in a single `wcar` body uses efficient Redis [pipelining][] under the hood, and returns a pipeline reply (vector) for easy destructuring:

```clojure
(wcar*
  (car/ping)
  (car/set "foo" "bar")
  (car/get "foo")) ; => ["PONG" "OK" "bar"] (3 commands -> 3 replies)
```

If the number of commands you'll be calling might vary, it's possible to request that Carmine _always_ return a destructurable pipeline-style reply:

```clojure
(wcar* :as-pipeline (car/ping)) ; => ["PONG"] ; Note the pipeline-style reply
```

If the server replies with an error, an exception is thrown:

```clojure
(wcar* (car/spop "foo"))
=> Exception ERR Operation against a key holding the wrong kind of value
```

But what if we're pipelining?

```clojure
(wcar*
  (car/set  "foo" "bar")
  (car/spop "foo")
  (car/get  "foo"))
=> ["OK" #<Exception ERR Operation against ...> "bar"]
```

### Serialization

The only value type known to Redis internally is the [byte string][]. But Carmine uses [Nippy][] under the hood and understands all of Clojure's [rich datatypes][], letting you use them with Redis painlessly:

```clojure
(wcar*
  (car/set "clj-key"
    {:bigint (bigint 31415926535897932384626433832795)
     :vec    (vec (range 5))
     :set    #{true false :a :b :c :d}
     :bytes  (byte-array 5)
     ;; ...
     })

  (car/get "clj-key"))

=> ["OK" {:bigint 31415926535897932384626433832795N
          :vec    [0 1 2 3 4]
          :set    #{true false :a :c :b :d}
          :bytes  #<byte [] [B@4d66ea88>}]
```

Types are handled as follows:

Clojure type             | Redis type
------------------------ | --------------------------
Strings                  | Redis strings
Keywords                 | Redis strings
Simple numbers           | Redis strings
Everything else          | Auto de/serialized with [Nippy][]

You can force automatic de/serialization for an argument of any type by wrapping it with `car/freeze`.

### Command coverage and documentation

Carmine uses the [official Redis command spec][commands.json] to generate its own command API and documentation:

```clojure
(clojure.repl/doc car/sort)

=> "SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]

Sort the elements in a list, set or sorted set.

Available since: 1.0.0.

Time complexity: O(N+M*log(M)) where N is the number of elements in the list or set to sort, and M the number of returned elements. When the elements are not sorted, complexity is currently O(N) as there is a copy step that will be avoided in next releases."
```

Each Carmine release will use the latest command spec. But if a new Redis command is available that hasn't yet made it to Carmine, you can always use the [redis-call](http://ptaoussanis.github.io/carmine/taoensso.carmine.html#var-redis-call) function to issue **arbitrary commands**.

This is also useful for **Redis module** commands, etc.

### Lua scripting

Redis offers powerful [Lua scripting](http://redis.io/commands/eval/) capabilities.

As an example, let's write our own version of the `set` command:

```clojure
(defn my-set
  [key value]
  (car/lua "return redis.call('set', _:my-key, 'lua '.. _:my-val)"
    {:my-key key}   ; Named     key variables and their values
    {:my-val value} ; Named non-key variables and their values
    ))

(wcar*
  (my-set  "foo" "bar")
  (car/get "foo"))
=> ["OK" "lua bar"]
```

Script primitives are also provided: `eval`, `eval-sha`, `eval*`, `eval-sha*`.

### Helpers

The `lua` command above is a good example of a Carmine *helper*.

Carmine will never surprise you by interfering with the standard Redis command API. But there are times when it might want to offer you a helping hand (if you want it). Compare:

```clojure
(wcar* (car/zunionstore  "dest-key" 3 "zset1" "zset2" "zset3"  "WEIGHTS" 2 3 5))
(wcar* (car/zunionstore* "dest-key"  ["zset1" "zset2" "zset3"] "WEIGHTS" 2 3 5))
```

Both of these calls are equivalent but the latter counted the keys for us. `zunionstore*` is another helper: a slightly more convenient version of a standard command, suffixed with a `*` to indicate that it's non-standard.

Helpers currently include: `atomic`, `eval*`, `evalsha*`, `info*`, `lua`, `sort*`, `zinterstore*`, and `zunionstore*`. See their docstrings for more info.

### Commands are (just) functions

In Carmine, Redis commands are *real functions*. Which means you can *use* them like real functions:

```clojure
(wcar* (doall (repeatedly 5 car/ping)))
=> ["PONG" "PONG" "PONG" "PONG" "PONG"]

(let [first-names ["Salvatore"  "Rich"]
      surnames    ["Sanfilippo" "Hickey"]]
  (wcar* (mapv #(car/set %1 %2) first-names surnames)
         (mapv car/get first-names)))
=> ["OK" "OK" "Sanfilippo" "Hickey"]

(wcar* (mapv #(car/set (str "key-" %) (rand-int 10)) (range 3))
       (mapv #(car/get (str "key-" %)) (range 3)))
=> ["OK" "OK" "OK" "OK" "0" "6" "6" "2"]
```

And since real functions can compose, so can Carmine's. By nesting `wcar` calls, you can fully control how composition and pipelining interact:

```clojure
(let [hash-key "awesome-people"]
  (wcar*
    (car/hmset hash-key "Rich" "Hickey" "Salvatore" "Sanfilippo")
    (mapv (partial car/hget hash-key)
      ;; Execute with own connection & pipeline then return result
      ;; for composition:
      (wcar* (car/hkeys hash-key)))))
=> ["OK" "Sanfilippo" "Hickey"]
```

### Listeners & Pub/Sub

Carmine has a flexible **Listener** API to support persistent-connection features like [monitoring](http://redis.io/commands/monitor) and Redis's fantastic [Pub/Sub](http://redis.io/topics/pubsub) facility:

```clojure
(def listener
  (car/with-new-pubsub-listener (:spec server1-conn)
    {"foobar" (fn f1 [msg] (println "Channel match: " msg))
     "foo*"   (fn f2 [msg] (println "Pattern match: " msg))}
   (car/subscribe  "foobar" "foobaz")
   (car/psubscribe "foo*")))
```

Note the map of message handlers. `f1` will trigger when a message is published to channel `foobar`. `f2` will trigger when a message is published to `foobar`, `foobaz`, `foo Abraham Lincoln`, etc.

Publish messages:

```clojure
(wcar* (car/publish "foobar" "Hello to foobar!"))
```

Which will trigger:

```clojure
(f1 '("message" "foobar" "Hello to foobar!"))
;; AND ALSO
(f2 '("pmessage" "foo*" "foobar" "Hello to foobar!"))
```

You can adjust subscriptions and/or handlers:

```clojure
(with-open-listener listener
  (car/unsubscribe) ; Unsubscribe from every channel (leave patterns alone)
  (car/psubscribe "an-extra-channel"))

(swap! (:state listener) assoc "*extra*" (fn [x] (println "EXTRA: " x)))
```

**Remember to close the listener** when you're done with it:

```clojure
(car/close-listener listener)
```

Note that subscriptions are *connection-local*: you can have three different listeners each listening for different messages, using different handlers. This is great stuff.

### Reply parsing

Want a little more control over how server replies are parsed? You have all the control you need:

```clojure
(wcar*
  (car/ping)
  (car/with-parser clojure.string/lower-case (car/ping) (car/ping))
  (car/ping))
=> ["PONG" "pong" "pong" "PONG"]
```

### Binary data

Carmine's serializer has no problem handling arbitrary byte[] data. But the serializer involves overhead that may not always be desireable. So for maximum flexibility Carmine gives you automatic, *zero-overhead* read and write facilities for raw binary data:

```clojure
(wcar*
  (car/set "bin-key" (byte-array 50))
  (car/get "bin-key"))
=> ["OK" #<byte[] [B@7c3ab3b4>]
```

### Message queue

Redis makes a great [message queue server](http://antirez.com/post/250):

```clojure
(:require [taoensso.carmine.message-queue :as car-mq]) ; Add to `ns` macro

(def my-worker
  (car-mq/worker {:pool {<opts>} :spec {<opts>}} "my-queue"
   {:handler (fn [{:keys [message attempt]}]
               (println "Received" message)
               {:status :success})}))

(wcar* (car-mq/enqueue "my-queue" "my message!"))
%> Received my message!

(car-mq/stop my-worker)
```

Guarantees:
 * Messages are persistent (durable) as per Redis config
 * Each message will be handled once and only once
 * Handling is fault-tolerant: a message cannot be lost due to handler crash
 * Message de-duplication can be requested on an ad hoc (per message) basis. In these cases, the same message cannot ever be entered into the queue more than once simultaneously or within a (per message) specifiable post-handling backoff period.

See the relevant [API][] docs for details.

### Distributed locks

```clojure
(:require [taoensso.carmine.locks :as locks]) ; Add to `ns` macro

(locks/with-lock
  {:pool {<opts>} :spec {<opts>}} ; Connection details
  "my-lock" ; Lock name/identifier
  1000 ; Time to hold lock
  500  ; Time to wait (block) for lock acquisition
  (println "This was printed under lock!"))
```

Again: simple, distributed, fault-tolerant, and _fast_.

See the relevant [API][] docs for details.

## Tundra (beta)

Redis is a beautifully designed datastore that makes some explicit engineering tradeoffs. Probably the most important: your data _must_ fit in memory. Tundra helps relax this limitation: only your **hot** data need fit in memory. How does it work?

 1. Use Tundra's `dirty` command **any time you modify/create evictable keys**
 2. Use `worker` to create a threaded worker that'll **automatically replicate dirty keys to your secondary datastore**
 3. When a dirty key hasn't been used in a specified TTL, it will be **automatically evicted from Redis** (eviction is optional if you just want to use Tundra as a backup/just-in-case mechanism)
 4. Use `ensure-ks` **any time you want to use evictable keys** - this'll extend their TTL or fetch them from your datastore as necessary

That's it: two Redis commands, and a worker! Tundra uses Redis' own dump/restore mechanism for replication, and Carmine's own message queue to coordinate the replication worker.

Tundra can be _very_ easily extended to **any K/V-capable datastore**. Implementations are provided out-the-box for: **Disk**, **Amazon S3** and **DynamoDB**.

```clojure
(:require [taoensso.carmine.tundra :as tundra :refer (ensure-ks dirty)]
          [taoensso.carmine.tundra.s3]) ; Add to ns

(def my-tundra-store
  (tundra/tundra-store
    ;; A datastore that implements the necessary (easily-extendable) protocol:
    (taoensso.carmine.tundra.s3/s3-datastore {:access-key "" :secret-key ""}
      "my-bucket/my-folder")))

;; Now we have access to the Tundra API:
(comment
 (worker    my-tundra-store {} {}) ; Create a replication worker
 (dirty     my-tundra-store "foo:bar1" "foo:bar2" ...) ; Queue for replication
 (ensure-ks my-tundra-store "foo:bar1" "foo:bar2" ...) ; Fetch from replica when necessary
)
```

Note that this API makes it convenient to use several different datastores simultaneously (perhaps for different purposes with different latency requirements).

See the relevant [API][] docs for details.

## Performance

Redis is probably most famous for being [fast](http://redis.io/topics/benchmarks). Carmine hold up its end and usu. performs w/in ~10% of the official C `redis-benchmark` utility despite offering features like command composition, reply parsing, etc.

## Thanks to Navicat

![Navicat-logo](https://github.com/ptaoussanis/carmine/blob/master/navicat-logo.png)

Carmine's SQL-interop features (forthcoming) were developed with the help of [Navicat](http://www.navicat.com/), kindly sponsored by PremiumSoft CyberTech Ltd.

## Thanks to YourKit

Carmine was developed with the help of the [YourKit Java Profiler](http://www.yourkit.com/java/profiler/index.jsp). YourKit, LLC kindly supports open source projects by offering an open source license.

## This project supports the ![ClojureWerkz-logo](https://raw.github.com/clojurewerkz/clojurewerkz.org/master/assets/images/logos/clojurewerkz_long_h_50.png) goals

 * [ClojureWerkz](http://clojurewerkz.org/) is a growing collection of open-source, **batteries-included Clojure libraries** that emphasise modern targets, great documentation, and thorough testing.

## Community tools, etc.

Link                       | Description
-------------------------- | -----------------------------------------------------
[@lantiga/redlock-clj][]   | Distributed locks for uncoordinated Redis clusters
[@danielsz/system][]       | PubSub component for system
[@oliyh/carmine-streams][] | Higher order stream functionality including consumers and message rebalancing
Your link here?            | **PR's welcome!**

## Contacting me / contributions

Please use the project's [GitHub issues page][] for all questions, ideas, etc. **Pull requests welcome**. See the project's [GitHub contributors page][] for a list of contributors.

Otherwise, you can reach me at [Taoensso.com][]. Happy hacking!

\- [Peter Taoussanis][Taoensso.com]

## License

Distributed under the [EPL v1.0][] \(same as Clojure).  
Copyright &copy; 2012-2022 [Peter Taoussanis][Taoensso.com].

<!--- Standard links -->
[Taoensso.com]: https://www.taoensso.com
[Break Version]: https://github.com/ptaoussanis/encore/blob/master/BREAK-VERSIONING.md
[backers]: https://taoensso.com/clojure/backers

<!--- Standard links (repo specific) -->
[CHANGELOG]: https://github.com/ptaoussanis/carmine/releases
[API]: http://ptaoussanis.github.io/carmine/
[GitHub issues page]: https://github.com/ptaoussanis/carmine/issues
[GitHub contributors page]: https://github.com/ptaoussanis/carmine/graphs/contributors
[EPL v1.0]: https://raw.githubusercontent.com/ptaoussanis/carmine/master/LICENSE
[Hero]: https://raw.githubusercontent.com/ptaoussanis/carmine/master/hero.png "Title"

<!--- Unique links -->
[Redis]: http://www.redis.io/
[Nippy]: https://github.com/ptaoussanis/nippy
[@lantiga/redlock-clj]: https://github.com/lantiga/redlock-clj
[@danielsz/system]: https://github.com/danielsz/system
[@oliyh/carmine-streams]: https://github.com/oliyh/carmine-streams
[pipelining]: http://redis.io/topics/pipelining
[byte string]: http://redis.io/topics/data-types
[rich datatypes]: http://clojure.org/datatypes
[commands.json]: https://github.com/redis/redis-doc/blob/master/commands.json
