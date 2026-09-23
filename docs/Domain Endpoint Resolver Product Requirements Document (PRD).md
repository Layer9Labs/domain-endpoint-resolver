# Product Requirements Document (PRD): Domain Endpoint Resolver

## 1. System Objective
To build a fault-tolerant, resilient and scalable Rust-based distributed mesh network for event domain endpoint discovery. An event sink, or router, can register the listeners that it binds to for incoming event data as well as registering its health check URI with a mesh node. An event source, or router can query a mesh network node for the endpoint IP address and port of a particular domain. Each node in the mesh will store the domains and their listener and health check endpoints as part of a local routing table. The registered domains and their listening addresses and health checks must be propagated through the mesh network. The integrity of the domain endpoint data is of critical importance, consequently, each mesh node, should on a periodic basis call the health check endpoints, and remove any domain/endpoints from its local routing table. Changes to the node routing table should be pushed into the mesh, following a Change Data Capture (CDC) pattern. Routing information is disseminated across mesh nodes using the HyParView membership protocol over gossip. Event sinks register what they can receive; event sources resolve where to send.

## 2. Architectural Elements

There are five components or elements that comprises a Domain Endpoint Node:

1. **HTTP/3 with HTTP/2 fallback** execution thread using tokio-quiche that exposes four HTTP routes as an API surface. There are:
   - ```/register```
   - ```/health```
   - ```/lookup```   
   - ```/remove```

    This thread communicates with downstream threads via a channel.

2. **Domain Endpoint Node Routing Table**, which is a dictionary of endpoint data, where the key of the dictionary is the fully qualified domain name, and the value is an array or list of endpoint and health data.

3. **Routing Table Manager**, an execution thread that will consume messages from the HTTP/3 thread via a channel, and them make updates to the **Domain Endpoint Node Routing Table** based on the HTTP REST Verb and route. 

    This thread communicates with downstream threads via a channel.
  
4. **Gossip Coordinator** an execution thread responsible for communicating routing table changes from the **Routing Table Manager** to other Domain Endpoint Nodes in the mesh. It is also responsible for communicating mesh changes an updates back to **Routing Table Manager**.
Communication with the **Routing Table Manager** thread is done via a channel. 

1. **Health Heartbeat** an execution thread, that is responsible for making HTTP health check for all of the locally scoped healthCheck entries in the **Domain Endpoint Node Routing Table**. This thread will communicate health check changes with the **Routing Table Manager** using the same channel that the **HTTP/3 with HTTP/2 fallback** thread uses.

## 3. Technology Stack

- Use HyParView and PlumTree to gossip and propagate node data within the mesh network.

- **Primary Language**: 
  - Rust
  - Strict adherence to idiomatic error handling
  - There should be no ```.unwrap()``` function calls in production code

- **Communication Protocol**:
  - HTTP/3 with fall back to HTTP/2

- **Key Rust Crate Dependencies**:
  - saorsa-gossip
  - clap
  - quiche + tokio-quiche
  - rmp
  - tracing

## 4. Functional Requirements

### Requirement 1 - Domain Endpoint Node Routing Table:
The Domain Endpoint Node Routing Table is an in-memory data store with no persistence capabilities. 

The structure of the routing table is a dictionary where the key is the fully qualified domain name, and the values for the key includes the following members/fields/elements:
- **"listener"** a string representing the IP/Port address at which the event sink will listen for incoming events.
- **healthCheck** a string representing the health check URI of the event sink. This implies that an event sink should expose a health check route.
- **healthCounter** an integer value, initially zero, but a counter of the number of times a health check has failed.
- **healthStatus** a string representing the health status of the "listener".
- **isLocalEntry** a boolean value indicating whether or not the entry in the Router Table is one that was registered by an event sink, or if the entry was obtained from a mesh lookup.

Outlined below is a JSON example of what the payload of the routing table could be.

```json
    {
        "starfoods.food@v1" : [
          {
            "listener" : "https://domain1:5050",
            "healthCheck" : "https://domain1:5076",
            "healthCounter" : 0,
            "healthStatus" : "healthy",
            "isLocalEntry" : true
          },
          {
            "listener" : "https://domain1:5051",
            "healthcheck" : "https://domain1:5077",
            "healthCounter" : 0,
            "healthStatus" : "degraded",
            "isLocalEntry" : true
          }
        ],
        "starfoods.quality@v1": [
          {
            "listener" : "https://domain2:5060",
            "healthcheck" : "https://domain2:5061",
            "healthCounter" : 0,
            "healthStatus" : "failed",
            "isLocalEntry" : true
          },
          {
            "listener" : "https://domain3:5080",
            "healthcheck" : "https://domain3:5081",
            "healthCounter" : 0,
            "healthStatus" : "healthy",
            "isLocalEntry" : false
          }
        ]
    }
```

The routing table must only be updated or changed by the **Routing Table Manager** thread, however the **Health Heartbeat** thread will need to have read access to the **Domain Endpoint Node Routing Table**. This means that both threads will need a reference to the routing table in order to either update it, or to be able to read the contents.

The lifetime of the **Domain Endpoint Node Routing Table** is that of the **Domain Endpoint Node**. The routing table is ephemeral.

### Requirement 2 - Endpoint Registration:
- The system will provide a ```/register``` route, that will allow an event sink or event router to register listener and healthcheck endpoint information with a mesh node.

- An event sink can register multiple domains, where each domain must contain one or more listeners. 

- The system needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- This is an HTTP Post, with the following message payload format:
```json
    [
        {
            "domainName" : "starfoods.food@v1",
            "domainUris" : [
              {
                "listener" : "https://domain1:5050",
                "healthcheck" : "https://domain1:5076"
              },
              {
                "listener" : "https://domain1:5051",
                "healthcheck" : "https://domain1:5077"
              }
            ]
        },
        {
            "domainName" : "starfoods.quality@v1",
            "domainUris" : [
              {
                "listener" : "https://domain2:5060",
                "healthcheck" : "https://domain2:5061"
              },
              {
                "listener" : "https://domain3:5080",
                "healthcheck" : "https://domain3:5081"
              }
            ]
        }
    ]
```

It is possible to have two event sinks listening for events for the same domain. In the mesh network there should be single representation of all domain nodes and they respective health checks.

- When two or more event sinks register the same domain concurrently, they will have different health check URI's and listener URI's. The system will merge the the two domain entries, by removing duplicates from both the "healthcheck" array and the "listener" entries.

- Given that there could be multiple "listener" and "healthcheck" URI's for a given domain, when a healthcheck fails, the healthcheck's corresponding listener and healthcheck URI should be removed from the list of "domainUris" URI's for the given domain. There is a 1:1 relationship between the "listener" URI and it's healthcheck endpoint. When the healthcheck is removed because of a failure, so should the listener entry for that healthcheck.

### Requirement 3 - Domain Lookup
- An event sink or event router needs to query the mesh network to look up the endpoint address or addresses for a given domain.
The system will provide a ```/lookup``` route.

- The HTTP GET request: ```GET /lookup?domain=starfoods.quality@v1```

- The system needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- The lookup will return a JSON payload with the following structure:
```json
    {
        "domain" : "starfoods.quality@v1",
        "endpoints" : [
            "https://domain3:5060",
            "https://domain4:5080"
        ]
    }
```

### Requirement 4 - Domain Removal
- The system will provide a ```/remove``` route, that will allow an event sink or event router to instruct the mesh network to remove a domain name and associated listener endpoints.
The HTTP ```DELETE /remove?domain=starfoods.quality@v1```
- The removal is scoped at the domain level, meaning that all listener and healthchecks will be deleted. There should be no trace of a domain left in the mesh network and after eventual consistency is reached. 

- The system must ensure that appropriate URL encoding is done to the HTTP DELETE request.

### Requirement 5 - Data Serialization
- All data sent from an event sink or router, to a mesh node, will be serialized into a binary format using messagepack. The mesh node will deserialize the data and then propagate the data through the mesh network.

- All data sent from a mesh node to a event sink or router must be serialized into a binary format using messagepack. Data received from a mesh node must be deserialized using messagepack.

### Requirement 6 - Encrypted Payload
- The system must provide the ability to encrypt the payload data whilst in transit over a network. The system will provide a command line argument specifying the certificate that is to be used in encrypting payload data. This is inherently provided by HTTP/3 TLS 1.3 or greater.

- Inter-node gossip traffic should be encrypted, terminating gossip transport over TLS, with the cert and key that is specified on the command line..

### Requirement 7 - Health Check
- On a user specified periodic basis, via a command line parameter setting, the mesh node should make a call to the health check URI's that was provided as part or the ```/Register``` call, to determine if the event sink or router is healthy. If the event sink or router returns an unhealthy state, the mesh node should remove the endpoint for the domain it belongs to.

- The healthy endpoint should return an HTTP Status code of 200 and the response body text should contain the text "healthy". An unhealthy endpoint should return an HTTP status code 503, and the text "unhealthy" in the response body.

- Given that there could be two or more concurrent registrations for a domain, and that these need to be merged and propagated back into the network. It is important that every healthcheck URI is called for a domain. If a domain has two different healthcheck URI's (after a merge) then the mesh should call both healthceck URI's.

- When a health check fails for a {listener, healthcheck} pair, it should not immediately be removed from the list of "domainUris". A numeric integer counter for the number of retries should be associated with each pair, staring with a value of zero. A user specified threshold for the maximum number of retries must be specified as a command line parameter. When an "unhealthy" status is returned from the health check, then the counter is increased by one. When the counter is either equal to or exceeds the maximum number of retries, then it is to be removed from the "domainUris" list. When a counter for a {listener, healthcheck} pair is greater than zero but has not exceeded the maximum number of retries value, and a health check returns "healthy", then the counter for the pair must be reset to zero.
  
### Requirement 8 - Logging
- The system should log data when:
  1.  an event sink registers with the mesh network. It should log:
      - the action being performed which is "Registration"
      - source IP/Port of the registering node
      - the domain that it is registering
      - the list of listener URI's
      - the list of healthchecks URI's
  2. an event sink does a lookup. It should log:
      - the action being performed which is "Lookup"
      - the name of the domain it is looking for
      - whether or not the lookup was successful, by logging the listener endpoint URI's
  3. an event sink does a remove. It should log:
      - the action being performed which is "Remove"
      - the name of the domain it removing

## 5. Out of Scope (Negative Constraints)
- Event Router and Event Sink implementations
- Do NOT implement persistent database storage this service only routes data
- The management interface into the mesh is out of scope
- The ability to retrieve all domain endpoints from the mesh is out of scope

## 6. Core Domain Entities
- **Event Sink:** A process that listens on an IP/Port address for incoming event data. The processing of the event itself is determined by the purpose of the event sink. For example, an event sink could receive incoming event messages and persist them to a Kafka topic. 

- **Event Router:** Is an event sink, that will listen for incoming event data on and IP/Port. The event will be processed by the ruleset defined for that router.

- **Event Source:** A process that generates event data and will transmit event data to an event sink or event router.

- **Endpoint:** Is either an IPv4 or IPv6 address inclusive of the port. The endpoint contains an actual IP address and Port.

## 7. Error Handling & Edge Cases

