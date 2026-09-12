# Product Requirements Document (PRD): Domain Endpoint Resolver

## 1. System Objective
A fault-tolerant, resilient and scalable Rust-based distributed mesh network for event domain endpoint discovery. An event sink, or router, can register its event endpoint as well as its health check URI with a mesh node. An event source, or router can query a mesh network node for the endpoint address of a particular domain. The integrity and health of the mesh network is of critical importance, consequently, the mesh network, should on a periodic basis call the health check endpoints, and remove any domain/endpoints from the mesh network.

## 2. Tech Stack & Architectural Constraints
- **Architecture Pattern**: Distributed Architecture utilizing Domain-Driven Design principles
- Use HyParView and PlumbTree to gossip and propagate node data within the mesh network.
- **Primary Language**: Rust, strict adherence to idiomatic error handling
- **Communication Protocol**: HTTP/3 with fall back to HTTP/2
- **Key Rust Crate Dependencies**:
  - saorsa-gossip
  - clap
  - quiche + tokio-quiche
  - rmp

## 3. Functional Requirements

### Requirement 1 - Endpoint Registration:
The system will provide a ```/Register``` endpoint, that will allow an event sink or event router to register endpoint data with the mesh.

An event sink can register multiple domains, where each domain must contain one or more listeners. 

This is an HTTP Post, with the following message payload format:
```
[
  {
    "domain" : "EventDomain.Food@v1",
    "healthcheck" : "127.0.0.1:5076",
    "listeners" : [
      "https://domain1:5050",
      "https://domain2:5051"
    ]  
  },
    {
    "domain" : "EventDomain.uality@v1",
    "healthcheck" : "127.0.0.1:5077",
    "listeners" : [
      "https://domain3:5060",
      "https://domain4:5061"
    ]  
  }
]
```

### Requirement 2 - Data Serialization
- All data sent from an event sink or router, to a mesh node will be serialized into a binary format using messagepack. The mesh node will deserialize the data and then propagate the data through the mesh network.
- All data sent from a mesh node to a event sink or router must be serialized into a binary format using messagepack. Data received from a mesh node must be deserialized using messagepack.

### Requirement 4 - Encrypted Payload
The system must provide the ability to encrypt the payload data whilst in transit over a network. The system will provide a command line argument specifying the certificate that is to be used in encrypting payload data. This is inherently provided by HTTP/3 TLS 1.3 or greater. 

### Requirement 5 - Health Check
On a user specified periodic basis via a command line parameter the mesh node should make a call to the health check URI that was provided as part or the ```/Register``` call, to determine if the event sink or router is healthy. If the event sink or router returns an unhealthy state, the mesh node should remove the endpoint for the domain it belongs to.
The healthy endpoint should return an HTTP Status code of 200 and the response body text should contain the text "healthy". An unhealthy endpoint should return an HTTP status code 503, and teh text "unhealthy" in the response body. 

## 4. Out of Scope (Negative Constraints)
- Event Router and Event Sink implementations
- Do NOT implement persistent database storage this service only routes data

## 5. Core Domain Entities
- Entity 1: Event Sink
- Entity 2: Event Router
- Entity 3: Event Source

## 6. Error Handling & Edge Cases

