# Product Requirements Document (PRD): Domain Endpoint Resolver
[!IMPORTANT] When feeding this into an AI context window, keep the language imperative and literal. Avoid descriptive marketing speak. The AI needs strict rules, not persuasion.

## 1. System Objective
> [!IMPORTANT]
> [Provide a 1-2 sentence description of the exact system to be built. Example: "A Rust-based edge service that ingests high-frequency sensor data via MQTT, normalizes the payload, and routes it to a central event stream."]

## 2. Tech Stack & Architectural Constraints
>[!IMPORTANT] 
[Explicitly lock in the boundaries so the AI does not hallucinate frameworks or make unwanted design choices.]

- Primary Language: [e.g., Rust, strict adherence to idiomatic error handling]
- Architecture Pattern: [e.g., Event-Driven Architecture utilizing Domain-Driven Design principles]

- Communication Protocols: [e.g., MQTT for ingestion, gRPC for internal microservice communication]

- Key Dependencies: [e.g., nom for string parsing, tokio for async runtime]

## 3. Functional Requirements
>[!IMPORTANT]
[Define the exact capabilities. Format as concrete rules or strict input/output transformations.]

- **Requirement 1 (Ingestion):** The system must accept incoming connections on [Port/Endpoint] and maintain a continuous listening state.
- **Requirement 2 (Validation):** The system must validate that the incoming Data Transfer Object (DTO) contains all required fields before processing.
- **Requirement 3 (Transformation):** The system must parse the raw input and map it to the internal domain entity.
- **Requirement 4 (Output):** The system must publish the validated payload to the designated downstream service or topic.

## 4. Out of Scope (Negative Constraints)
> [!IMPORTANT]
> [Crucial for AI: List exactly what the AI should NOT build to prevent over-engineering and token waste.]
- Do NOT build a frontend or UI component.
- Do NOT implement persistent database storage (this service only routes data
- Do NOT write authentication logic (assume payloads are authenticated at the gateway layer).
- Do NOT include dynamic configuration management (use static environment variables for now).

## 5. Core Domain Entities
> [!IMPORTANT]
> [Define the primary nouns of your system. This sets up the AI to accurately generate your core structs and domain logic without leaking concerns.]
- Entity 1: [Entity Name] (Attributes: id (type), timestamp (type), value (type)).
- Entity 2: [Entity Name] (Attributes: code (type), severity_level (enum)).

## 6. Error Handling & Edge Cases
>[!IMPORTANT]
[Specify exactly how the system should fail gracefully.]

If a DTO is malformed, log a structured warning containing the raw payload and drop the packet. Do not panic or crash the thread.

If the downstream network connection is unavailable, implement an exponential backoff retry mechanism (max 3 attempts) before discarding the message.
