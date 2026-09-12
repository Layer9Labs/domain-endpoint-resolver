# DER Sequence Diagrams

Interaction sequences for the **Domain Endpoint Resolver (DER)** mesh described in
*Technical Brief: Event Network Architecture (ENA)* — Layer 9 Labs LLC.

The DER mesh is a distributed cache of domain names, their IP/Port addresses, and endpoint
health status. Routing information is disseminated across DER nodes using the HyParView
membership protocol over gossip. Event **sinks** *register* what they can receive; event
**sources** *resolve* where to send.

---

## 1. Event Sink ↔ DER Node

An event sink (or event router acting as a sink) registers the domains it accepts events for,
is health-checked by its associated DER node, and is removed from the mesh on shutdown or
failure.

```mermaid
sequenceDiagram
    autonumber
    participant Sink as Event Sink / Domain Router
    participant DER as Associated DER Mesh Node
    participant Mesh as Peer DER Nodes

    Note over Sink,DER: The DER mesh must be operational before sinks can be discovered

    Sink->>DER: Establish link on startup over HTTP/3
    DER-->>Sink: Association confirmed, communication path open
    Note over Sink,DER: One DER node may be associated with many event sinks

    Sink->>DER: Register domain names and IP/Port for every domain it accepts
    Note right of Sink: Includes static routes declared in the router source language

    DER->>DER: Update internal routing table with domain, IP/Port and health
    DER->>Mesh: Propagate new entries by gossip using HyParView
    Mesh-->>DER: Routing information converged across the mesh

    loop Periodic heartbeat
        DER->>Sink: Health check
        alt Sink responds healthy
            Sink-->>DER: Healthy
        else No response or unhealthy
            DER->>DER: Remove domain and IP/Port from routing table
            DER->>Mesh: Propagate removal across the mesh
        end
    end

    alt Orderly shutdown
        Sink->>DER: Remove call for its domain names and IP/Port
        DER->>DER: Delete entries from routing table
        DER->>Mesh: Propagate removal across the mesh
        DER-->>Sink: Removal acknowledged
    end
```

**Notes**

- The sink's programmable-interface registration is what makes it discoverable — no central broker holds the table.
- Every DER node acts as an element of the distributed cache rather than a replica of a central routing table.
- Removal happens two ways: an explicit `remove` call during orderly shutdown, or an unhealthy heartbeat result.

---

## 2. Event Source ↔ DER Node

An event source resolves the domains it intends to send to, caches those routes locally, and
then talks point-to-point with the event sink. The DER mesh is not in the data path.

```mermaid
sequenceDiagram
    autonumber
    participant Source as Event Source
    participant DER as DER Mesh Node
    participant Cache as Source Local Route Cache
    participant Sink as Event Sink / Domain Router

    Note over Source,DER: Dynamic discovery is requested at source startup

    Source->>DER: Request routing data for the domains it wants to send to
    DER->>DER: Look up domain entries in the distributed cache
    DER-->>Source: Domain names with IP/Port addresses and health status
    Source->>Cache: Store routes locally
    Note over Cache: Cache holds only domains this source has sent events to

    Source->>Sink: Open encrypted point-to-point connection using X.509
    Sink-->>Source: Accepted, source present on the access control list
    Note over Source,Sink: The DER mesh is out of the data path from here on

    loop For each emitted event
        Source->>Cache: Resolve domain to endpoint
        Cache-->>Source: IP/Port for the domain
        Source->>Sink: Event message of Domain, Entity, Attribute, Value, Timestamp
    end

    alt Endpoint fails or leaves the mesh
        Sink--xSource: Delivery fails
        Source->>DER: Re-request routing data for that domain
        DER-->>Source: Updated IP/Port set for surviving endpoints
        Source->>Cache: Refresh local cache
        Source->>Sink: Resume delivery on an alternate endpoint
    end
```

**Notes**

- Decoupling is the point: an event sink never needs to know which event source will send to it. The association is managed dynamically by the DER mesh.
- A source caches only the domains it actually uses, keeping the local footprint small enough for microcontrollers and single-board computers.
- Static endpoint definitions and dynamic discovery can coexist — a source may use both.
- Broadcast is the default behaviour for an event source, so one resolved domain may map to several sink endpoints for redundancy.

---

*Source: Technical Brief — Event Network Architecture (ENA), sections 7.1 Domain Endpoint
Resolver (DER), 7.2 Event Network and Sidecars, and 7.3 Dynamic Discovery.*
