---
layout: post
title: "plantuml-baskc-usage"
date: 2016-09-07 11:12:58 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

# 基本框架
```
@startuml

'comment

/' multiline
comments
'/

title First Page Title
' some diagram

newpage

' some other diagram

' legend 图注 left/center/right
legend center
XX Diagram
endlegend

' note left/left of/right/right of/over Alice
note left: First note

note right
Second note
end note

' == Divider ==

/' space
|||
||45|| == 45px
'/

@enduml
```

# 类图

```
class Parent
class Child
Parent <|-- Child

' 同生同死
class CompositeChild
CompositeChild : Parent strongRef
Parent *-- CompositeChild

' 不同生同死
class AggregateChild {
Parent gettWeakParent()
{static} getStaticField()
{abstract} doAbstractMath()
}
Parent o-- AggregateChild

abstract class "Pure Abstract Class within parens"
interface IInterface
class GenericClass<? extends G>

package "My Package" {
IInterface ()- Parent
}
```

# 时序图

```
A -> B: -> solid
B --> A: --> dash
A ->x B: ->x
A ->> B: ->>
A \[#blue]- B: \[#blue]-
autonumber
A -> B: autonumber1, ||| == space
|||
autonumber 10
A -> B: autonumber 10, ... == delay
... Delayed ...
autonumber 20 5
A -> B: autonumber 20 5-1
A -> B: autonumber 20 5-2
A -> B: autonumber stop == (should work)
A -> B: activate B
activate B
B -> A: deactivate B
deactivate B
A -> C: activate C
activate C
A -> C: destroy C
destroy C

create D
A -> D: create D

[-> A: [ Incoming
A ->]: ] Outgoing

participant E << Stereotype >>
A -> E: stereotype
participant F << (S,#ADD1B2) Stereotype >>
A -> F: Stereotype with colored Spot

box "box #LightBlue" #LightBlue
participant E
participant F
end box
E -> F: box encompass

actor Actor1
boundary Boundary1
control Control1
entity Entity1
database Database1

participant "Long name #red" as L #red
L -> L: recursion
```