# Product Requirements Document (PRD): Domain Endpoint Resolver

## 1. System Objective
To build a fault-tolerant, resilient and scalable Rust-based distributed mesh network for event domain endpoint discovery. An event sink, or router, can register the listeners that it binds to for incoming event data as well as registering its health check URI with a mesh node. An event source, or router can query a mesh network node for the endpoint IP address and port of a particular domain. Each node in the mesh will store the domains and their listener and health check endpoints as part of a local routing table. The registered domains and their listening addresses and health checks must be propagated through the mesh network. The integrity of the domain endpoint data is of critical importance, consequently, each mesh node, should on a periodic basis call the health check endpoints, and remove any domain/endpoints from its local routing table. Changes to the node routing table should be pushed into the mesh, following a Change Data Capture (CDC) pattern. Routing information is disseminated across mesh nodes using the HyParView membership protocol over gossip. Event sinks register what they can receive; event sources resolve where to send.

## 2. Architectural Elements

The following diagram defines the structural element and communication paths between the elements that make up the Domain Endpoint Node.

![Domain Endpoint Node](<Domain EndPoint Node.svg>)

There are five components or elements that comprises a Domain Endpoint Node:

1. **HTTP Server** execution thread using tokio-quiche that exposes four HTTP routes as an API surface. There are:
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

1. **Health Heartbeat** an execution thread, that is responsible for making HTTP health check for all of the locally scoped healthCheck entries in the **Domain Endpoint Node Routing Table**. This thread will communicate health check changes with the **Routing Table Manager** using the same channel that the **HTTP Server** thread uses.

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

### Requirement 1 - Channel Payload Format

The data that is written and read from the communication frontend and backend channels between the **HTTP Server** thread and the **Routing Table Manager** thread, between the **Routing Table Manager** and the **Gossip Coordinator** thread, and between the **Health Heartbeat** thread and the **Routing Table Manager** thread should conform to the following structure:

- A member named "origin" that is an enumerated type that can take one of two values 1) HTTP 2) HEALTH.
- A member named "action" that is an enumerated type that can take one of several values 1) ADD 2) REMOVE 3) UPDATE 4) LOOKUP
- A member named "domain" that is a string containing the fully qualified domain name
- A member named "listener" that is the listening endpoint for the event sink that is being inserted, updated or removed.
- A member named "healthCheck" that is the health check URI for the event sink that is being inserted, updated or removed.
- A member named "healthCheck" that is the health check status that would be return by the healthCheck endpoint.

Below is a JSON example of what the payload data could look like.

```json
{
    "origin" : "HTTP",
    "action" : "INSERT",
    "domain" : "starfoods.food@v1",
    "listener" : "https://domain1:5050",
    "healthCheck" : "https://domain1:5076",
    "healthStatus" : "healthy"
}
```

### Requirement 2 - Data Serialization
- All data sent across the channels between threads, must be serialized into a binary format using messagepack. The **Gossip Coordinator** thread node will deserialize the data and then propagate the data through the mesh network.

- All data sent from a mesh node to a event sink or router must not be serialized with messagepack.

### Requirement 3 - Encrypted Payload
- The system must provide the ability to encrypt the payload data whilst in transit over a network. The system will provide a command line argument specifying the certificate and key that is to be used in encrypting payload data. This is inherently provided by HTTP/3 TLS 1.3 or greater.**

- Inter-node gossip traffic should rely on what is provided by the saorsa-gossip crate.

### Requirement 4 - Domain Endpoint Node Routing Table:
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
            "healthCheck" : "https://domain1:5077",
            "healthCounter" : 0,
            "healthStatus" : "degraded",
            "isLocalEntry" : true
          }
        ],
        "starfoods.quality@v1": [
          {
            "listener" : "https://domain2:5060",
            "healthCheck" : "https://domain2:5061",
            "healthCounter" : 0,
            "healthStatus" : "failed",
            "isLocalEntry" : true
          },
          {
            "listener" : "https://domain3:5080",
            "healthCheck" : "https://domain3:5081",
            "healthCounter" : 0,
            "healthStatus" : "healthy",
            "isLocalEntry" : false
          }
        ]
    }
```

The routing table must only be updated or changed by the **Routing Table Manager** thread, however the **Health Heartbeat** thread will need to have read access to the **Domain Endpoint Node Routing Table**. This means that both threads will need a reference to the routing table in order to either update it, or to be able to read the contents.

The lifetime of the **Domain Endpoint Node Routing Table** is that of the **Domain Endpoint Node**. The routing table is ephemeral.


### Requirement 5 - HTTP Server

This thread "listens" for inbound HTTP requests on an IP Address/Port that is specified on the command line.

#### Requirement 5.1 - Endpoint Registration:
- The **HTTP Server** thread will provide a ```/register``` route, that will allow an event sink or event router to register listener and healthCheck endpoint information with a Domain Endpoint Node or mesh node.

- An event sink can register multiple domains, where each domain must contain one or more listeners. 

- The **HTTP Server** thread needs to ensure that appropriate URL encoding is done to the HTTP POST request.

- This is an HTTP POST, with the following register message payload format:
    ```json
    [
        {
            "domainName" : "starfoods.food@v1",
            "domainUris" : [
              {
                "listener" : "https://domain1:5050",
                "healthCheck" : "https://domain1:5076"
              },
              {
                "listener" : "https://domain1:5051",
                "healthCheck" : "https://domain1:5077"
              }
            ]
        },
        {
            "domainName" : "starfoods.quality@v1",
            "domainUris" : [
              {
                "listener" : "https://domain2:5060",
                "healthCheck" : "https://domain2:5061"
              },
              {
                "listener" : "https://domain3:5080",
                "healthCheck" : "https://domain3:5081"
              }
            ]
        }
    ]
    ```

Upon receipt of this payload, the **HTTP Server** thread will iterate through the list of domains in the **Domain Endpoint Routing Table** and create a list of Channel Payload Format records and write them serially to the frontend communication channel.

#### Requirement 5.2 - Domain Lookup

- The **HTTP Server** thread will provide a ```/lookup``` route to enable an event sink or event router to query the mesh node to look up the endpoint address or addresses for a given domain.

- The **HTTP Server** thread needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- The **HTTP Server** will write the following data payload to the frontend channel upon receiving a HTTP GET request: ```/lookup?domain=starfoods.quality@v1```.

  ```json
  {
    "origin" : "HTTP",
    "action" : "LOOKUP",
    "domain" : "starfoods.food@v1",
    "listener" : "",
    "healthCheck" : "",
    "healthStatus" : ""
  }
  ```

- Upon receipt of a response from the frontend channel the **HTTP Server** will format a response to the HTTP GET request. 

- A successful response will return a JSON payload resembling the following structure and a HTTP Status code of 200:
    ```json
    {
        "domain" : "starfoods.quality@v1",
        "endpoints" : [
            "https://domain3:5060",
            "https://domain4:5080"
        ]
    }
    ```
- A response that contains no endpoints will return a JSON payload resembling the following structure and a HTTP Status code of 404:
    ```json
    {
        "domain" : "starfoods.quality@v1",
        "endpoints" : [
        ]
    }
    ```

#### Requirement 5.3 - Domain Removal
- The **HTTP Server** thread will provide a ```/remove``` route, that will allow an event sink or event router to instruct the **Routing Table Manager** to remove a domain name and associated listener endpoints from it's local Routing Table or from the mesh.
  
The HTTP ```DELETE /remove?domain=starfoods.quality@v1&listener=https://domain1:5050&healthCheck=https://domain1:5076```
- The removal is based a tuple containing (domain, listener, healthCheck) as part of the ```/remove``` URL. All three values in the request must be an exact match to an entry in the **Domain Endpoint Routing Table** in order for it to be removed.

- The **HTTP Server** thread needs to ensure that appropriate URL encoding is done to the HTTP DELETE request.

- The **HTTP Server** will write the following data payload to the frontend channel upon receiving a  HTTP DELETE ```/remove``` request.  

    ```json
    {
        "origin" : "HTTP",
        "action" : "DELETE",
        "domain" : "starfoods.quality@v1",
        "listener" : "https://domain3:5080",
        "healthCheck" : "https://domain3:5081",
        "healthStatus" : ""
    }
    ```

#### Requirement 5.4 - Health Check
- The **HTTP Server** will provide a ```/health``` route, that will allow an external process to determine the health status of a Domain Endpoint Node.
- The HTTP Server thread needs to ensure that appropriate URL encoding is done to the HTTP GET request.

- A health state should return a HTTP Status code of 200 and a JSON response body structured like this:

    ```json
    {
      "ok": true,
      "status": "healthy"
    }
    ```

### Requirement 6 - Routing Table Manager

It is possible to have two event sinks listening for events for the same domain. In the mesh network there should be single representation of all domain nodes and they respective health checks.

- When two or more event sinks register the same domain concurrently, they will have different health check URI's and listener IP Address/Port. The system will merge the the two domain entries, by removing duplicates from the ```domainUris``` array by matching on the ```{ listener, healCheck }``` pair.

- Given that there could be multiple "listener" and "healthCheck" URI's for a given domain, when a healthCheck fails, the healthCheck's corresponding listener and healthCheck URI should be removed from the list of "domainUris" URI's for the given domain. There is a 1:1 relationship between the "listener" URI and it's healthCheck endpoint. When the healthCheck is removed because of a failure, so should the listener entry for that healthCheck.

### Requirement 7 - Gossip Coordinator


### Requirement 8 - Health Heartbeat


### Requirement 9 - Health Check
- On a user specified periodic basis, via a command line parameter setting, the mesh node should make a call to the health check URI's that was provided as part or the ```/Register``` call, to determine if the event sink or router is healthy. If the event sink or router returns an unhealthy state, the mesh node should remove the endpoint for the domain it belongs to.

- The healthy endpoint should return an HTTP Status code of 200 and the response body text should contain the text "healthy". An unhealthy endpoint should return an HTTP status code 503, and the text "unhealthy" in the response body.

- Given that there could be two or more concurrent registrations for a domain, and that these need to be merged and propagated back into the network. It is important that every healthCheck URI is called for a domain. If a domain has two different healthCheck URI's (after a merge) then the mesh should call both healthCheck URI's.

- When a health check fails for a {listener, healthCheck} pair, it should not immediately be removed from the list of "domainUris". A numeric integer counter for the number of retries should be associated with each pair, staring with a value of zero. A user specified threshold for the maximum number of retries must be specified as a command line parameter. When an "unhealthy" status is returned from the health check, then the counter is increased by one. When the counter is either equal to or exceeds the maximum number of retries, then it is to be removed from the "domainUris" list. When a counter for a {listener, healthCheck} pair is greater than zero but has not exceeded the maximum number of retries value, and a health check returns "healthy", then the counter for the pair must be reset to zero.
  
### Requirement 10 - Logging
- The system should log data when:
  1.  an event sink registers with the mesh network. It should log:
      - the action being performed which is "Registration"
      - source IP/Port of the registering node
      - the domain that it is registering
      - the list of listener URI's
      - the list of healthChecks URI's
  2. an event sink does a lookup. It should log:
      - the action being performed which is "Lookup"
      - the name of the domain it is looking for
      - whether or not the lookup was successful, by logging the listener endpoint URI's
  3. an event sink does a remove. It should log:
      - the action being performed which is "Remove"
      - the name of the domain it removing

## 5. System Interaction
<<Sequence Diagrams>>

## 6. Out of Scope (Negative Constraints)
- Event Router and Event Sink implementations
- Do NOT implement persistent database storage this service only routes data
- The management interface into the mesh is out of scope
- The ability to retrieve all domain endpoints from the mesh is out of scope

## 7. Core Domain Entities
- **Event Sink:** A process that listens on an IP/Port address for incoming event data. The processing of the event itself is determined by the purpose of the event sink. For example, an event sink could receive incoming event messages and persist them to a Kafka topic. 

- **Event Router:** Is an event sink, that will listen for incoming event data on and IP/Port. The event will be processed by the ruleset defined for that router.

- **Event Source:** A process that generates event data and will transmit event data to an event sink or event router.

- **Endpoint:** Is either an IPv4 or IPv6 address inclusive of the port. The endpoint contains an actual IP address and Port.

## 8. Error Handling & Edge Cases

