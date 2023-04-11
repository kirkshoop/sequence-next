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


_Sender_/_Receiver_, as described in [@P2300R5], can represent a single asynchronous value. 


Today, using a range-v3 generator, a `range<Sender>` can represent a set of asynchronous values, where the range generates each sender synchronously. 


This paper is about sequence-senders that can asynchronously provide `0..N` value-senders. This paper also describes some of the algorithms that operate on sequence-sender.


Some of the features provided by this design are: 

 - back-pressure, which slows down chatty producers
 - no-allocations, by default
 - parallel value senders

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
  on_each(pool, fork([](sender auto forked){
    return forked | then_each([](int v){
      return std::this_thread::get_id();
    });
  })) |
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

This is 'ported' from a twitter application.

```cpp
auto requesttwitterstream = twitter_stream_reconnection(
  defer_construction([=](){
    auto url = oauth2SignUrl("https://stream.twitter...");
    return http.create(http_request{url, method, {}, {}}) |
      then_each([](http_response r){
        return r.body.chunks;
      }) |
      merge_each();
  }));
```

### Connect to a firehose and parse

```cpp
struct Tweet;

auto tweets = requesttwitterstream |
  parsetweets(poolthread) |
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
      repeat_always();
  };
```

## User Interface

This is 'ported' from a twitter application.

### Build a set of reducers

```cpp
struct Model;
using Reducer = std::function<Model(Model&)>;

vector<any_sender<Reducer>> reducers;

// produce side-effect of dumping text to terminal
reducers.push_back(
  tweets | 
  then_each([](const Tweet& tweet) -> Reducer {
    return [=](Model&& model) -> Model {
      auto text = tweet.dump();
      cout << text << "\r\n";
      return std::move(model);
    };
  });

// group tweets, by the timestamp_ms value
reducers.push_back(
  tweets |
  then_each([](const Tweet& tweet) -> Reducer {
    return [=](Model&& model) -> Model {
        auto ts = timestamp_ms(tweet);
        update_counts_at(model.counts_by_timestamp, ts);
        return std::move(model);
    };
  });

// group tweets, by the local time that they arrived
reducers.push_back(
  tweets |
  then_each([](const Tweet& tweet) -> Reducer {
    return [](Model&& model) -> Model {
        update_counts_at(model.counts_by_arrival, system_clock::now());
        return std::move(model);
    };
  });
```

### Apply a set of reducers to a model

```cpp
// merge many sequences of reducers 
// into one sequence of reducers
// and order them in the uithread
auto actions = on(uithread, iterate(reducers) | merge_each());

auto models = actions |
  // when each reducer arrives
  // apply the reducer to the Model 
  scan_each(Model{}, [=](Model&& m, Reducer rdc){
    auto newModel = rdc(std::move(m));
    return newModel;
  }) |
  // every 200ms emit the latest Model
  sample_all(200ms) |
  publish_all(); // share
```

### Build a set of renderers

```cpp
vector<any_sender<void>> renderers;

auto on_draw = screen.when_render(just()) |
    with_latest_from(models);

renderers.push_back(
  on_draw |
  then_each(render_tweets_window));

renderers.push_back(
  on_draw |
  then_each(render_counts_window));
```

### Apply a set of renderers to a model

```cpp
async_scope scope;
scope.spawn(iterate(renderers) |
  merge_each() | 
  ignore_all());

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

`set_next` is a new customization point object for the receiver. `set_next` applies algorithms to the given sender of a value. a sequence-receiver concept subsumes a receiver concept. a sequence-receiver is required to only produce a void `set_value` to signal the end of the sequence. a sequence-receiver is required to provide `set_next` for all the value senders produced by the sequence operation. 

A sequence operation calls `set_next(receiver, valueSender)` with a sender that will produce the next value. `set_next` applies algorithms to the `valueSender` and returns the resulting sender.

A sequence operation connects and starts the sender returned from each call to `set_next`.

For lock-step sequences, the receiver that the sequence operation connected the sender to will call `set_next` from the 'set_value` completion.

> NOTE: To prevent stack overflow, there needs to be a trampoline scheduler applied to each value sender. A tail-sender will be defined in a separate paper that can be used instead of a scheduler to stop stack overflow.

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

Algorithms
==========

Marble diagrams are often used to describe algorithms for asynchronous sequences.

## then_each

`then_each` applies the given function to each input value and emits the result of the given function.

```plantuml
caption marble diagram for ""then_each""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """then_each([](auto v){return v+1;})""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o11 as "11"
usecase o21 as "21"
usecase o31 as "31"
card oE as "|" #transparent
oS -- o11
o11 -- o21
o21 -- o31
o31 -- oE
}
```

## filter_each

`filter_each` applies the given predicate to each input value and only emits the value if the given predicate returns `true`.

```plantuml
caption marble diagram for ""filter_each""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """filter_each([](auto v){return v != 20;})""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o30 as "30"
card oE as "|" #transparent
oS -- o10
o10 --- o30
o30 -- oE
}
```

## take_while

`take_while` applies the given predicate to each input value and if the given predicate returns `true` cancels the input and emits no more values.

```plantuml
caption marble diagram for ""take_while""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """take_while([](auto v){return v != 20;})""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
card oE as "|" #transparent
oS -- o10
o10 -- oE
}
```

## distinct

`distinct` compares each input value to a stored copy of the previous input value, if the input value and the previous input value are not the same replace the stored copy with the input value and emit the input value, otherwise do not emit the input value.

```plantuml
caption marble diagram for ""distinct""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20a as "20"
usecase i20b as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20a
i20a -- i20b
i20b -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """distinct()""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o20 as "20"
usecase o30 as "30"
card oE as "|" #transparent
oS -- o10
o10 -- o20
o20 --- o30
o30 -- oE
}
```

## ignore_all

`ignore_all` does not emit any input values. This converts a sequence of values to a sender-of-void that can be passed to `sync_wait()`, etc..

```plantuml
caption marble diagram for ""ignore_all""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """ignore_all()""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
card oE as "|" #transparent
oS ----- oE
}
```

## generate_each

`generate_each` repeatedly calls the given function and emits the result value.

```plantuml
caption marble diagram for ""generate_each""
left to right direction
rectangle {
usecase anc as " " #transparent;line:transparent
card fn as """generate_each(""\n""  [v=0]() mutable {""\n""    return v+=10;""\n""  })""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o20 as "20"
usecase o30 as "30"
card oE as "|" #transparent
oS -- o10
o10 -- o20
o20 -- o30
o30 --- oE
}
```

## iotas

`iotas` produces a sequence of values from the given first value to the given last value with the given increment applied to each value emitted to get the next value to emit.

```plantuml
caption marble diagram for ""iotas""
left to right direction
rectangle {
usecase anc as " " #transparent;line:transparent
card fn as """iotas(10, 30, 10)""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o20 as "20"
usecase o30 as "30"
card oE as "|" #transparent
oS -- o10
o10 -- o20
o20 -- o30
o30 --- oE
}
```

## fork

`fork` takes values from the input sequence and emits them in parallel on the execution-context provided by the receiver's environment.

```plantuml
caption marble diagram for ""fork""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """fork([last = now + 30ms](sender auto forked){""\n""  return forked | then_each([](int v){""\n""    sleep_until(last);""\n""    return v+1;""\n""  });""\n""})""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o11 as "---\nthrd0: sleeping\n---\n"
usecase o21 as "---\nthrd0: sleeping\n---\nthrd1: sleeping\n---\n"
usecase o31 as "---\nthrd0: 11""      ""\n---\nthrd1: 21""      ""\n---\nthrd2: 31""      ""\n---\n"
card oE as "|" #transparent
oS -- o11
o11 -- o21
o21 -- o31
o31 -- oE
}
```


## merge_each

`merge_each` takes multiple input sequences and merges them into a single output sequence.

```plantuml
caption marble diagram for ""merge_each""
left to right direction
rectangle {
card iS0 as "a:" #transparent;line:transparent 
usecase i10 as "10"
card iE0 as "|" #transparent
card iS1 as "b:" #transparent;line:transparent 
usecase i20 as "20"
card iE1 as "|" #transparent
card iS2 as "c:" #transparent;line:transparent 
usecase i30 as "30"
card iE2 as "|" #transparent
iS0 -- i10
i10 -- iE0
iS1 --- i20
i20 ---- iE1
iS2 ---- i30
i30 -- iE2
usecase anc as " " #transparent;line:transparent
card fn as """merge_each(a, b, c)""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o20 as "20"
usecase o30 as "30"
card oE as "|" #transparent
oS -- o10
o10 -- o20
o20 -- o30
o30 --- oE
}
```

## scan_each

`scan_each` is like a reduce, but emits the state after each change.

```plantuml
caption marble diagram for ""scan_each""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """scan_each(3, [](s, v){return s+v;})""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o13 as "13"
usecase o33 as "33"
usecase o63 as "63"
card oE as "|" #transparent
oS -- o13
o13 -- o33
o33 -- o63
o63 -- oE
}
```

## sample_all

`sample_all` emits the most recent stored copy of the most recent input value at the frequency determined by the given interval.

```plantuml
caption marble diagram for ""sample_all""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
usecase i30 as "30"
usecase i40 as "40"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 -- i30
i30 -- i40
i40 -- iE
usecase anc as " " #transparent;line:transparent
card fn as """sample_all(20ms)""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o20 as "20"
usecase o40 as "40"
card oE as "|" #transparent
oS --- o20
o20 --- o40
o40 -- oE
}
```
## timeout_each

`timeout_each` completes the sequence with a `timeout_error` if any two input values are separated by more than the given interval.

```plantuml
caption marble diagram for ""timeout_each""
left to right direction
rectangle {
card iS as "in:" #transparent;line:transparent 
usecase i10 as "10"
usecase i20 as "20"
card iE as "|" #transparent
iS -- i10
i10 -- i20
i20 --- iE
usecase anc as " " #transparent;line:transparent
card fn as """timeout_each(20ms)""" 
anc -[hidden]-- fn
card oS as "out:" #transparent;line:transparent
usecase o10 as "10"
usecase o20 as "20"
card oE as "X" #transparent
oS -- o10
o10 -- o20
o20 -- oE
}
```
