---
title: "Asynchronous Value Sequences"
subtitle: "Draft Proposal"
document: D0000R0
date: today
audience:
  - "SG1 - parallelism and concurrency"
author:
  - name: Kirk Shoop
    email: <kirk.shoop@gmail.com>
toc: true
---

Introduction
============

This was the end goal all along.

_Sender_/_Receiver_, as described in [@P2300R5], can represent a single asynchronous value. A generator of `range<Sender>` can represent a set of asynchronous values, where the range generates each sender synchronously. This paper is about _Senders_ that can asynchronously send `0..N` values and some of the algorithms, for those _Senders_, that operate on an asynchronous value sequence.

## Basic polling

Polling works well for sampling sensors. 

This particular example is simple, but has limited use as a general pattern. User interactions are better represented with events.

```cpp
sync_wait(generate_each(&::getchar) |
  take_while([](char v) { return v != '0'; }) |
  filter_each(std::not_fn(&::isdigit)) |
  then_each(&::toupper) |
  then_each([](char v) { std::cout << v; }) |
  ignore_all());
```

## Bulk processing

```cpp
auto counters = std::map<std::thread::id, std::ptrdiff_t>{};
sync_wait(itoas(1, 3000000) |
  fork(pool, [](sender auto forked){
    return forked | then_each([](int v){
      return std::this_thread::get_id();
    });
  }) |
  then_each([&counters](std::thread::id tid) { 
    ++counters[tid];
  }) |
  ignore_all() |
  then([&counters](){
    for(auto [tid, c] : counters){
      std::print("{} : {}\n", tid, c);
    }
  }));
```

## Web Requests

```cpp
auto requesttwitterstream = twitter_stream_reconnection(
  defer([=](){
    auto url = oauth2SignUrl("https://stream.twitter...");
    return http.create(http_request{url, method, {}, {}}) |
      then_each([](http_response r){
        return r.body.chunks;
      }) |
      merge_all();
  }));
```

### Connect to a firehose and parse

```cpp
auto tweets = twitterrequest(tweetthread, http) |
  parsetweets(poolthread, tweetthread) |
  publish_all(); // share
```

### Satisfying Web Service Retry contract

```cpp
auto twitter_stream_reconnection = 
  return [=](auto sender chunks){
    return chunks |
      timeout_each(90s, tweetthread) |
      upon_error([=](exception_ptr ep) {
        try {
          rethrow_exception(ep);
        } catch (const http_exception& ex) {
          return twitterRetryAfterHttp(ex);
        } catch (const timeout_error& ex) {
          return empty<string>();
        }
        return error<string>(ep);
      }) |
      repeat_forever();
  };
```

## User Interface

### Apply a set of actions to a model

```cpp
auto models = iterate(actions) |
  merge_each()|
  scan_each(Model{}, [=](Model& m, auto f){
    auto r = f(m);
    return r;
  }) |
  sample_all(200ms, uithread) |
  publish_all(); // share
```

### Apply a set of renderers to a model

```cpp
async_scope scope;
scope.spawn(on(uithread, iterate(renderers) |
  merge_each() | 
  ignore_all()));

ui.loop();
scope.request_stop();
sync_wait(scope.on_empty());
```

Design
======

The basic progression, for a sequence of values, is to have a sender that completes when the sequence ends and a separate sender for each value.

## Consuming a Sequence 

```plantuml
title Sequence using ""set_next()"" 
caption sequence using ""set_next()""
start
:""seqOp = connect(sequenceSndr, consumeRcvr)""]
:""start(seqOp)""]
fork
->1..concurrency;
while (not end-of-sequence and \nnot stop-requested)
->produce value ""N"";
:""consumeNValueSndr = set_next(consumeRcvr, valueNSndr)""]
:""valueNOp = connect(consumeNValueSndr, consumeNRcvr)""]
:""start(valueNOp)""]
->complete ""valueNOp"";
split
  :""set_value(consumeNRcvr, vn...)""]
split again
  :""set_error(consumeNRcvr, err)""]
split again
  :""set_stopped(consumeNRcvr)""]
endsplit
:""++N""}
endwhile 
endfork
->complete ""seqOp"";
split
  :""set_value(consumeRcvr)""]
split again
  :""set_error(consumeRcvr, err)""]
split again
  :""set_stopped(consumeRcvr)""]
endsplit
end
```

## `set_next()`

`set_next` is a new customization point object for the receiver. `set_next` applies algorithms to the given sender of a value. a sequence-receiver concept subsumes a receiver concept. a sequence-receiver is required to only produce a void `set_value` to signal the end of the sequence. 

A sequence operation calls `set_next(receiver, valueSender)` with a sender that will produce the next value. `set_next` applies algorithms to the `valueSender` and returns the resulting sender.

A sequence operation connects and starts the sender returned from each call to `set_next`.

For lock-step sequences, the receiver that the sequence operation connected the sender to will call `set_next` from the 'set_value` completion.

To prevent stack overflow, there needs to be a trampoline scheduler applied to each value sender. A tail-sender will be defined in a separate paper that can be used instead of a scheduler to stop stack overflow.

## Sequences

```plantuml
title Lock-Step [shows connect and start]
caption sequence diagram for lock-step ""set_next()"" w/connect+start
autonumber
hide footbox
participant consumer as cnm
participant producer as pdc
participant transformer as trn
group connect/start transformSeq
cnm -> trn : ""transformOp = connect(transformSeq, consumeRcvr)""
activate cnm
activate trn
trn -> pdc : ""producerOp = connect(produceSeq, transformRcvr)""
activate pdc
deactivate pdc
deactivate trn
cnm -> trn : ""start(transformOp)""
activate trn
trn -> pdc : ""start(producerOp)""
activate pdc
end
deactivate pdc
deactivate trn
deactivate cnm
loop values 0..N
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""consumeValN = set_next(transformRcvr, produceValN)""
activate pdc
activate trn
trn --> cnm : ""consumeValN = set_next(consumeRcvr, transformValN)""
activate cnm
deactivate cnm
deactivate trn
group connect/start consumeValN
pdc -> cnm : ""consumeValNOp = connect(consumeValN, produceValN+1Rcvr)""
activate cnm
cnm -> trn : ""transformValNOp = connect(produceValN, transformValNRcvr)""
activate trn
trn -> pdc : ""produceValNOp = connect(produceValN, consumeValNRcvr)""
activate pdc
deactivate pdc
deactivate trn
deactivate cnm
pdc -> cnm : ""start(consumeValNOp)""
activate cnm
cnm -> trn : ""start(transformValNOp)""
activate trn
trn -> pdc : ""start(produceValNOp)""
end
deactivate trn
deactivate cnm
deactivate pdc
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""set_value(transformValNRcvr, ValN...)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeValNRcvr, transformValN...)""
activate cnm
cnm --> pdc : ""set_value(produceValN+1Rcvr)""
deactivate pdc
deactivate trn
deactivate cnm
end
... wibbily wobbly timey wimey stuff ...
group end of sequence
pdc --> trn : ""set_value(transformRcvr)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeRcvr)""
activate cnm
deactivate cnm
deactivate trn
deactivate pdc
end
```

### Lock-Step

lock-step sequences are inherently serial. The next value will not be emitted until the previous value has been completely processed.

```plantuml
title Lock-Step [omits connect and start]
caption sequence diagram for lock-step ""set_next()"" wo/connect+start
autonumber
hide footbox
participant consumer as cnm
participant producer as pdc
participant transformer as trn
activate cnm
group connect/start
hnote over cnm:""transformSeq -> consumeRcvr""
activate trn
activate pdc
end
deactivate pdc
deactivate trn
deactivate cnm
loop values 0..N
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""consumeValN = set_next(transformRcvr, produceValN)""
activate pdc
activate trn
trn --> cnm : ""consumeValN = set_next(consumeRcvr, transformValN)""
activate cnm
group connect/start 
hnote over pdc:""consumeValN -> produceValN+1Rcvr""
end
deactivate pdc
deactivate trn
deactivate cnm
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""set_value(transformValNRcvr, ValN...)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeValNRcvr, transformValN...)""
activate cnm
cnm --> pdc : ""set_value(produceValN+1Rcvr)""
deactivate pdc
deactivate trn
deactivate cnm
end
... wibbily wobbly timey wimey stuff ...
group end of sequence
pdc --> trn : ""set_value(transformRcvr)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeRcvr)""
activate cnm
deactivate cnm
deactivate trn
deactivate pdc
end
```

### External Event

External events are very common. User events like pointer-move and key-down and sensor readings like orientation and ambient-light, are examples of events that produce sequences of values over time.

```plantuml
title External Event [omits connect and start]
caption sequence diagram for external-event ""set_next()"" wo/connect+start
autonumber
hide footbox
participant consumer as cnm
participant producer as pdc
participant transformer as trn
activate cnm
group connect/start
hnote over cnm:""transformSeq -> consumeRcvr""
activate trn
activate pdc
end
deactivate pdc
deactivate trn
deactivate cnm
group each event
... wibbily wobbly timey wimey stuff ...
[-> pdc : event
activate pdc
pdc --> trn : ""consumeValN = set_next(transformRcvr, produceValN)""
activate trn
trn --> cnm : ""consumeValN = set_next(consumeRcvr, transformValN)""
activate cnm
group connect/start 
hnote over pdc:""consumeValN -> completeValNRcvr""
end
deactivate trn
deactivate cnm
pdc --> trn : ""set_value(transformValNRcvr, ValN...)""
activate trn
trn --> cnm : ""set_value(consumeValNRcvr, transformValN...)""
activate cnm
cnm --> pdc : ""set_value(completeValNRcvr)""
deactivate pdc
deactivate trn
deactivate cnm
end
... wibbily wobbly timey wimey stuff ...
group end of sequence
[-> pdc : stop
activate pdc
pdc --> trn : ""set_value(transformRcvr)""
activate trn
trn --> cnm : ""set_value(consumeRcvr)""
activate cnm
deactivate cnm
deactivate trn
deactivate pdc
end
```

### Parallelism

Sequences may be consumed in parallel. Be it network requests or ML data chunks, there is a need to use overlapping consumers for the values.

```plantuml
title Bulk [omits connect and start]
caption sequence diagram for bulk ""set_next()"" w/connect+start
autonumber
hide footbox
participant consumer as cnm
participant producer as pdc
participant transformer as trn
activate cnm
group connect/start
hnote over cnm:""transformSeq -> consumeRcvr""
activate trn
activate pdc
end
deactivate pdc
deactivate trn
deactivate cnm
par values 0..N
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""consumeValN = set_next(transformRcvr, produceValN)""
activate pdc
activate trn
trn --> cnm : ""consumeValN = set_next(consumeRcvr, transformValN)""
activate cnm
group connect/start 
hnote over pdc:""consumeValN -> completeValNRcvr""
end
deactivate pdc
deactivate trn
deactivate cnm
... wibbily wobbly timey wimey stuff ...
pdc --> trn : ""set_value(transformValNRcvr, ValN...)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeValNRcvr, transformValN...)""
activate cnm
cnm --> pdc : ""set_value(completeValNRcvr)""
deactivate pdc
deactivate trn
deactivate cnm
end
... wibbily wobbly timey wimey stuff ...
group end of sequence
pdc --> trn : ""set_value(transformRcvr)""
activate pdc
activate trn
trn --> cnm : ""set_value(consumeRcvr)""
activate cnm
deactivate cnm
deactivate trn
deactivate pdc
end
```

## Algorithms

### then_each

```plantuml
left to right direction
package then_each: {
(10) -- (20)
(20) -- (30)
(30) -- (|)
rectangle ""then_each([](auto v){return v+1;})""
(11) -- (21)
(21) -- (31)
(31) -- (| )
}
```

### filter_each

```plantuml
left to right direction
package filter_each: {
(10) -- (20)
(20) --- (30)
(30) -- (|)
rectangle ""filter_each([](auto v){return v != 20;})""
(10 ) ---- (30 )
(30 ) -- (| )
}
```

### generate_each

```plantuml
left to right direction
package generate_each: {
rectangle ""generate_each([v=0]() mutable {return v+=10;})""
(10) -- (20)
(20) --- (30)
(30) -- (|)
}
```

### iotas

```plantuml
left to right direction
package iotas: {
rectangle ""iotas(10, 30, 10)""
(10) -- (20)
(20) --- (30)
(30) -- (|)
}
```

### fork

### merge_each

### scan_each

### ignore_all

```plantuml
left to right direction
package ignore_all: {
(10) -- (20)
(20) --- (30)
(30) -- (|)
rectangle ""ignore_all()""
}
```

### take_while

```plantuml
left to right direction
package take_while: {
(10) -- (20)
(20) --- (30)
(30) -- (|)
rectangle ""take_while([](auto v){return v != 20;})""
(10 ) -- (| )
}
```

### sample_all

### timeout_each

