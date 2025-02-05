---
title: Open 3D Engine Multiplayer Gem Overview
description:
linkTitle: Overview
toc: true
---

In **Open 3D Engine (O3DE)**, the synchronization needed for multiplayer support is performed through *multiplayer components*. Like other components in O3DE, these are attached to *entities*. Multiplayer components offer the capability to receive function calls from a remote source, *remote procedure calls (RPC)*, and define the data to be synchronized across the network. Each multiplayer component, normally generated as an [autocomponent](./autocomponents), defines its own requirements for other data to be present and which functions are available.

Some of the features offered in the O3DE Multiplayer Gem include:

* Role-based synchronization and replication rules
* Local prediction and backwards reconciliation ("rollback")
* Server-authoritative asynchronous multiplayer

## Multiplayer components

O3DE multiplayer components are built on top of standard O3DE components, offering the following data and functionality for network communication:

* **Network properties**, state information about the component that gets replicated between hosts.
* **Remote procedure calls (RPCs)**, allow messages to be sent to a particular component on an entity.

Together, these represent an event-driven messaging system. When a property changes on a network component, it triggers an event; Similarly, when an RPC is invoked, it sends a trigger to the relevant component. This means that the multiplayer Gem uses a *push-based* model rather than *poll-based* model for synchronizing information across the clients and server. Using this model keeps code and entity management simpler and offers performance benefits by not requiring a network state check on a timer.

{{< important >}}
Network components are the **only** built-in way to communicate between entities in O3DE. Due to the engine's ability to distribute and interact with entities across multiple hosts, there is no guarantee that any two entities have authoritative representations on the same host.
{{< /important >}}

## Network properties

The O3DE Multiplayer Gem uses *delta replication* to transmit property changes across hosts. The delta replication method uses a system where a server maintains information about both the last acknowledged state of each client and the current world state, and then sends the differences between the two states across the network. When the client receives this information, it updates the world state if the packet was received in the correct sequence.

One important consideration of this style of replication is that it requires the client to send an acknowledgement (`ACK`) message back to the server. The O3DE AzNetworking framework offers reliable UDP packets, including acknowledgement.

Without taking advanced networking features into consideration, a server checks for a delta on every frame by:

1. Inspecting a snapshot of the world state against each client's acknowledged state.
1. If the world state is different from the last state that resulted in an `ACK`, generate a delta for that client.
1. Transmit the delta to the client.

Then, clients update themselves from a delta by:

1. Checking if the received delta is in sequence.
1. Extracting the delta information and updating the world state.

After, the AzNetworking layer becomes aware that the world was updated. This occurs automatically through ACK vector replication, and does not require any further action from the client.

## Remote procedure calls (RPCs)

Remote procedure calls are how O3DE allows for pushing messages between hosts, rather than sending an update that's based on the server world state. O3DE offers both *reliable* and *unreliable* RPCs. By default, RPCs are reliable.

**Reliable** RPCs use a queue to guarantee the delivery of a message. When sending any reliable packet, the packet is also inserted into a priority queue and given a timeout value that's related to the latency of the connection that the message is sent on. On timeout, if the reliable packet was not explicitly acknowledged, the packet will be retransmitted. On the receiving end, the O3DE client tracks every reliable packet received and guarantees that any packet will only be delivered at most once. While this feature provides *guaranteed* delivery, it doesn't provide *ordered* delivery.

**Unreliable** RPCs are sent over a "fire and forget" method. The host sending the message has no way to ensure that the message was received.

## Multiplayer entity roles

Because each entity with a multiplayer component is replicated across the network, you can think of them as existing on more than one host simultaneously. This consideration introduces the problem of *authority* over the network - which hosts have precedence to set the network state, define the replication rules, and determine which roles a host can play. In O3DE, these roles are intentionally compile-time enforced. The Multiplayer Gem assigns and handles the roles of multiplayer entities automatically. 

The roles offered for O3DE multiplayer hosts are:

* **Client** (`NetEntityRole::Client`): The lowest privilege role for a component. The smallest possible subset of network properties are replicated to this role, and its behavior is strictly read-only. This role is used on client entities that are controlled by a host, and should contain only presentation logic and act as a proxy for invoking RPCs. Examples of entities that should use this role are AIs and other players moving in the world.

* **Autonomous** (`NetEntityRole::Autonomous`): A role with the _illusion_ of write access. Autonomous roles are usually assigned to components directly under local user control. These roles receive a larger amount of network information than a Client role. Autonomous roles can also take advantage of predictive networking.

* **Authority** (`NetEntityRole::Authority`): The role with ultimate authority. It has full read and write access to all network properties on the component.

In addition to the previously described roles, O3DE has an additional role found only in multiserver instances:

* **Server** (`NetEntityRole::Server`): Provided to servers that *don't* have authority over a particular entity. This role is strictly read-only, and all interaction with these entities should be handled using RPCs.

## Related topics

| Topic | Description |
|--|--|
| [Networking](/docs/user-guide/networking/) | Learn about `AzNetworking`, the core networking framework that O3DE uses, and that the Multiplayer Gem is built on top of. |
| [Automate Source Generation from Templates with AzAutoGen](/docs/user-guide/engine/autogen/) | Learn about the AzAutoGen code generation system used to create [auto-components](./autocomponents) for projects using the Multiplayer Gem. |
| [Multiplayer Gem API Reference](/docs/api/gems/multiplayer/) | The C++ API reference for classes that the O3DE Multiplayer Gem provides. |