---
theme: seriph
background: https://images.unsplash.com/photo-1595515770330-ceeea7d82cfd?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=3270&q=80
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  TiCDC Sink Component.

  Learn more at [TiFlow](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2)
drawings:
  persist: false
title: A sink
---

# A Sink

A Deep Dive

Based on TiCDC [v6.5.1](https://github.com/pingcap/tiflow/tree/v6.5.1)

RUSTIN LIU

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Begin <carbon:arrow-right class="inline"/>
  </span>
</div>

---
transition: slide-up
---

# Rustin Liu

<div class="leading-8 opacity-80">
PingCAPer.<br/>
Data Replication Team.<br/>
Cargo Contributor.<br/>
Rustup Maintainer.<br/>
</div>

<div my-10 grid="~ cols-[40px_1fr] gap-y4" items-center justify-center>
  <div i-ri-github-line op50 ma text-xl/>
  <div><a href="https://github.com/hi-rustin" target="_blank">hi-rustin</a></div>
  <div i-ri-twitter-line op50 ma text-xl/>
  <div><a href="https://twitter.com/hi_rustin" target="_blank">hi_rustin</a></div>
  <div i-ri-firefox-line op50 ma text-xl/>
  <div><a href="https://hi-rustin.rs" target="_blank">hi-rustin.rs</a></div>
</div>

<img src="https://avatars.githubusercontent.com/u/29879298?v=4" rounded-full w-40 abs-tr mt-22 mr-22/>

<div flex="~ gap2">
</div>

---
transition: slide-up
layout: center
---

# Sink Refactoring

<br>

### - Phase 1: Introduce a async sink and a new conflict detector.
<br>

### - Phase 2: Change it from push model to pull model.
<br/>

---
transition: slide-up
layout: center
---

<div text-6xl fw100>
  Agenda
</div>

<br>

<div class="grid grid-cols-[3fr_2fr] gap-4">
  <div class="border-l border-gray-400 border-opacity-25 !all:leading-12 !all:list-none my-auto">

  - TiCDC Architecture
  - What is the problem?
  - What we did to solve it?
  - What are my challenges?
  - What I learned?
  - Q&A

  </div>
</div>

---
transition: slide-up
layout: center
---

# Architecture


---
transition: slide-up
layout: center
---

<div class="arch">
<div>

# Architecture

</div>

<div
  class="relation"
>

- A TiCDC cluster has only one owner.
- A capture will have multiple processors.
- A processor can only process one changefeed.
- A changefeed can synchronize multiple tables.

</div>

<div>

```plantuml {scale: 0.8}
@startuml
!theme plain
package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Owner" {
  [Scheduler]
  [DDLSink]
  package "DDL" as DDL1 {
    [OwnerSchemaStorage]
    [OwnerDDLPuller] --> [gRPC]
  }
}

node "Processor" {
  [ProcessorMounter]
  [SinkManager] #FF6655
  package "DDL" as DDL2 {
    [ProcessorSchemaStorage]
    [ProcessorDDLPuller] --> [gRPC]
  }
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] --> [gRPC]
      [Sorter1]
      [TableSink1] #Yellow
    }
    package "Table2 Pipeline" {
      [Puller2] --> [gRPC]
      [Sorter2]
      [TableSink2] #Yellow
    }
  }
}

database "MySQL/Kafka" {
  [MySQL]
  [Broker]
}

[DDLSink] --> [MySQL]
[DDLSink] --> [Broker]
DDL1 --> [DDLSink]
[OwnerSchemaStorage] ..> [OwnerDDLPuller] : use
[SinkManager] --> [MySQL]
[SinkManager] --> [Broker]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> [SinkManager] : use
[TableSink2] ..> [SinkManager] : use
[ProcessorMounter] ..> DDL2 : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.arch {
  display: flex;
}

.arch img {
  margin-top: -80px;
}

.relation {
  position: absolute;
  z-index: 1;
  left: 120px;
  top: 60px;
  font-size: 12px;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

---
transition: slide-up
---

# Table Pipeline

Each changefeed creates a processor, and each processor maintains multiple table pipelines.


### Pipeline
<br>
<br>

```mermaid {scale: 1.5}
flowchart LR
    puller((PullerNode)) --> sorter((SorterNode)) --> mounter((Mounter)) --> sink((SinkNode))
```

---
transition: slide-up
layout: center
---

# What is the problem?

---

<div class="relation">

<div class="title">

# Old Sink Design

</div>
<div class="uml">

```plantuml {scale: 1}
@startuml
!theme plain
package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Processor" {
  [ProcessorMounter]
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] --> [gRPC]
      [Sorter1]
      [TableSink1] #Yellow
    }
    package "Table2 Pipeline" {
      [Puller2] --> [gRPC]
      [Sorter2]
      [TableSink2] #Yellow
    }
    package "Sink Manager" {
      folder "Combination" as Combination {
        [MySQLSink] #FF6655
        [BufferSink] #FF6655
      }
    }
  }
}

database "MySQL"

note right of [MySQLSink]
  It can be either
  MQ Sink or BlackHoleSink.
end note

[MySQLSink] --> [MySQL]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> Combination : use
[TableSink2] ..> Combination : use
Combination .[#FF6655].> [TableSink1] : use
Combination .[#FF6655].> [TableSink2] : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.relation {
  display: flex;
  justify-content: flex-start;
}

.relation img {
  height: 500px;
}

.relation .title {
  flex-grow: 4;
}

.relation .uml {
  flex-grow: 2;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>


---
transition: slide-up
---

<div class="arch">
<div>

# Old Sink Module Abstract

</div>

<div>
  <br/>
  <br/>
  <br/>

```plantuml
@startuml
!theme plain
folder "Sink Manager" as SM {
  class BS as "Buffer Sink" #FF6655
  folder "MySQL Sink" as MS {
    class DML as "DML Sink" #FF6655
    class DDL as "DDL Sink" #FF6655
  }
}
class SN as "Sink Node" #Yellow
class TS as "Table Sink" #Yellow

SN *-- TS : use
SM *-[#FF6655]- TS : manage
TS *-[#FF6655]- SM : use

note left of TS
  Table Sink Managed by Sink Manager
  and Table Sink also use Sink Manager.
  ⚠️This is a circular dependency.
end note

note right of MS
  MySQL Sink is a combination of
  DML Sink and DDL Sink.
  ⚠️However, only one of these is used per module.
end note
@enduml
```

</div>
</div>

<style>
.arch {
  display: flex;
}

.arch img {
  margin-top: -80px;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

---
transition: slide-up
---

# Old Data Sequence (Sync)

```plantuml {scale: 0.8}
@startuml
!theme plain
SinkNode <[bold,#FF6655]- SinkNode: ⚠️added to buffer
SinkNode -> TableSink: buffer is full and SinkNode calls EmitRowChangedEvents
SinkNode <-[bold,#FF6655]- TableSink: ⚠️added to buffer
SinkNode -> TableSink: calls FlushRowChangedEvents
TableSink -> "SinkManager": calls EmitRowChangedEvents
TableSink <-[bold,#FF6655]- "SinkManager": ⚠️added to buffer
TableSink -> "SinkManager": calls FlushRowChangedEvents
TableSink <-[bold,#FF6655]- "SinkManager": ⚠️added to ResolvedTs Buffer
SinkNode <-- TableSink: added to buffer sink
loop BufferSink
  "SinkManager" -> "SinkManager)": BufferSink buffer is full
  "SinkManager" -> MySQLSink: calls EmitRowChangedEvents \nand FlushRowChangedEvents of MySQLSink
end
MySQLSink -> MySQL: Execute SQL
MySQLSink <-- MySQL: SQL Result


note left of SinkNode #FF6655
  Buffer One.
end note

note left of TableSink #FF6655
  Buffer Two.
end note

note left of "SinkManager" #FF6655
  Buffer Three and Four.
end note

@enduml
```

---
transition: slide-up
---

# Old Sink Problems

- Circular dependency
- Too many buffers, too much memory usage
- Leaks table information everywhere
- Call stack is too deep and all calls are synchronous
- Abstraction is very complicated and there are too many functions in the interface
- Some implementations are not thread-safe, but they should be

---
transition: slide-up
layout: center
---

# What we did to solve it?


Introduce a async sink and a new conflict detector.

---
transition: slide-up
layout: center
---

<div class="relation">

<div class="title">

# New Sink Design

</div>
<div class="uml">

```plantuml {scale: 1}
@startuml
!theme plain
package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Processor" {
  [ProcessorMounter]
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] --> [gRPC]
      [Sorter1]
      [TableSink1] #Yellow
    }
    package "Table2 Pipeline" {
      [Puller2] --> [gRPC]
      [Sorter2]
      [TableSink2] #Yellow
    }
    package "Event Sink" {
      [Worker1] #FF6655
      [Worker2] #FF6655
    }
  }
}

database "MySQL"

note right of [Event Sink]
  It can be
  MySQL Sink、MQ Sink、Cloud Storage Sink or BlackHoleSink.
end note

[Worker1] --> [MySQL]
[Worker2] --> [MySQL]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> [Event Sink] : use
[TableSink2] ..> [Event Sink] : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.relation {
  display: flex;
  justify-content: flex-start;
}

.relation img {
  height: 500px;
}

.relation .title {
  flex-grow: 4;
}

.relation .uml {
  flex-grow: 2;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>


---
transition: slide-up
---

# New Sink Module Abstract

<br/>
<br/>

```plantuml
@startuml
!theme plain
class TS as "Table Sink" #Yellow
interface ES as "Event Sink" #FF6655
class MQS as "MQ Event Sink"
class TXNS as "Transaction Event Sink"
class CSS as "Cloud Storage Event Sink"

TS *-- ES : use

ES <|.. MQS : implement
ES <|.. TXNS : implement
ES <|.. CSS : implement

interface DDLS as "DDL Sink"
class DDLMQS as "MQ DDL Sink"
class DDLTXNS as "Transaction DDL Sink"
class DDLCSS as "Cloud Storage DDL Sink"

DDLS <|.. DDLMQS : implement
DDLS <|.. DDLTXNS : implement
DDLS <|.. DDLCSS : implement
@enduml
```

<br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Table Sink](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2/tablesink)
· [Event Sink](https://github.com/pingcap/tiflow/blob/v6.5.1/cdc/sinkv2/eventsink/event_sink.go) · [DDL Sink](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2/ddlsink) · [MQ Event Sink](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2/eventsink/mq) · [Transaction Event Sink](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2/eventsink/txn) · [Cloud Storage Event Sink](https://github.com/pingcap/tiflow/tree/v6.5.1/cdc/sinkv2/eventsink/cloudstorage)


---
transition: slide-up
---

# Table Sink

## Abstract

```go{all|6|10|14}
// It is used to sink data in table units.
type TableSink interface {
	// AppendRowChangedEvents appends row changed events to the table sink.
	// Usually, it is used to cache the row changed events into table sink.
	// This is a not thread-safe method. Please do not call it concurrently.
	AppendRowChangedEvents(rows ...*model.RowChangedEvent)
	// UpdateResolvedTs writes the buffered row changed events to the eventTableSink.
	// Note: This is an asynchronous and not thread-safe method.
	// Please do not call it concurrently.
	UpdateResolvedTs(resolvedTs model.ResolvedTs) error
	// GetCheckpointTs returns the current checkpoint ts of table sink.
	// For example, calculating the current progress from the statistics of the table sink.
	// This is a thread-safe method.
	GetCheckpointTs() model.ResolvedTs
	// We should make sure this method is cancellable.
	Close(ctx context.Context)
}
```

---
transition: slide-up
---

# Table Sink

## Implementation


```go{all|2|8-9|4|6|5}
// EventTableSink is a table sink that can write events.
type EventTableSink[E eventsink.TableEvent] struct {
	...
	maxResolvedTs   model.ResolvedTs
	backendSink     eventsink.EventSink[E]
	progressTracker *progressTracker
	...
	// NOTICE: It is ordered by commitTs.
	eventBuffer []E
	state       state.TableSinkState
	...
}
```

```go{0|all}
type EventSink[E TableEvent] interface {
	// WriteEvents writes events to the sink.
	// This is an asynchronously and thread-safe method.
	WriteEvents(events ...*CallbackableEvent[E]) error
	Close() error
}
```

---
transition: slide-up
---

<div class="arch">
<div>

# Row Change Data Sequence

</div>

<div>
<br/>
<br/>

```plantuml {scale: 0.9}
@startuml
!theme plain
participant SN as "Sink Node"
participant TS as "Table Sink"
participant PT as "Progress Tracker"
participant TXNS as "Transaction Event Sink"
participant MW as "MySQL Worker"
participant M as "MySQL Server"

group #lightyellow Add Event
SN -> TS: call AppendRowChangedEvents
SN <- TS: added to buffer
end

group #lightyellow Update ResolvedTs
SN -> TS: call UpdateResolvedTs
TS -> PT: call AddEvent
TS <- PT: return a callback
TS -> PT: call AddResolvedTs
TS <- PT: added to records
TS -> TXNS: call WriteEvents with callback
TS <- TXNS: added to Conflicts Detector
SN <- TS: updated resolvedTs
end
group #lightyellow Async Write Events
TXNS -> MW: Dispatch Txn Events (Conflict detection)
MW -> M: Execute SQL
M -> MW: Execute SQL Result
MW --> PT: call Callback
end

group #lightyellow Get CheckpointTs
SN -> TS: call GetCheckpointTs
TS -> PT: call advance
TS <- PT: return checkpointTs
end

note left TS #green
  Only one buffer.
end note
@enduml
```

</div>
</div>

<style>
.arch {
  display: flex;
}

.arch img {
  margin-top: -80px;
}

.relation {
  position: absolute;
  z-index: 1;
  left: 120px;
  top: 60px;
  font-size: 12px;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

---
transition: slide-up
---

# Table Sink

## Progress Tracker

Example: txn1, txn2, resolvedTs2, txn3-1, txn3-2, resolvedTs3, resolvedTs4, resolvedTs5

### How to track the progress?

<v-click>
```plantuml {scale: 0.9}
@startjson
#highlight "records" / "0" / "value"
#highlight "records" / "1" / "value"
{
  "records": [
    {
      "key": "event1",
      "value": "nil"
    },
    {
      "key": "event2",
      "value": "resolvedTs1"
    },
    {
      "key": "event3",
      "value": "nil"
    },
    {
      "key": "event4",
      "value": "nil"
    },
    {
      "key": "event5",
      "value": "resolvedTs2"
    }
  ]
}
@endjson
```

</v-click>


---
transition: slide-up
---

# Table Sink

## Progress Tracker

<br/>

<v-click>

### What about the performance?

</v-click>

<v-click>

It is not good enough because we need get `lock()` to update the progress.

</v-click>

<v-click>

### How can we make it faster?

</v-click>

<br/>
<v-click>

##### BitMap

```go{all|6}
// Set the corresponding bit to 1.
// For example, if the eventID is 3, the bit is 3 % 64 = 3.
// 0000000000000000000000000000000000000000000000000000000000000000 ->
// 0000000000000000000000000000000000000000000000000000000000001000
// When we advance the progress, we can try to find the first 0 bit to indicate the progress.
postEventFlush = func() { atomic.AddUint64(&lastBuffer[len(lastBuffer)-1], 1<<bit) }
```

</v-click>


---
transition: slide-up
layout: center
---

# MySQL Sink

---
transition: slide-up
---

# MySQL Sink Data Sequence (Async)

## Row Change Data Sequence

<br/>

```plantuml
@startuml
!theme plain
participant TS1 as "Table Sink1"
participant TS2 as "Table Sink2"
participant TXNS as "Transaction Event Sink"
participant MW as "MySQL Worker"
participant M as "MySQL Server"

TS1 -> TXNS: call WriteEvents
TS2 -> TXNS: call WriteEvents
TXNS -[bold,#FF6655]> MW: Dispatch Txn Events(Conflict detection)
MW -> M: Execute SQL
M --> MW: Execute SQL Result
MW --> TS1: call Callback
MW --> TS2: call Callback

note left of TS1 #FF6655
  Only one buffer.
end note

@enduml
```

---
transition: slide-up
---

# MySQL Sink

## Old Conflict Detection - Union Set

```sql{all|1|2|3|4|5|6}
DML1: INSERT INTO t VALUES (1, 2);
DML2: INSERT INTO t VALUES (2, 3);
DML3: UPDATE t SET pk = 4, uk = 3 WHERE pk = 2;
DML4: DELETE FROM t WHERE pk = 1;
DML5: REPLACE INTO t VALUES (1, 3);
DML6: INSERT INTO t VALUES (5, 6);
```

```plantuml
@startuml
!theme plain

node "PK:1" as PK1 #pink
node "UK:2" as UK1 #pink
node "Group 1" as G1 #736060
node "PK:1" as PK4 #d4b4b4
node "UK:2" as UK4 #d4b4b4

PK1 --> G1
UK1 --> G1
PK4 -up-> G1
UK4 -up-> G1

node "PK:2" as PK2 #yellow
node "UK:3" as UK2 #yellow
node "PK:2(Old)" as PK32 #lightyellow
node "UK:3(Old)" as UK32 #lightyellow
node "PK:4(New)" as PK31 #lightyellow
node "UK:3(New)" as UK31 #lightyellow
node "Group 2" as G2 #736060
PK2 --> G2
UK2 --> G2
PK31 -up-> G2
UK31 -up-> G2
PK32 -up-> G2
UK32 -up-> G2

node "PK:1" as PK5 #red
node "UK:3" as UK5 #red

PK5 -right-> G1
UK5 -left-> G2

node "PK:5" as PK6 #green
node "UK:6" as UK6 #green
node "Group 3" as G3 #736060
PK6 --> G3
UK6 --> G3
@enduml
```


---
transition: slide-up
---

# MySQL Sink

## New Conflict Detection - DAG (Directed Acyclic Graph)

- Node: Transaction received by Conflict Detector, that has not been executed.
- Edge: T2 -> T1, T1 exists one edge to T2, only if T1 modifies the same key as T2.
<br/>

> We can ignore T2 -> T1, if there exists one path T2 -> Ta -> Tb ... -> Tx -> T1.


```sql{all|1|2|3|4|5|6}
DML1: INSERT INTO t VALUES (1, 2);
DML2: INSERT INTO t VALUES (2, 3);
DML3: UPDATE t SET pk = 4, uk = 3 WHERE pk = 2;
DML4: DELETE FROM t WHERE pk = 1;
DML5: REPLACE INTO t VALUES (1, 3);
DML6: INSERT INTO t VALUES (5, 6);
```

```plantuml
@startuml
!theme plain

node "T1(PK:1, UK:2)" as T1 #pink
node "T2(PK:2, UK:3)" as T2 #yellow
node "T3(old: PK:2, UK:3, new: PK:4, UK:3)" as T3 #lightyellow
node "T4(PK:1, UK:2)" as T4 #d4b4b4
node "T5(PK:1, UK:3)" as T5 #red

T3 -right-> T2
T4 -right-> T1
T5 -right-> T4
T5 --> T3

node "T6(PK:5, UK:6)" #green
@enduml
```

---
transition: slide-up
---

# Outcomes

- Better performance: Union Set -> DAG. (With pull-based-sink: 63.3k/s -> 103k/s)
- Removes some buffers.
- Better abstraction and data flow: Table Sink -> Transaction Event Sink -> MySQL.
- Better testability.
- Better maintainability: all functions are thread-safe and async in Event Sink.
- Easy to add new sinks.

---
layout: center
---

# Q&A

<br/>
<br/>

## Do you have any questions?

---
layout: center
class: text-center
---

# Learn More

[Documentations](https://docs.pingcap.com/tidb/dev/ticdc-overview) · [GitHub](https://github.com/pingcap/tiflow)  · [How to write a new sink](https://hi-rustin.rs/TiCDC-Sink-%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97/)