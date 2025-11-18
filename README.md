🎫 Scalable Event Ticketing & Seat Allocation System

Project Overview

This project implements a highly available, high-performance system for selling tickets and allocating seats for large-scale events. The architecture is specifically designed to handle spiky, high-contention traffic typical of flash sales while guaranteeing strong consistency to prevent overselling of seats.

The core challenge addressed is managing the atomic state transition of seats (Available -> Reserved -> Sold) across distributed services.

✨ Key Design Goals (NFRs)

Requirement

Target Metric

Rationale & Implementation

Scalability

Peak RPS > 100,000

Achieved via stateless microservices and Geo-Sharding of the seat database.

Performance

P99 Checkout < 2s

Mandates aggressive caching (Redis) of seat maps and low-latency distributed SQL (CockroachDB).

Consistency

Strong Consistency

Essential for seat status updates to guarantee zero oversells (ACID transactions at the database level).

Availability

99.95% Uptime

Requires multi-region deployment and robust failover mechanisms.

📐 System Architecture

The system is decomposed into several decoupled microservices, communicating via synchronous API calls (for critical path) and asynchronous messaging (for fulfillment and notifications).

Core Service Domains

Seat Service (Write-Critical): Manages the seat inventory, reservation locks, and the atomic state transition.

Primary Database: CockroachDB (Distributed SQL) for strong consistency and write scalability via sharding.

Cache: Redis Cluster for reading seat availability status (offloading >95% of read traffic).

Order Service: Handles order persistence, final pricing, and fulfillment logic.

Payment Service: Orchestrates communication with the External Payment Gateway and enforces Idempotency checks.

Event Service: Manages event metadata, listings, and venue details (Read-Heavy).

Data Flow Backbone

Asynchronous Communication: Kafka is used as the central message queue for inter-service communication and load leveling.

Key Topics: TICKET_COMMITTED, SEAT_RESERVED, SEAT_RELEASED.

🔒 Critical Path: Seat Reservation Flow

The checkout process uses a two-step commit process with explicit locking to prevent oversells:

Phase 2: Reserve (Atomic Lock):

Client calls POST /v1/seats/reserve.

The Seat Service initiates a Database Transaction (Available -> Reserved). This lock is enforced by the database's isolation level.

A Time-To-Live (TTL) is set on the reservation (e.g., 5-10 minutes).

Phase 4: Commit (Atomic Write):

Called by the Payment Service after successful external payment.

The Seat Service verifies the reservation is still Reserved (and not expired).

It initiates a final Database Transaction (Reserved -> Sold).

On successful commit, it publishes the TICKET_COMMITTED event to Kafka.

💾 Data Sharding Strategy

To handle the high write throughput on the Seat Service, data is horizontally sharded:

Database: CockroachDB.

Sharding Key: The primary index for the Seat and Reservation tables is built on (event_id, seat_id).

Benefit: All read and write operations related to a single high-contention event are routed to the same physical database shard, minimizing cross-shard latency during a flash sale.

Mitigation: For major events, ranges are pre-split to distribute the event's load across multiple CockroachDB nodes before the sale begins.

🛠️ Maintenance & Future Work

Resilience Measures

Oversell Prevention: Guaranteed by atomic database transactions and strong consistency locks.

Reservation Cleanup: A dedicated Reservation Cleaner background worker periodically checks for and releases expired reservations, publishing a SEAT_RELEASED event.

Idempotency: Implemented via an Idempotency Key check in both the Payment Service and Seat Service before any transaction execution, preventing double charges or double commits from client retries.

Future Enhancements

Dynamic Pricing Engine: Implement a separate service to adjust seat prices in real-time based on demand and inventory levels.

Ticket Transfer/Resale: Implement a secure ledger or mechanism for verified users to transfer tickets.

Anti-Bot Integration: Add specialized filtering services at the API Gateway to block malicious or aggressive automated traffic.
