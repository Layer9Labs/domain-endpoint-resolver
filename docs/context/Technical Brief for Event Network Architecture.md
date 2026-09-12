# Technical Brief: Event Network Architecture (ENA)

# Table of Contents

[1. Executive Summary 4](#executive-summary)

[1. Overview 4](#overview)

[2. Strategic Value Proposition 4](#strategic-value-proposition)

[3. Key Operational Capabilities 4](#key-operational-capabilities)

[2. High-Level Capabilities & Architecture 5](#high-level-capabilities-architecture)

[4. Core Capabilities 8](#core-capabilities)

[1. A Unified Fixed Event Structure: 8](#a-unified-fixed-event-structure)

[2. Hardware-Agnostic Execution: 9](#hardware-agnostic-execution)

[3. Decentralized Point-to-Point Event Network: 9](#decentralized-point-to-point-event-network)

[4. Processing at the Source 9](#processing-at-the-source)

[5. Data-Defined Topology 9](#data-defined-topology)

[5. The Simpler Cognitive Model 9](#the-simpler-cognitive-model)

[6. Semantic Map 10](#semantic-map)

[3. The Data Model: Domains, Schemas, and Events 12](#the-data-model-domains-schemas-and-events)

[1. Domains and Schemas 12](#domains-and-schemas)

[2. The Fixed Event Structure 13](#the-fixed-event-structure)

[3. Granularity vs. DTOs 15](#granularity-vs.-dtos)

[4. Contextual Metadata 16](#contextual-metadata)

[4. The Event Router and Runtime Environment 17](#the-event-router-and-runtime-environment)

[1. Router Source Language 17](#router-source-language)

[2. Compilation and Bytecode 18](#compilation-and-bytecode)

[3. The Micro Virtual Machine 18](#the-micro-virtual-machine)

[5. Logic and Processing Rules 19](#logic-and-processing-rules)

[1. Pipeline Components 19](#pipeline-components)

[2. Rulesets 19](#rulesets)

[1. Drop / Filter Rule 21](#drop-filter-rule)

[2. Forward Rule 22](#forward-rule)

[3. Accept Rule 22](#accept-rule)

[6. Operational Modes: Efficiency and Latency 22](#operational-modes-efficiency-and-latency)

[7. Time-Biased vs. Bandwidth-Biased 23](#time-biased-vs.-bandwidth-biased)

[8. Event Transmission Strategies 23](#event-transmission-strategies)

[9. Network Topology and Discovery 25](#network-topology-and-discovery)

[10. Domain Endpoint Resolver (DER) 25](#domain-endpoint-resolver-der)

[11. Event Network and Sidecars 26](#event-network-and-sidecars)

[12. Dynamic Discovery 27](#dynamic-discovery)

[13. Resiliency, Redundancy, and Security 27](#resiliency-redundancy-and-security)

[14. Routing Algorithms 28](#routing-algorithms)

# Executive Summary

## Overview

The global digital economy is undergoing a profound structural transformation, migrating from centralized cloud computing models toward decentralized, autonomous edge processing and intelligence. As the volume of data generated at the network edge outpaces the bandwidth and latency capabilities of traditional transport infrastructures, organizations across every vertical face a systemic "data gravity" crisis. In response to this architectural bottleneck, Event Network Architecture (ENA) presents a foundational capability that cleanly decouples logical event processing from physical hardware constraints. Event Network Architecture (ENA) is a proprietary patent-pending (US-20240129230-A1) technology, developed by Layer 9 Labs LLC, designed to enable the shift from a centralized bias to an autonomous edge compute bias that will enable processing of data at the Edge, and enable Edge AI.

## Strategic Value Proposition

Current digital ecosystems often require distinct skill sets and tool chains for embedded devices versus cloud distributed systems, creating a "significant tax" on solution development. ENA addresses this by introducing a simpler cognitive model. Developers use a single Router Source Language to define logic that is compiled into hardware-agnostic bytecode. This binary interacts with a deterministic micro virtual machine. Making the same bytecode file portable on a low-power sensor or a high-performance server, significantly reducing development time and operational complexity.

## Key Operational Capabilities

**Unified Interoperability**: ENA replaces variable Data Transfer Objects (DTOs) with a fixed event message structure of (Domain, Entity, Attribute, Value, Timestamp). This consistency eliminates the need for translation layers or "glue code," making event sources and sinks composable and interchangeable. Unlike traditional DTOs, this granular approach preserves data veracity by ensuring every attribute change retains its specific timestamp, even during high-volume batching. This interoperability transcends organizational boundaries allowing other actors to participate in data coordination.

**Edge Intelligence and Efficiency**: ENA prioritizes decision-making at the source. Event Routers can filter, transform, and route data immediately upon receipt on an incoming event. This capability supports bandwidth-biased processing, where data can be batched and de-duplicated to maximize network efficiency, or time-biased processing for real-time critical operations. It is also possible to create localized closed decision-loops without the need for externalizing the decision-making loop.

**Decentralized Event Networking**: ENA uses a distributed network of lightweight processing nodes, avoiding the high costs and computing demands of a central broker. The nodes that run at the edge or in the cloud are registered through dynamic endpoint discovery and mesh sidecar, enabling scalable and resilient point-to-point event handling.

**Security and Resilience:** ENA is built for enterprise requirements, supporting encrypted transport (X.509 certificates) and signed binary files to prevent tampering. It offers flexible routing algorithms, including round-robin, lowest latency, and hierarchical failover, ensuring that data reliably reaches its destination even if primary endpoints fail.

# High-Level Capabilities & Architecture

The scope of the Event Network Architecture (ENA) is to establish a network for event data. Each node in the network is either a data domain server that will only accept event messages for the domain(s) it is configured for, or it is an event source that sends event messages to one or more domain servers(s). The ENA establishes a point-to-point data network topology as an overlay on a physical network, to enable event processing seamlessly across the spectrum of Microcontrollers, Single Board Computers, Edge Compute, and Cloud Compute. Event messages within the network may be filtered, transformed, or routed to domain endpoints within the event network based on the contents of the event.

In the event network, there are event sources that produce events, events sinks, that consume events, and combination of the two and, a process that consumes events, processes them, and then emits them, encompassing the behavior of both a sink and a source. The Event Router is one such process that consumes and produces events.

The event network architecture consists of the following core components:

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image1.png)

Figure 1. Event Network Architecture Components

1.  > **A Fixed Event Structure**: Communication within the event network is based on a granular fixed event structure. This is to encourage interoperability with components within the event network, also to remove data translation layers that permeate traditional approaches.

2.  > **Domain**: A domain is a named set of attributes, where each attribute is of a specific date type. Below is an example of a domain definition in the router source language:

```
domain FoodStar.Kitchen.Floor.Poultry.Oven1@V1 {
    schema TemperatureProbe {
       attributes [
           float   Battery_Percent;
           float   External_Temp;
           float   Internal_Temp;
           float   RSSI "Signal Strength";
           string  Device_MAC;
           string  FW_Version;
       ]
   }
}
```

3.  > **Router Runtime Source Language**: A domain specific language used to configure and define the behavior of an Event Router. This configuration specification is compiled by the Router Source Language Compiler into a binary bytecode file.

4.  > **Event Sources**: Event sources are processes that generate event notifications when a change of state in a system is detected. In the case of an Event Network Architecture event source, the event source process will generate an event message and send it using the fixed event message structure. An event source could be a process that receives input from sensors, clicks from a mouse, that then generates an event that is sent to an event sink.

5.  > **Event Sinks:** An event sink is a process that receives a fixed event structure message from an event source. The flow of events from event sources to event sinks is unidirectional. An event source will send a message to an event sink for a specific domain or multiple domains. The sink’s domain endpoints can be statically defined or can be dynamically created and discovered. A standalone source can be considered as an event injection / ingress point for integration into the ENA. Every event source is a computational node in the ENA. An event sink could be a process that when it receives an incoming event with the appropriate values, it could turn an actuator on or off. This implies that an event sink is listening to an IP/Port for incoming event messages using a specified transport protocol. In ENA, an event sink is configured to ‘listen’ to a specified set of domains, meaning that if a message that is sent to a sink for a domain that it is not configured for, it will ignore that message. In addition, if it receives a message for a domain that it is listening to, and the domain attribute specified in the incoming message does not match an attribute in the domain specification, then that incoming event message will also be ignored.

> An event sink for the domain definition above, will only process messages for the FoodStar.Kitchen.Floor.Poultry.Oven1@V1 domain, and for attributes defined in the schema for this domain. An event sink can be configured for multiple domains.

The listeners section in the router source language is where the event sink is configured to accept event messages and the association between a domain name and IP/Port is made. Below is an example of a listener section:

```
listeners {
    grpc interface LocalListen {
        bind "layer9labs.local:5000";
        cert "file:///C:/.ssh/Certs/layer9labs.local.pfx";
    }
    listen FoodStar.Kitchen.Floor.Poultry.Oven1@V1  [LocalListen]
}
```

The example above specifies the protocol, IP/Port and a certificate name and location. The listen clause then binds the domain name with the interface name.

The bind address can be statically hardcoded with an IP address and Port, or it can use the Domain Endpoint Resolver (DER) or DNS resolution to resolve the IP address. In addition, the bind string can be templatized to accent any information from an environment variable.

```
listeners {
    grpc interface LocalListen {
        bind $"{IP_ADDRESS:PORT}";
        cert "file:///C:/.ssh/Certs/layer9labs.local.pfx";
    }
    listen FoodStar.Kitchen.Floor.Poultry.Oven1@V1 [LocalListen];
}
```
> 
> In the example above, the IP\_ADDRESS and PORT environment variables will need to be declared before the execution of the event sink process, otherwise the process will terminate with an error. This ability to templatize values is essential for a dynamic environment where another process may declare these values at runtime and then launch the event sink process.

6.  > **Event Router**: An event router is an event server that can receive incoming event messages, evaluate the contents of the incoming event data, then either filter, transform or route events to other domain endpoints. It is both an event sink as well as an event source. It can act as an event data switch, router or firewall. It consumes an incoming event in the fixed event structure and will create or route other event messages in the same fixed event message format. The behavior and configuration of the event router is defined by a Router Source Language definition file that has been compiled into a binary bytecode representation.
    
    In addition to a ‘listeners’ section in the router source language, the event router also has a ‘targets’ section in the router source language. Event targets (sinks) can be statically declared in the targets section of the router source language. The list of endpoints in this section ties the target domain and its IP/Port so that the event router can emit to that endpoint for rules that emit to that domain.

```
    targets {
        grpc endpoint ParquetBroadcast {
            address $"{L9L_TARGET}:5004";
            mode binary_stream;
        }

    target FoodStar.Kitchen.Floor.Poultry.Oven1@V1 {
        bc [ParquetBroadcast];
    }
```

Given these core components and utility sinks, arbitrary complex event data networks can be created to ensure that event data is routed to appropriate endpoints. This is depicted in the following illustration.

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image3.png)

Figure 2. Depiction of an ENA Network

## Core Capabilities

### A Unified Fixed Event Structure:

> Unlike systems that use variable Data Transfer Objects (DTOs), ENA enforces a fixed event message structure for all interactions within the network. Every event is transmitted as a "quintuple" consisting of Domain, Entity, Attribute, Value, and Timestamp.

  - **Interoperability**: This structure ensures that event sources and sinks are composable and interchangeable without needing translation layers or "glue code".

  - **Granularity**: Events are emitted at the attribute level (e.g., a single temperature change) rather than as aggregate records, ensuring precise time-series accuracy.

### Hardware-Agnostic Execution:

ENA employs a "write once, deploy anywhere" operational model.

  - **Router Source Language**: Developers define logic, schemas, and rules in a high-level domain-specific language.

  - **Bytecode and Micro VM**: This source code is compiled into a binary bytecode representation that is hardware architecture agnostic. A deterministic micro virtual machine executes this bytecode, allowing the exact same rule file to run on a tiny microcontroller or a massive cloud server.

### Decentralized Point-to-Point Event Network:

> ENA moves away from central brokers and establishes a peer-to-peer event network. The peer definition is either statically defined or leverages the Domain Endpoint Resolver to enable dynamic event node discovery.

  - **Decoupling**: When leveraging the Domain Endpoint Resolver mesh, event producers (sources) are decoupled from consumers (sinks).

  - **Dynamic Discovery**: Nodes use a distributed routing table to dynamically discover endpoints and update routing tables automatically, allowing the network to heal and scale without manual intervention.

### Processing at the Source

A core principle is "decision making closest to the source of data".

  - **Inline Logic**: Event Routers can filter, transform, or redirect data immediately upon receipt of an event. Filtering events may help in bandwidth-constrained environments, where not all events need to be emitted to event sinks. Transformations allow for translation of events between domains to provide improved data semantics. The filtering and transformations happen inline, rather than the need for external processes to subscribe on one topic and republish on another via a central broker– a common pattern in EDAs.

  - **Efficiency**: By filtering out irrelevant data or transforming values (e.g., converting Celsius to Fahrenheit or raw numbers to status strings) at the source. ENA minimizes the need to exfiltrate raw data to compute-intensive cloud environments, thereby optimizing bandwidth and reducing latency.

### Data-Defined Topology

> With ENA, the data topology becomes the network topology. Domains act as proxies for topics, and the network paths are constrained by the data paths defined in the schemas and rulesets. This creates a "transparent data network topology" where the flow of information dictates the shape of the event network. In ENA it is not the case that every node will communicate with every other node. Technically they could, but the communication paths within the point-to-point network is defined by the producers and consumers of data. Not all data produced will be consumed by all other nodes. An event source cannot consume an event, therefore cannot be communicated to.

## The Simpler Cognitive Model

Delivering solutions across microcontrollers, single-board computers, edge, and cloud platforms presents considerable complexity. Each environment demands a distinct set of knowledge, skills, trade-offs, and tool chains. Microcontroller development requires expertise in hardware and proficiency with languages such as Rust, Zig, C, or C++, each environment with varying drivers and tool chains. Edge and cloud implementations necessitate familiarity with native cloud architectures and advanced understanding of distributed systems design, implementation, and maintenance. Successfully managing solutions that span these domains impose significant technical requirements and operational challenges. Managing multiple clusters at the Edge and Cloud requires significant investments to make it operational, at scale.

While ENA is not a comprehensive solution for all necessary knowledge and skills, it offers a streamlined cognitive model. An event sink or event source can be deployed on a microcontroller, that can make localized decisions via a ruleset definition and then emit an event to an endpoint that could be located anywhere in the event network.

This approach enables functionality to be developed on a PC or laptop using a consistent language, which can then be compiled and deployed across various environments. By doing so, it considerably shorten the development cycle from event extraction to message delivery, regardless of extensive device programming expertise. Events can be emitted to endpoints without requiring an understanding of the specific design, implementation, or maintenance details of those endpoints, like a Kafka cluster within the Edge layer.

The Event network is designed to integrate seamlessly with existing event-driven solutions. Its purpose is not to replace these systems, but rather to function as the foundational event fabric, facilitating data delivery and serving as the endpoint where events leave the Event network to interact with the broader ecosystem.

## Semantic Map

The following semantic map shows all the key elements, concepts, and their relationships discussed in this document.

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image4.png)

Figure 3. Semantic map of key elements of ENA

# The Data Model: Domains, Schemas, and Events

## Domains and Schemas

A domain is a named set of fields/attributes, where each field/attribute has a name and a data type associated with it. The name of a domain is a text string that may use dotted notation to encode a standard such as ISA-95 or a proprietary hierarchical structure, with a version suffix separated by the ‘@’ character. A domain name acts as a proxy for a topic, and event messages are transmitted to one or more designated domain endpoints. Conceptually this implies that you have a domain ‘server’ that listens on a specific IP/Port, using a defined message exchange protocol.

The following UML diagram illustrates the relationships between Domains, Schema Attributes and Data types.

![A diagram of a computer AI-generated content may be incorrect.](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image5.jpg)

Figure 4. Domain and Schema UML Diagram

The table below lists the data types supported by an Attribute:

| **Data Type**         | **Description**                                |
| --------------------- | ---------------------------------------------- |
| bool                  | Logical value of either true or false.         |
| short or i16          | Signed 16-bit integer.                         |
| unsigned short or u16 | Unsigned 16-bit integer.                       |
| int or i32            | Signed 32-bit integer.                         |
| unsigned int or u32   | Unsigned 32-bit integer.                       |
| long or i64           | Signed 64-bit integer.                         |
| unsigned long or u64  | Unsigned 64-bit integer.                       |
| float or f32          | 32-bit floating point number.                  |
| double or f64         | 64-bit double-precision number.                |
| binary                | array of 8-bit bytes.                          |
| string                | A sequence of zero or more Unicode characters. |

An example of a domain definition in the Router Source Language, a wireless-enabled thermometer, could look like this.

```
domain FoodStar.Kitchen.Floor.Poultry.Oven1@V1 {
       schema TemperatureProbe {
           attributes [
               float   Battery_Percent;
               float   External_Temp;
               float   Internal_Temp;
               float   RSSI "Signal Strength";
               string  Device_MAC;
               string  FW_Version;
           ]
       }
}
```

An Attribute can contain a label and a format specifier that may be used for display purposes. A domain can only have one schema associated with it.

## The Fixed Event Structure

The exchange of data between an event source and event sink has a fixed structure. There are five elements to each event message.

  - **DOMAIN**: the name of the domain for which the event message is intended for.

  - **ENTITY**: The primary key or entity to which the value belongs. An entity typically has a unique identifier associated with it, consequently this identifier is what distinguishes one entity from another. The Entity represents the identifiable element for the domain and represents a unique instance of something in the world.

  - **ATTRIBUTE**: The name of the attribute within the domain schema to which the value belongs. The attribute must be a valid attribute name as defined by the domain schema. The Attribute is treated as a string constant and can be used in any expression where a string type is permitted.

  - **VALUE**: The value of the ATTRIBUTE for the ENTITY within the DOMAIN. The data type of the Value can be any one of the supported Data Types and must match the data type of the Attribute of the Schema definition for a given Domain.

  - **TIMESTAMP**: The date time stamp of the event message (optional value). If a timestamp is not supplied, the current date/time at ingestion will be used.

The event message format has a fixed structure is represented in the router source language as either a quadruple (DOMAIN, ENTITY, ATTRIBUTE, VALUE) or as a quintuple (DOMAIN, ENTITY, ATTRIBUTE, VALUE, TIMESTAMP)

The following UML diagram outlines the fixed nature and the data types that the VALUE element could take.

![A diagram of a computer code AI-generated content may be incorrect.](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image6.jpg)

Figure 5. Event UML Diagram

The order of the elements of the event message structure is positional, in that first element will be interpreted as being the DOMAIN, the second as the ENTITY, the third as the ATTRIBUTE, and the fourth as the VALUE, with the last optional parameter being the TIMESTAMP. If TIMESTAMP is not given, the event message's timestamp is used; otherwise, the TIMESTAMP parameter applies. This enables generating a distinct timestamp from the original event.

An example of the event message in JSON.

```
{
    "Domain": " FoodStar.Kitchen.Floor.Display.1@V1",
    "Entity": "Probe_5",
    "Attribute": "Internal_Temp",
    "Value": 147,
    "EventTime": "2025-08-21T18:45:33.5132845Z"
}
```

Having a message structure that emits at the attribute level makes it possible to filter values that are relevant before they are sent over the wire, coupled with a low header overhead protocol (UDP, TCP, QUIC) and batching of events can be very effective in maximizing network bandwidth and keeping the data deluge under control.

## Granularity vs. DTOs

A domain schema defines the attributes for a specific area of interest.

In the TemperatureProbe schema example. The area of interest is measuring temperature using a wireless temperature probe. The attributes define the fields of interest that we want to measure or track over time.

```
schema TemperatureProbe {
    attributes [
        float   Battery_Percent;
        float   External_Temp;
        float   Internal_Temp;
        float   RSSI "Signal Strength";
        string  Device_MAC;
        string  FW_Version;
    ]
}
```

The **TemperatureProbe** schema defines a logical data layout. This can be represented as:

| **Battery\_Percent** | **External\_Temp** | **Internal\_Temp** | **RSSI** | **Device\_MAC** | **FW\_Version** |
| -------------------- | ------------------ | ------------------ | -------- | --------------- | --------------- |
|                      |                    |                    |          |                 |                 |

A traditional approach would be to define a Data transfer object (DTO) inclusive of a unique identifier, and to use the DTO as the event message payload. With ENA the granularity of the event message is at the attribute level, and the Domain, Entity and Attribute identify what value is being transmitted, and the destination endpoint determined by the domain value.

The granular event structure helps overcome sparse data issues, where traditional Domain Transfer Objects (DTO) are sent in entirety even when many attributes remain unchanged or null. A common situation in Industrial IoT datasets where only a few attribute change over time.

The advantage of having a timestamp per attribute value, is that it makes it possible to materialize the event stream into a schema structure with additional attributes to capture the time series nature of the data. For a traditional DTO, this is not possible because the timestamp attribute applies to all attributes in the DTO. Converting the TemperatureProbe schema into a DTO would look as follows:

```
struct TemperatureProbe {
float   Battery_Percent;
float   External_Temp;
float   Internal_Temp;
float   RSSI;
string  Device_MAC;
string  FW_Version;
TimeStamp Timestamp;
}
```

An instance of this DTO could look like:

| **Entity** | **Battery\_Percent** | **External\_Temp** | **Internal\_Temp** | **RSSI** | **Device\_MAC**   | **FW\_Version** | **Timestamp** |
| ---------- | -------------------- | ------------------ | ------------------ | -------- | ----------------- | --------------- | ------------- |
| P1         | 0.6                  | 196.56             | 143.24             | \-40     | 00:1A:2B:3C:4D:5E | 1               | T5            |

In this example above, it is unclear which attribute change caused the DTO to be emitted, and to which attribute the timestamp applies. Assuming the timestamp applies to the external temperature, the DTO as instantiated now implies that the internal temperature at time T5 was 143.24, which may factually be incorrect. When a timestamp applies at the aggregate level, then the veracity of the data values from a time perspective may be incorrect. If all values in the DTO attributes were sampled at exactly T5, then the data would be factually correct, and the timestamp would then apply to all attributes in the DTO.

In a traditional DTO approach, a device might aggregate temperature, vibration, and battery status into a single packet sent every minute. The packet typically carries a single timestamp representing the transmission time, not the measurement time of each individual attribute. If the temperature spiked 30 seconds prior to transmission and the anomaly occurred 10 seconds prior, the DTO flattens this temporal nuance, assigning both events the same transmission timestamp. In predictive maintenance or clinical monitoring, this loss of temporal fidelity, data veracity, renders the dataset toxic for near real time applications.

There is somewhat of an art to defining DTOs. Make them too small then you run the risk of having lots of small payloads across the network which can be bandwidth inefficient. Make them too large and you run the risk of sending unchanged and therefore bloated data across the network where it adds no real value.

The fixed event message structure is not designed or intended to replace or compete with current formats, standards or technologies; instead, it provides a consistent framework so that any interaction between a source and a sink always follows this set structure. This has several advantages:

  - Event sources and sinks become composable and interchangeable building blocks

  - There is no need for a translation layer to translate between DTO formats

  - It is easier to evolve an Event Network Architecture

  - It makes for a simpler architecture

It is possible to write a standardized event sink for technologies like Kafka or Parquet that can take in a fixed event structure from any event source in the network, provided it is targeted for the domain.

## Contextual Metadata

Additionally, the event specification allows for contextual and meta data as key value pairs to be included as header data as part of the event message. There are two primary sources of contextual information that can accompany an event 1) Environmental variable values 2) A list of key / value pairs of values that are passed in the header block of a message transmission.

![A white rectangular object with black text AI-generated content may be incorrect.](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image7.png)

Figure 6. Event Message Structure

Within the router source language, the ‘-\>’ operator is used to emit an event, followed by a quintuple, or quadruple event message.

For example:

```
-> ("FoodStar.Kitchen.Floor.Display.1@V1", “Probe_5”, "Celsius",  (Value - 32.0_f32) * (5.0_f32/9.0_f32)) @ EventTime;
```

Contextual data can be passed into the header of the outgoing event message payload. This is a list of key/value pairs, and can be specified in the router source language as follows:

```
["Batch" : "22", "Run" : "Run-" + meta[“Batch”]];

-> ("FoodStar.Kitchen.Floor.Display.1@V1", “Probe_5”, "Celsius", (Value - 32.0_f32) * (5.0_f32/9.0_f32)) @ EventTime;
```

In the example above, two header key/value pairs will be transmitted for all subsequent emitted events. Incoming header data is accessed via the meta\[\] function, containing the name of the key, and returns the value for that key. The value of an environment variable can be accessed by the env\[\] accessor. Both **meta** and **env** accessors return a string value and can therefore be used wherever it is legal to use a string withing the router source language.

A header block is cleared by simply providing an empty header block as in the example below:

```
[];
-> ("FoodStar.Kitchen.Floor.Display.1@V1", “Probe_5”, "Celsius", (Value - 32.0_f32) * (5.0_f32/9.0_f32)) @ EventTime;
```

EventTime in the examples above is a constant that contains the time of the originating event. This is to preserve and propagate the original timestamp. This is optional and could be omitted or replaced with the utcNow() function if preserving the original timestamp is not necessary or important to do.

# The Event Router and Runtime Environment

## Router Source Language

The Router Runtime Language defines all aspects of an event router.

1.  It defines the domain(s) schema(s). If more than one instance of the same schema is defined, the different instances will be merged into a single schema definition.

<!-- end list -->

7.  It specifies the set of listeners for event IP/Port(s) and the ‘listening’ protocol. For protocols that require secure transmissions, it specifies the X.509 certificate to be used to establish a secure connection.

8.  It specifies the static set of targets or endpoints, and the associated protocols and security certificates if required. It also specifies the algorithm (round robin, lowest latency, …) by which events will be sent.

9.  It defines a grammar for processing incoming event messages. These rules determine whether an event message is dropped based on a conditional statement, forwarded to another domain endpoint, or transformed into one or more outgoing events. Additionally, an event can be looped back into the router in support of additional validation or other processing. A ruleset is where the processing logic for a domain is declared.

## Compilation and Bytecode

The Router Source Language Compiler takes as input a router source language specification and produces a binary bytecode representation of any constants as well as the instruction set to implement the rulesets. The compiler ensures syntactic and semantic consistency for a specified configuration. The binary bytecode file produced by the compiler is loaded by an event router.

## The Micro Virtual Machine

The Event Router virtual machine is best characterized as a micro virtual machine. It does not allow dynamic memory allocations therefore does not need to do garbage collection of unused memory. It does not allow preemptive scheduling or concurrent thread execution. It is deterministic in its execution of the instruction set. Consequently, the virtual machine has a small memory footprint and can execute fast and does not have the executional overhead of a full-blown virtual machine, like a Java Virtual Machine.

The time it takes to process a single event to convert a temperature value from Fahrenheit to Celsius.

<table>
<tbody>
<tr class="odd">
<td>Hardware Platform</td>
<td>Architecture</td>
<td>Clock Speed</td>
<td>Event Latency</td>
</tr>
<tr class="even">
<td>STM32H723ZG</td>
<td><p>Arm Cortex-M7 using</p>
<p>RTIC framework</p></td>
<td>400 MHz</td>
<td>33 µs</td>
</tr>
<tr class="odd">
<td>Ryzen 5950x</td>
<td>x86_64</td>
<td>5.1 GHz</td>
<td>422 ns</td>
</tr>
</tbody>
</table>

The number and complexity of the ruleset being processed will affect execution times.

Plans include the ability to compile a Router Source file into native executable instructions targeted for the hardware that the router will run on. This will improve executional speed at the expense of portability.

# Logic and Processing Rules

1.  ## Pipeline Components
    
    An Event Router effectively creates an event pipeline, a) where events are received / ingested, b) event message content is evaluated against a ruleset, and based on the evaluation of the ruleset, c) event messages are emitted to other domains. There are three high level components embodied within an event router.

<!-- end list -->

10. > The listener that receives the event validates that the domains and attributes are valid and then passes the event onto the domain rule evaluator.
    
    1.  > The domain rule evaluation component executes the rules for a given domain, which may result in the creation of new events.

11. > The resolver receives the list of events to emit and then sends them to the endpoint(s) specified based on the default algorithm of round robin, lowest latency or fail over.
    
    These are logically depicted as:
    
    ![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image8.png)

Figure 7. Listeners, Events and Targets

## Rulesets

When an event router starts up, it will load the compiled binary bytecode representation of the router source language file, that contains evaluation and action rules. The bytecode file contains a code block that is executed within the virtual machine of the event router for each incoming event. The bytecode file is hardware architecture agnostic, ensuring that it can be executed on whatever hardware architecture that an ever router can be compiled to. Consequently, an even router can run on a microcontroller, single board computer, server, or cloud compute, both server and serverless, using the same bytecode file. This is significant because it provides the same programming model that can span the continuum from microprocessor to cloud and compute environments in between. It also enables processing the message locally without needing to send the event to an edge or cloud device for processing. This localized event decisioning can be very helpful in bandwidth constrained environments. This means that decision making is one step closer to the source of the data than with traditional Edge based solutions.

Changing the behavior of an event router means changing the rulesets within the router source language file, recompiling that to its bytecode representation, and then making that bytecode file available to the event router. This is a very quick cycle of changing and testing behavior. It does not require the compilation of the event router executable and redeploying it into its environment. It is, as mentioned, as simple as changing the ruleset, compiling it, and making that available to the event router, in whatever environment it is deployed.

Example rule specification:

```
 1. ruleset BroadCastRuleSet FoodStar.Kitchen.Floor.Poultry.Oven1@V1 {
 2.
 3.   accept rule DisplayTemperature : double
 4.       when (Attribute == "Internal_Temp") {
 5.         -> ("FoodStar.Kitchen.Floor.Display.1@V1", Entity, Attribute, Value);
 6.         -> ("FoodStar.Kitchen.Floor.Display.1@V1", Entity, "TimeStamp", EventTime);
 7.       }
 8.
 9.   forward rule ForwardTemperature
10.       when (Attribute == "Internal_Temp" && meta["transform"] == "true")
11.             to ("FoodStar.Kitchen.Floor.Display.2@V1")
12.
13.   accept rule Transform : double
14.       when (Attribute == " Internal_Temp " && meta["transform"] == "true" ) {
15.       string status =
16.      (Value <= 45.0) ? "Raw" :
17.             (Value > 45.0 && Value < 166.0)
18.                         ? "Cooking"
19.                         : (Value >= 166.0 && Value <= 180.0)
20.                             ? "Cooked"
21.                             : "Burned";
22.
23.        -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "Celsius",
24.                         (Value - 32.0) * (5.0/9.0));
25.        -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "Status", status);
26.        -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "TimeStamp", EventTime);
27.       }
28. }
```

Every valid incoming event will be evaluated against the ruleset for this given domain. Depending upon the processing logic specified within the different rules, different events will be emitted whenever the **when** clause for the rule evaluates true.

The evaluation engine maps the fixed event structure values into different constants:

  - > Domain – the value of the domain for the event under evaluation.

  - > Entity – the value of the entity element of the incoming event under evaluation.

  - > Attribute – the value of the attribute element of the incoming event under evaluation.

  - > Value - the value of the value element of the incoming event under evaluation.

  - > EventTime – the timestamp value of the incoming event.
    
    In addition to these values, any environmental variable values can be accessed by using the env\[\] accessor. Similarly, any meta data or header data can be accessed using the meta\[\] accessor.
    
    These constants can be used anywhere within the router source language where the data type for the constants is allowed.
    
    Rules are executed in the order they are declared.
    
    With the router source language, it is possible to define a rule that dynamically generates the domain to which an event will be sent at runtime.
    
    Event routers act as domain servers, that, in effect, create a logical point-to-point data domain network topology over the physical network. The data topology becomes the network topology, they are synonymous. With the point-to-point network there is consequently no need for a central broker to act as an intermediary between event source and sink.
    
    The conceptual relationships between Events, Domains, Listeners, Targets and RuleSets are depicted below:
    
    ![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image9.png)

Figure 8. Relationship between Events, Listeners, Targets and RuleSets

An event router will only accept event messages for the domains and their corresponding attributes. The mapping on the domain topology and that of the network topology makes it very unlikely that every event router will communicate with all other event routers. The domain constrains the network paths based on the data paths.

## Drop / Filter Rule

A drop rule, as its name suggests, will drop an incoming event if it’s corresponding ‘when’ clause evaluates to true, for example:

```
drop rule FilterFreezing : float
    when (Attribute == "Internal_Temp" && Value < 32.0_f32)
```

The drop rule could be used to ensure data quality, or to act as a data firewall.

This rule will effectively filter out values where the Internal\_Temp attribute’s Value is less than 32.0 degrees Fahrenheit.

When a dop rule is invoked, that is, the ‘**when’** clause evaluates to true, the event is dropped and no further rules will be evaluated, and the event router will process the next event.

The ordering of the rules in the source language is very important, and it is good practice to add drop rules and forward rules and then accept rules in that order, otherwise unnecessary or erroneous processing of the event may occur.

## Forward Rule

A forward rule will forward the current event being processed to a designated domain endpoint only if the ‘when’ clause evaluates to true.

```
forward rule ForwardTemperature
when (Attribute == "Internal_Temp" && meta["display"] == "true")
to ("FoodStar.Kitchen.Floor.Display.2@V1")
```

It is possible to have an ‘always’ forward rule by specifying:

```
forward rule ForwardTemperature
when (true)
to ("FoodStar.Kitchen.Kafka.Cooking.2@V1")
```

Once a forward rule is executed, the evaluation engine will proceed to the next rule in the ruleset if there are any.

## Accept Rule

The body of the accept rule is where transformation or the creation of a new event logic can occur. In the example below, the status variable will be assigned a value of “Raw”, “Cooked” or “Burned” depending upon the internal temperature value of the thermometer. This value is the emitted ‘-\>’ to a specified domain.

```
accept rule Transform : double
when (Attribute == " Internal_Temp " && meta["transform"] == "true" ) {
       string status =
      (Value <= 45.0) ? "Raw" :
(Value > 45.0 && Value < 166.0)
? "Cooking"
                        : (Value >= 166.0 && Value <= 180.0)
                             ? "Cooked"
                             : "Burned";

         -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "Celsius",
                         (Value - 32.0) * (5.0/9.0));
.        -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "Status", status);
         -> ("FoodStar.Kitchen.Floor.Display.2@V1", Entity, "TimeStamp", EventTime);
    }
```

It is also possible to normalize or to create more semantically informative events. If the event router is executing on a microcontroller, this transformation logic will happen there, making sure that downstream systems have normalized and correct data.

# Operational Modes: Efficiency and Latency

At present, event listeners and targets could be configured for one of the following protocols:

  - UDP Router Protocol

  - gRPC unary and streaming

  - QUIC

  - HTTP and HTTPS

Event processing is either time biased, or bandwidth biased and is determined by the use case being solved for. The event router will, upon receiving an incoming event, deconstruct the event message into its internal representation, hand this to the evaluation runtime (virtual machine) component and depending on the ruleset(s) defined, will produce zero or more outbound event messages.

## Time-Biased vs. Bandwidth-Biased

Time bias means processing and delivering events as fast as possible, such as needing to control the temperature of a very critical chemical process, where a delay in receiving the temperature reading can cause failure of the process. A bandwidth bias speaks to processing and sending as much data as is possible in a single transmission. Batching events is a typical mechanism for achieving improved bandwidth efficiency. An example of bandwidth bias is to batch soil temperature readings that occur once a second into a batch of 60 events to then only emit one large data block once per minute, thereby significantly improving the ratio of payload data to protocol header size.

## Event Transmission Strategies

Depending upon the specific use case, the events will either be sent one at a time or could be batched to improve throughput efficiency at the expense of immediate delivery. If it is specified that the router should batch every 10 events (for the same domain), then the timeliness of the event’s arrival at its destination point is of less importance than achieving bandwidth efficiency. The most optimal configuration is use-case dependent.

Using the following series of events as example:

| **Domain**                                | **Entity** | **Attribute**    | **Value**         | **Timestamp** |
| ----------------------------------------- | ---------- | ---------------- | ----------------- | ------------- |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | Device\_MAC      | 00:1A:2B:3C:4D:5E | T1            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | FW\_Version      | 1                 | T2            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | Battery\_Percent | .67               | T3            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | RSSI             | \-40              | T4            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | External\_Temp   | 170.00            | T5            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | Internal\_Temp   | 118.00            | T6            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | Internal\_Temp   | 122.00            | T7            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | RSSI             | \-35              | T8            |
| <FoodStar.Kitchen.Floor.Poultry.Oven1@V1> | TPP\_01    | Internal\_Temp   | 128.00            | T9            |

Points to observe from the example above:

  - Attributes Device\_MAC and FW\_Version are emitted once.

  - There is a single event for External\_Temp.

  - There are several RSSI and Internal\_Temp events.

<!-- end list -->

  - Every event has its own time stamp.

  - The attributes in the schema have different change frequencies. Some data changing slower than other data attributes.

**Use case 1**: Timely delivery of an event message is more important than latency or bandwidth optimization. For example: emitting the internal temperature of a chemical process in real time needs to be transmitted quickly and ideally efficiently. This means reducing the connection overhead associated with transmitting a single event over the wire, consequently gRPC in streaming mode or QUIC could be used. It does mean that there will be nine discrete events being emitted and that from a bandwidth efficiency perspective this is not optimal. This could be alleviated within the RuleSet where certain temperature ranges could be filtered out, thereby reducing the number of events that need to be transmitted. Filtering events could be a viable strategy to manage bandwidth economics.

**Use case 2**: Bandwidth is more important than hard real time delivery of an event or events. Batching events are specified in one of two ways 1) the batch size in terms of the number of events to batch before sending e.g. 10, 100 etc. 2) Batching by time duration, e.g. 5 ms or 5 seconds etc. In both cases bandwidth is being optimized. In this case the events are batched, and all event data is de-duplicated, before it is sent over the wire. Given the event stream above and not accounting for connection headers there is approximately a 60% reduction in the number of bytes being sent over the wire. Filtering to reduce the number of events that need to be transmitted is also a viable option to reduce the number of events that are sent over the wire. This is especially effective when this filtering happens as close to the source as possible.

To illustrate the point, assume that all nine events are sent sequentially. In effect a batch size of 1. In this case the total payload size for all nine messages would be:

<table>
<thead>
<tr class="header">
<th><p><strong>Domain</strong></p>
<p><strong>(bytes)</strong></p></th>
<th><p><strong>Entity</strong></p>
<p><strong>(bytes)</strong></p></th>
<th><p><strong>Attribute</strong></p>
<p><strong>(bytes)</strong></p></th>
<th><p><strong>Value</strong></p>
<p><strong>(bytes)</strong></p></th>
<th><p><strong>Timestamp</strong></p>
<p><strong>(bytes)</strong></p></th>
<th><p><strong>Total</strong></p>
<p><strong>(bytes)</strong></p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>39</td>
<td>6</td>
<td>10</td>
<td>17</td>
<td>12</td>
<td>84</td>
</tr>
<tr class="even">
<td>39</td>
<td>6</td>
<td>10</td>
<td>1</td>
<td>12</td>
<td>68</td>
</tr>
<tr class="odd">
<td>39</td>
<td>6</td>
<td>15</td>
<td>4</td>
<td>12</td>
<td>76</td>
</tr>
<tr class="even">
<td>39</td>
<td>6</td>
<td>4</td>
<td>4</td>
<td>12</td>
<td>65</td>
</tr>
<tr class="odd">
<td>39</td>
<td>6</td>
<td>13</td>
<td>4</td>
<td>12</td>
<td>74</td>
</tr>
<tr class="even">
<td>39</td>
<td>6</td>
<td>13</td>
<td>4</td>
<td>12</td>
<td>74</td>
</tr>
<tr class="odd">
<td>39</td>
<td>6</td>
<td>13</td>
<td>4</td>
<td>12</td>
<td>74</td>
</tr>
<tr class="even">
<td>39</td>
<td>6</td>
<td>4</td>
<td>4</td>
<td>12</td>
<td>65</td>
</tr>
<tr class="odd">
<td>39</td>
<td>6</td>
<td>13</td>
<td>4</td>
<td>12</td>
<td>74</td>
</tr>
<tr class="even">
<td>Total bytes transmitted</td>
<td> </td>
<td> </td>
<td>654</td>
<td></td>
<td></td>
</tr>
</tbody>
</table>

If the batch size is increased to 9, then the total number of bytes transmitted would equal:

| **Wire Layout**         | **Bytes** |
| ----------------------- | --------- |
| Domain                  | 39        |
| Entity                  | 6         |
| Attributes              | 65        |
| Values                  | 46        |
| Timestamp               | 108       |
| Total bytes transmitted | 264       |

Bandwidth efficiency can be enhanced by selecting transmission protocols with smaller headers and optimized connection setup and teardown.

In almost all data schemas, there are attributes that change values more frequently than others. There is “fast” data attributes and “slow” data attributes. The rate of change depends on the underlying domain. Sensor data can change frequently, whereas personal information like name and address is unlikely to change frequently.

# Network Topology and Discovery

## Domain Endpoint Resolver (DER)

To create a dynamic peer-to-peer network of domain routers, a look-up function that can resolve a domain name to an IP/Port akin to a Domain Name Server (DNS) must exist. For the event network architecture, the Domain Endpoint Resolver (DER) mesh performs the function of storing domain endpoint names and IP/Port information of all domain endpoints in the system. The DER mesh is a network of resolver servers that communicate with other DER servers. The DER network of servers uses the HyParView membership protocol to disseminate the routing information across the mesh of DER nodes. Where HyParView (Hybrid Partial View) is a membership protocol designed for creating and maintaining overlay networks, primarily used in gossip-based broadcasting. It enables highly scalable, decentralized systems to maintain connectivity and operate efficiently, even when facing high rates of node failures (up to 90%).

The DER mesh maintains a distributed cache of domain names and their corresponding IP/Port addresses with endpoint health status information.

Each DER node has a programmable interface that allows a new event sink to register its domain name and IP/Port for all the domains that it will receive event messages for.

When an event sink registers with a DER, that DER node is responsible for propagating the routing information of that event sink throughout the DER mesh, using a gossip protocol.

The DER node’s programmable interface also allows for domains to be removed from the DER mesh. This can happen in one or two ways.

1)  When an event sink terminates in an orderly fashion, as part of its shutdown process it will send a ‘remove’ call to a DER, that will in turn propagate the removal of the domain name and IP/Port from the DER mesh.

2)  When an event sink does not shut down properly, a periodic health check from an associated DER node to its associated event sink, will report an unhealthy status. The moment that occurs, the domain name and IP/Port information is removed from a DER node, that will in turn propagate the removal of the domain name and IP/Port through the DER mesh.

The network of DER servers create an adjacent network to the event network nodes. This is depicted in the following illustration.

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image10.png)

Figure 9. Event Network with adjacent DER Mesh

## Event Network and Sidecars

When a new event sink or event router is started, it is possible to specify that it should establish a link with the DER mesh network. The DER Mesh network must be operational before event sinks can dynamically be discovered. Once it establishes communication with a DER node in the mesh network the following will happen:

1.  The connecting event sink will be associated with a DER node in the mesh network, creating a communication path between the event sink node and a DER mesh node. It is possible for a DER mesh node to be associated with more than one event sink node.

<!-- end list -->

12. The event sink node will send its list of IP/Port addresses for each of the domains that it can receive event messages for inclusive of static routes defined in the router source language, to its sidecar DER mesh node using HTTP/3 as the communications protocol.

13. The mesh DER node will update its internal routing table with the new routing information and then propagate the new entries throughout the DER mesh network.

14. Periodically every mesh DER node will send out a heartbeat to its list of associated event sinks to determine the health of the event sink node. If an associated event sink does not respond, or becomes unhealthy, the IP/Port for that domain is then removed from the DER mesh node’s routing table, and this change is propagated to all other DER mesh nodes.
    
    Rather than relying on a centralized routing table, every node in the DER mesh network acts as an element of a distributed cache.

## Dynamic Discovery

When an event source wishes to participate in dynamic endpoint discovery, upon startup, it will make a request of a DER mesh node for the routing data for the domains that it is interested in sending events to. Upon receipt of the routing information, it is cached locally with the event sink, and the event source can now establish a point-to-point connection with any of the event sink nodes in its local cache. An event source will therefore only contain a local cache of domain names and IP/Port information for any of the domains that it has sent events to.

The DER mesh exists to ensure that an event sink does not need to know which event source will be sending events. The association between an event sink and event source is dynamically managed by the DER mesh.

There can be any number of DER mesh nodes deployed within or across environments to provide suitable levels of redundancy and/or resiliency. A DER mesh node can execute within edge devices and /or in cloud infrastructure. There is flexibility in how the DER mesh is to be deployed to achieve improved resiliency. For example, it does not make sense to have an Event Router as well as a DER node running on a microcontroller, so the DER node can run on a Single Board Computer, where it acts as a sidecar for the Event Router on the microcontroller.

Static endpoints definitions and dynamic node discovery can coexist alongside one another.

A DER mesh node’s programmable interface has a management interface, enabling external access to the internal state of the node. For instance, it is possible to retrieve the routing table for all nodes within the event network by querying the DER mesh’s routing tables. This enables real-time rendering of the event network topology, making debugging of the data network easier than traditional approaches.

# Resiliency, Redundancy, and Security

From a message transport perspective, all event messages are encrypted between even source and sinks.

An event sink will only accept a message from a known event source. This is done via an access control list for that event router, stating which event sources may send events to it.

To ensure that the compiled binary version of the router source language is tamper proof, the binary file is signed by a user provided certificate, meaning that only that certificate can be used to load and execute the binary instructions.

## ![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image11.png)Routing Algorithms

In Figure 10. Event Broadcast, there is one event source that is configured to broadcast the same event message to three event sinks, all for the same domain. This is an example of redundant paths for the same event. Broadcasting is the default behavior mode for an event source. Each of the three event sources numbered 1 – 3 could execute on different hardware with possibly different and isolated physical network paths / circuits. Path \#3 could be to a Kafka Sink that will create an event log. Path \#1 could be used to drive local logic, and path \#2 could do Cloud based analytics. In this scenario, three event messages are transmitted.

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image12.png)

In Figure 11. Event Broadcast with Round Robin, Lowest Latency and Fail Over, there are three groupings with seven event sinks, two in the first group, three in the second and two in the last group. For each of the three groups there are different event processing modalities.

Round Robin – rotating to which event sink the message will be sent upon each event emit. Only one sink will receive the event message.

Lowest Latency – Sending the message to the event sink that has the lowest latency. Between the three events sinks, only one will receive the event message, the one with the lowest latency. Latency metrics are typically monitored by sending heartbeats to each destination or through the DER mesh.

Fail Over – The event message will be sent to the primary event sink. If that fails, then it will fail over to the next sink in the group.

> In the example above, three messages will be broadcast to each of the three groups, where each group can have a different event handling modality.

To the earlier point, each of the event sinks could be on different network circuits or even in different Cloud regions or Datacenters. For the last group, the primary event sink could be on Could Region A, and the failover sink could be in Cloud Region B, thereby ensuring that the event will reach the intended endpoint.

The ENA does not have the concept of a topic mailbox. With ENA in asynchronous mode, if the event sink is unavailable, and there is no fail over group with another sink in that group, that event will be discarded and lost. In synchronous mode, the event sink will process every event it receives.

![](Technical%20Brief%20for%20Event%20Network%20Architecture_media/media/image13.png)In Figure 12. Round Robin between Failover Groups, there are two levels of grouping, one where the event will be broadcast to the outer group, then within that group, the event handling modality is round robin, meaning that the event will first be sent to the first failover group, and the next event message will be sent to the second failover group. Within each of the failover groups only one event sink will get the message. In this example scenario only one event will be sent.

The event handling modalities defined in the Router Runtime Source Language. Below is a snippet of how the example above can be defined in the Source Language.

```
 1.     targets {
 2.         http endpoint A1 {
 3.             address "12.45.12.67:5010";
 4.         }
 5.         http endpoint A2 {
 6.             address "122.65.12.45:5010";
 7.         }
 8.         http endpoint A3 {
 9.             address "18.45.12.32:5010";
10.         }
11.         http endpoint A4 {
12.             address "73.45.112.32:5010";
13.         }
14.         http endpoint A5 {
15.             address "12.145.12.68:5010";
16.         }
17.         target Domain.A@V1 {
18.             round_robin [
19.                 fail_over [A1,A2],
20.                 fail_over [A3,A4,A5]
21.             ]
22.         }
23.     }
24.
```

It is noteworthy that in either Round Robin or Lowest Latency modality, an event router becomes a load balancer.

There is flexibility in how messages will be sent to the event sink and can be designed to suit the situation at hand. There are effectively two levels of redundancy at play:

  - Physical Network

  - Event Network

Physical and event networks may be jointly leveraged to address requirements related to redundancy, resilience, or scalability.
