# starkiller

A lightweight, scalable messaging system.

## Theory of Operation

We run some number of nodes running our server, which has a simple "junction" concept within it: a central bus where you can send or receive messages to a topic ID. Messages are transient; they are either sent to a listener, or they are discarded. Sending messages uses a timeout to "block" sending the message until a consumer appears.

Clients use a cluster-aware junction client that uses a service discovery system (such as [consul](https://consul.io)) to discover available nodes. Clients query each node for their set of "tokens": a set of 64-bit integers randomly chosen by the server. When sending a message, the client hashes the topic ID to a 64-bit integer, and selects the node that has the biggest token that is less than the topic hash.

## Network Protocol

Servers run on TCP port 3443 by default. The protocol is MessagePack encoded arrays of "method calls", framed by a 16-bit integer length.

Each method call is a simple array of values:

* Method name, a string, either ":recv!", ":send!", or ":tokens".
* Message ID, a unique integer for the current connection (clients should use an atomically incrementing counter for message IDs).
  
Method calls except ":tokens" will also include more arguments:

* A timeout. A positive integer, the number of milliseconds the caller is willing to wait for the call to succeed.
* The ID being sent to or received from. This can be any value MessagePack can encode, but a string is a typical good data type.
* For `:send!` calls only, the value being sent. This can be any value that can be encoded in MessagePack.

Responses are likewise sent back to the client as arrays of values, laid out as follows:

* The method name, ":send!", ":recv!", or ":tokens".
* The message ID, matching the message ID in the request.
* The value. This will be:
    * The boolean `true`, or the timeout value, for ":send!" calls (so a send will either succeed or time out).
    * The received value, or the timeout value, for ":recv!" calls.
    * The array of tokens for ":tokens" calls.
    * A special timeout value, if the call timed out.
  
The timeout value is encoded as a msgpack extension, with type 0x54 (the character `T`).
The value of the extension is a single zero byte.

The method call:

```clojure
[":recv!", 1, 1000, "foo"]
```

Would be encoded as bytes (in hexadecimal, including the length field prefix):

```
001094a63a726563762101cd03e8a3666f6f
```

The send method call:

```clojure
[":send!", 2, 1000, "foo", "bar"]
```

Would be encoded as bytes:

```
001495a63a73656e642102cd03e8a3666f6fa3626172
```

The response:

```clojure
[":recv!", 1, "bar"]
```

Would be encoded as:

```
000d93a63a726563762101a3626172
```

And the response:

```clojure
[":recv!", 1, timeout-value]
```

Would be encoded as:

```
000c93a63a726563762101d45400
```

## Getting Started

The docker compose file can be used to spin up an example cluster of 2 starkiller nodes and 5 consul nodes:

```
docker compose up -d
```

You can then start an nREPL server within the same network:

```
docker run --network starkiller_default -p 7888:7888 rsdio/starkiller-nrepl
```

Connect to local port 7888 to run a clojure repl. Then you can run:

```clojure
(require '[clojure.core.async :as async])
(require '[starkiller.client :as client])
(require '[starkiller.core :as s])

(def client1 (async/<!! (client/remote-junction (java.net.InetSocketAddress. "starkiller1" 3443) {})))
(def client2 (async/<!! (client/remote-junction (java.net.InetSocketAddress. "starkiller2" 3443) {})))

(let [recv (s/recv! client1 "foo" {})
      send (s/send! client2 "foo" "bar" {})]
  (async/<!! (async/into [] (async/merge [recv send]))))
; should print ["bar" true]
```
