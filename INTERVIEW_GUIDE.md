# RideFlow interview guide

RideFlow is a ride-hailing backend built with Java and Spring Boot. A passenger can create an account, request a ride, and later receive a driver assignment. Drivers publish their location, and the platform looks for drivers close to the passenger before sending a real-time ride request.

## The short explanation

> I built a ride-hailing backend as a group of small Spring Boot services. Authentication handles passenger login with a JWT cookie. Booking stores the ride and coordinates the search for a driver. Location keeps driver coordinates in Redis so nearby drivers can be found quickly. Socket Server uses WebSocket/STOMP to notify selected drivers in real time. Review Service keeps the booking status flow and allows one review after a ride is completed. Eureka helps the services find each other without hardcoding every host name.

## Why the services are separated

Each service has one clear responsibility:

| Service | Simple responsibility |
| --- | --- |
| Service Discovery | Keeps a list of running services |
| Auth Service | Signs up passengers and validates login tokens |
| Entity Service | Publishes the shared database models as a small library |
| Location Service | Saves driver coordinates and searches Redis for nearby drivers |
| Booking Service | Creates the booking and starts driver assignment |
| Socket Server | Sends a ride request to the selected drivers over WebSocket |
| Review Service | Demonstrates booking status changes and passenger reviews |

This is a learning project, so the services currently share one MySQL database. That makes local setup easier, but a production system would normally give each service ownership of its own data.

## Main request flow

1. The passenger signs up or logs in through Auth Service.
2. The passenger sends a booking request with a pickup coordinate.
3. Booking Service saves the booking as ASSIGNING_DRIVER.
4. Booking Service asks Location Service for drivers within the configured radius.
5. Location Service searches the Redis GEO set.
6. Booking Service sends the nearby driver IDs to Socket Server.
7. Socket Server publishes the request to /topic/rideRequest/{driverId} for each selected driver.
8. A driver sends an acceptance message to /app/rideResponse/{driverId}.
9. Socket Server asks Booking Service to assign that driver.
10. Booking Service changes the ride to SCHEDULED.

The important design choice is that creating a booking returns quickly. Driver lookup and notification happen after the booking is saved, so the passenger is not forced to wait for every downstream service before receiving a booking ID.

## What I would demonstrate

Use Swagger for Auth, Booking, and Review, and a small STOMP client for the socket flow.

A simple demo sequence is:

1. Start MySQL, Redis, Kafka, and Eureka.
2. Publish Rideflow-EntityService to Maven Local.
3. Start Location Service, Socket Server, Booking Service, and Auth Service.
4. Add one driver location to Redis.
5. Create a passenger and log in.
6. Create a booking near the saved driver coordinate.
7. Subscribe as that driver to /topic/rideRequest/{driverId}.
8. Accept the ride through /app/rideResponse/{driverId}.
9. Read the booking and show that it is now scheduled.

For a shorter interview demo, show the Location Service independently first. Save a coordinate, search from the same area, then explain that Booking Service uses those driver IDs to start the real-time part.

## Improvements included in the interview-ready branch

- Fixed the JWT filter path logic so login and signup are public, while /validate actually checks the JWT.
- Added stateless security and JSON responses for unauthenticated and forbidden requests.
- Added request validation for authentication, location, booking callback, and review inputs.
- Removed the hardcoded JWT secret and database/service settings from the main configuration path; local defaults remain available for learning.
- Fixed Redis GEO coordinate order: Redis uses longitude as x and latitude as y.
- Made the search radius configurable.
- Passed nearby driver IDs from Booking Service to Socket Server.
- Marked a booking as canceled when location lookup or socket notification fails.
- Sent ride requests only to selected driver topics instead of broadcasting to every driver.
- Replaced the hardcoded Socket Server to Booking Service URL with Eureka lookup.
- Replaced Optional wrappers in request/response JSON DTOs with normal JSON fields.
- Added a unique database constraint for passenger email.
- Added validation and clear 400 errors to the review workflow.
- Removed generated JVM crash dumps from the Review Service repository.
- Moved the changes into review branches so they can be tested and merged one service at a time.

## What is intentionally still simple

These are good next steps, but I would not claim them as finished in an interview:

- The services still share the same MySQL database.
- The socket server uses a simple in-memory WebSocket broker.
- A driver's acceptance is not yet protected by a database lock, so two drivers could race to accept the same booking.
- The first available driver in Review Service is a demo placeholder; the distributed flow uses Redis nearby search.
- Kafka is currently a small messaging demonstration, not the source of truth for booking state.
- There is no payment, map routing, cancellation timeout, driver authentication, or production deployment pipeline yet.

Being clear about these boundaries makes the project more believable. You can say what you built, what trade-off you made, and what you would do next.

## Questions you should be ready for

### Why Redis for driver locations?

Driver coordinates change often, and the main operation is “find members near this point.” Redis GEO is a simple fit for that operation. MySQL remains useful for durable booking and account data.

### Why save the booking before searching?

The booking ID gives the request a durable identity. If a downstream service is temporarily unavailable, the booking can be marked canceled or retried instead of losing the request.

### What happens if no driver is found?

The improved flow marks the booking as CANCELED and logs the reason. A future version could retry with a larger radius or notify the passenger.

### Why use WebSocket?

A driver should receive a new ride request without repeatedly asking the server for updates. WebSocket lets the server push the event to the driver's open connection.

### What would you improve next?

I would add integration tests for the complete booking flow, protect driver acceptance with a transaction and a status check, add Actuator health endpoints, and add a small Docker Compose setup for local development.

## Learning order

1. Spring Boot controllers, services, repositories, and validation.
2. Spring Security and JWT authentication.
3. JPA relationships and Flyway migrations.
4. Redis GEO commands and coordinate systems.
5. WebSocket/STOMP request and subscription paths.
6. Eureka service discovery and service-to-service calls.
7. Kafka basics and when asynchronous events are useful.
8. Integration testing with Testcontainers.

Learn each topic by tracing one request through the project. For example, follow POST /api/v1/booking from the controller, into the service, into the repository, then into Location Service and Socket Server.
