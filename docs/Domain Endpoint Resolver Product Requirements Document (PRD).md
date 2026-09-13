# Product Requirements Document (PRD): Domain Endpoint Resolver

## 1. System Objective
A fault-tolerant, resilient and scalable Rust-based distributed mesh network for event domain endpoint discovery. An event sink, or router, can register its event endpoint as well as its health check URI with a mesh node. An event source, or router can query a mesh network node for the endpoint address of a particular domain. The integrity and health of the mesh network is of critical importance, consequently, the mesh network, should on a periodic basis call the health check endpoints, and remove any domain/endpoints from the mesh network.

## 2. Tech Stack & Architectural Constraints
- **Architecture Pattern**: Distributed Architecture utilizing Domain-Driven Design principles. The DER mesh is a distributed cache of domain names, their IP/Port addresses, and endpoint health status. Routing information is disseminated across DER nodes using the HyParView membership protocol over gossip. Event sinks register what they can receive; event sources resolve where to send.

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
- The system will provide a ```/register``` route, that will allow an event sink or event router to register listener endpoint information with the mesh.

- An event sink can register multiple domains, where each domain must contain one or more listeners. 

- The system needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- This is an HTTP Post, with the following message payload format:
```
    [
        {
            "domain" : "starfoods.food@v1",
            "healthcheck" : "127.0.0.1:5076",
            "listeners" : [
                "https://domain1:5050",
                "https://domain2:5051"
            ]  
        },
        {
            "domain" : "starfoods.quality@v1",
            "healthcheck" : "127.0.0.1:5077",
            "listeners" : [
                "https://domain3:5060",
                "https://domain4:5061"
            ]  
        }
    ]
```

### Requirement 2 - Domain Lookup
- An event sink or event router needs to query the mesh network to look up the endpoint address or addresses for a given domain.
The system will provide a ```/lookup``` route.

- The HTTP GET request: ```GET /lookup?domain=starfoods.quality@v1```

- The system needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- The lookup will return a JSON payload with the following structure:
```
    {
        "domain" : "starfoods.quality@v1",
        "endpoints" : [
            "https://domain3:5060",
            "https://domain4:5061"
        ]
    }
```

### Requirement 4 - Domain Removal
- The system will provide a ```/remove``` route, that will allow an event sink or event router to instruct the mesh network to remove a domain name and associated listener endpoints.
The HTTP ```DELETE /remove?domain=starfoods.quality@v1```

- The system must ensure that appropriate URL encoding is done to the HTTP DELETE request.

### Requirement 3 - Data Serialization
- All data sent from an event sink or router, to a mesh node, will be serialized into a binary format using messagepack. The mesh node will deserialize the data and then propagate the data through the mesh network.

- All data sent from a mesh node to a event sink or router must be serialized into a binary format using messagepack. Data received from a mesh node must be deserialized using messagepack.

### Requirement 4 - Encrypted Payload
The system must provide the ability to encrypt the payload data whilst in transit over a network. The system will provide a command line argument specifying the certificate that is to be used in encrypting payload data. This is inherently provided by HTTP/3 TLS 1.3 or greater. 

### Requirement 5 - Health Check
On a user specified periodic basis, via a command line parameter setting, the mesh node should make a call to the health check URI that was provided as part or the ```/Register``` call, to determine if the event sink or router is healthy. If the event sink or router returns an unhealthy state, the mesh node should remove the endpoint for the domain it belongs to.
The healthy endpoint should return an HTTP Status code of 200 and the response body text should contain the text "healthy". An unhealthy endpoint should return an HTTP status code 503, and teh text "unhealthy" in the response body. 

## 4. Out of Scope (Negative Constraints)
- Event Router and Event Sink implementations
- Do NOT implement persistent database storage this service only routes data

## 5. Core Domain Entities
- **Entity 1**: Event Sink. A process that listens on an IP/Port address for incoming event data. The processing of the event itself is determined by the purpose of the event sink. For example, an event sink could receive incoming event messages and persist them to a Kafka topic. 

- **Entity 2**: Event Router. Is an event sink, that will listen for incoming event data on and IP/Port. The event will be processed by the ruleset defined for that router.

- **Entity 3**: Event Source. A process that generates event data and will transmit event data to an event sink or event router.

## 6. Error Handling & Edge Cases

