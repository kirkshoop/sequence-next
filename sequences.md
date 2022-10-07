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

Sender/Receiver, as described in @P2300R5, can represent a single asynchronous value. A generator of `range<Sender>` can represent a set of asynchronous values, where the range generates each sender synchronously. This paper is about a Sender that can asynchronously send `0-N` values and some of the algorithms that operate on an asynchronous value sequence.

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
   fork(pool, [](sender forked){
      return forked | then_each([](int v){ return std::this_thread::get_id();})}) |
   then_each([&counters](std::thread::id tid) { ++counters[tid];}) |
   ignore_all() |
   then([&counters](){for(auto& c : counters){
    std::cout << c.first << " : " << c.second;
   }}));
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
        try {rethrow_exception(ep);
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

## Sequences

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
