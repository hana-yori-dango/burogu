+++
title = "Event Sourcing"
weight = 1
order = 1
date = 2023-04-04
insert_anchor_links = "right"
[taxonomies]
tags = ["event-sourcing", "definition", "tldr"]
+++

Event sourcing is an approach to software development where every action that changes the state of a system is captured as an immutable event and stored in an event log. By replaying these events, the system's state can be reconstructed at any point in time, providing a complete audit trail of all changes and enabling advanced features.

<!-- more -->

<img src="https://cluzeau.pro/event-sourcing-example-e-commerce.jpg" alt= "example of e-commerce events" width="75%" height="75%"/>

*Example of a series of events for an e-commerce application*

## Longer definition

Event sourcing is an approach to software development where the state of a system is determined by a sequence of events. In event sourcing, every action that changes the state of the system is captured as an event and stored in an event log. These events are immutable, meaning that they cannot be changed once they have been recorded.

By replaying the events in the event log, it is possible to reconstruct the state of the system at any point in time. This approach allows for a complete audit trail of all changes to the system, as well as the ability to rebuild the state of the system if it becomes corrupted or lost.

Event sourcing can also be used to enable a number of advanced features, such as temporal queries that allow users to query the state of the system at a specific point in time, and event-driven architectures where different parts of the system can react to events in real time.

Overall, event sourcing is a powerful approach to software development that provides a complete audit trail of all changes to the system, enables advanced features, and allows for easy recovery from system failures.

## Links

- [wikipedia.org/event sourcing](https://en.wikipedia.org/wiki/Event-driven_architecture)
- [martinfowler.com/eaaDev/EventSourcing](https://martinfowler.com/eaaDev/EventSourcing.html)

## Acknowledgment

Definitions made using [chat.openai.com/chat](https://chat.openai.com/chat) with the following prompts:

> Summarize Event Sourcing
>
> more concise
