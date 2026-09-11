# System Design & Specification Document: Domain Endpoint Resolver
>[!NOTE]
This document acts as the absolute source of truth for the AI. Where the PRD defines the behavior, this document defines the structure. Use strict types, explicit file paths, and exact payload schemas. AI models parse schemas and tables much better than paragraph descriptions.

# 1. Architectural Boundaries & Bounded Contexts
> [!IMPORTANT]
> [Define the strict domains. This prevents the AI from leaking logic between different parts of the system.]

- Domain 1 (e.g., Ingestion): Responsible strictly for terminating network connections (e.g., MQTT/UDP), parsing raw bytes, and yielding DTOs.
- Domain 2 (e.g., Core Engine): Responsible strictly for applying business rules to validated entities. Has no knowledge of the network layer.
- Domain 3 (e.g., Egress): Responsible strictly for serializing internal entities back to external contracts (e.g., gRPC) and transmitting.

## 2. Directory & Module Structure
>[!IMPORTANT]
[Force the AI to use a specific file tree. If you do not provide this, the AI will default to a flat directory or a framework's default structure, making it hard to apply Domain-Driven Design.]

```
src/
├── ingestion/          # Network termination and parsing
│   ├── mod.rs
│   ├── mqtt_handler.rs # Network listener
│   └── parser.rs       # Uses nom to parse raw string/byte inputs
├── domain/             # Core business logic (No network/IO code allowed here)
│   ├── mod.rs
│   ├── entities.rs     # Strict domain structs 
│   └── rules.rs        # State transitions and validations
├── adapter/            # Interface contracts
│   ├── mod.rs
│   └── dtos.rs         # Data Transfer Objects
└── main.rs             # Dependency injection and application bootstrapping
```
## 3. Data Models & Interface Contracts
> [!IMPORTANT]
> [Never let the AI guess the data structures. Define them as strict schemas. Explicitly forbid flexible schema anti-patterns like Entity-Attribute-Value (EAV) models; force strict relational or strongly-typed object structures.]

### A. External Data Transfer Objects (DTOs)
[The exact shape of data entering or leaving the system.]

Target Payload Name: TelemetryIngestDTO
Protocol/Format: JSON over MQTT

Schema:
```
{
  "sensor_id": "string (UUID)",
  "timestamp": "i64 (Unix epoch milliseconds)",
  "metrics": {
    "temperature": "f64",
    "voltage": "f64"
  }
}
```

### B. Internal Domain Entities
> [!IMPORTANT]
> [The strictly typed representation used by your business logic, separated from the DTO.]

- Entity Name: SensorReading Attributes:
id: UUID
- recorded_at: DateTime
- temp_celsius: f64
- status: Enum (Nominal, Warning, Critical)

### C. The Mapping Contract
Rule: The AI must implement a strict mapping function (TryFrom or equivalent) that converts TelemetryIngestDTO into SensorReading. If metrics.temperature is missing, the mapping must return a specific validation error, not a generic panic.

## 4. API & Network Definitions
> [!IMPORTANT]
> [Specify the exact protocols, header limits, and connection behaviors.]

- Ingestion Endpoint: tcp://0.0.0.0:1883 (MQTT)
- Egress Endpoint: gRPC service defined in telemetry_v1.proto.
- Performance Constraints: The system must process inputs under strict header overhead and bandwidth constraints. Avoid wrapping payloads in heavy application-layer metadata where a lighter protocol approach is preferred.

## 5. Execution Flow & State Machine
> [!IMPORTANT]
> [Provide a step-by-step logical flow for a single transaction. This is the recipe the AI uses to write the main function or controller.]

- Receive: Network handler receives a byte array on the MQTT topic sensors/data/#.
- Parse: Route the byte array to the parser module.
- Validate DTO: Confirm all required fields exist. Return an InvalidPayload error if missing.
- Map to Entity: Convert the DTO to the SensorReading domain entity. Apply the rule to determine the status enum based on the temp_celsius value.
- Dispatch: Pass the valid SensorReading entity to the egress service for gRPC serialization.
