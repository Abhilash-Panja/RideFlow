# RideFlow

RideFlow is a ride-hailing platform built as a set of independent Spring Boot microservices, communicating through service discovery, event streaming, and shared data contracts. This repo is the entry point — it links to each service and shows how they fit together.

## Architecture

```
                        ┌─────────────────────┐
                        │   Service-Discovery  │  (Eureka, :8761)
                        └──────────┬───────────┘
                                   │ registers with
        ┌───────────────┬─────────┼─────────────┬───────────────┐
        │               │         │             │               │
┌───────▼──────┐ ┌───────▼─────┐ ┌▼────────────┐ ┌▼─────────────┐
│ Auth Service │ │  Location   │ │  Booking     │ │ Socket Server│
│   :8080      │ │  Service    │ │  Service     │ │   :3002      │
│              │ │   :7777     │ │   :8001      │ │              │
└──────┬───────┘ └──────┬──────┘ └──────┬───────┘ └──────┬───────┘
       │                │               │                │
       │         ┌──────▼──────┐        │         ┌──────▼──────┐
       │         │   Review    │        │         │    Kafka    │
       │         │   Service   │        │         │  (events)   │
       │         │   :8081     │        │         └─────────────┘
       │         └─────────────┘        │
       │                                │
       └────────────────┬───────────────┴──────────┐
                         │                          │
                  ┌──────▼──────┐            ┌──────▼──────┐
                  │    MySQL    │            │    Redis    │
                  └─────────────┘            └─────────────┘

  All services share domain models via Rideflow-EntityService (published
  as a versioned library, not run as its own network service).
```

## Services

| Service | Repo | Port | Responsibility |
|---|---|---|---|
| Service Discovery | [Service-Discovery](https://github.com/Abhilash-Panja/Service-Discovery) | 8761 | Eureka registry so services find each other by name instead of hardcoded URLs |
| Auth Service | [RideFlow-AuthService](https://github.com/Abhilash-Panja/RideFlow-AuthService) | 8080 | User authentication, JWT issuing, session cookies |
| Entity Service | [Rideflow-EntityService](https://github.com/Abhilash-Panja/Rideflow-EntityService) | — (library) | Shared domain models/entities consumed by other services |
| Location Service | [Location-Service](https://github.com/Abhilash-Panja/Location-Service) | 7777 | Tracks and serves real-time driver/rider location data |
| Booking Service | [Booking-Service](https://github.com/Abhilash-Panja/Booking-Service) | 8001 | Ride booking lifecycle; publishes events to Kafka, caches via Redis |
| Review Service | [UberReviewService](https://github.com/Abhilash-Panja/UberReviewService) | 8081 | Ride ratings and reviews |
| Socket Server | [Socket-Server](https://github.com/Abhilash-Panja/Socket-Server) | 3002 | Real-time updates to clients via WebSockets, driven by Kafka events |

## Tech stack

- **Language/Framework:** Java 17, Spring Boot
- **Service discovery:** Netflix Eureka (Spring Cloud)
- **Database:** MySQL
- **Caching:** Redis
- **Messaging:** Apache Kafka
- **Build:** Gradle, with a shared entity library distributed via Maven

## Running the full platform locally

This repo doesn't contain the service source code itself — each service lives in its own repo (see table above) so it can be built, tested, and deployed independently. To run everything together for a demo:

1. Clone this repo and each service repo as sibling folders:
   ```bash
   git clone https://github.com/Abhilash-Panja/RideFlow-Platform.git
   cd RideFlow-Platform
   git clone https://github.com/Abhilash-Panja/Service-Discovery.git
   git clone https://github.com/Abhilash-Panja/RideFlow-AuthService.git
   git clone https://github.com/Abhilash-Panja/Rideflow-EntityService.git
   git clone https://github.com/Abhilash-Panja/Location-Service.git
   git clone https://github.com/Abhilash-Panja/Booking-Service.git
   git clone https://github.com/Abhilash-Panja/UberReviewService.git
   git clone https://github.com/Abhilash-Panja/Socket-Server.git
   ```

2. Publish the shared entity library to your local Maven cache (other services resolve it from there):
   ```bash
   cd Rideflow-EntityService && ./gradlew publishToMavenLocal && cd ..
   ```

3. Enable Docker BuildKit (needed for the Maven-cache mount during builds):
   ```bash
   export DOCKER_BUILDKIT=1        # Linux/macOS
   $env:DOCKER_BUILDKIT=1          # Windows PowerShell
   ```

4. Start everything:
   ```bash
   docker compose up --build
   ```

5. Once containers are healthy, check the Eureka dashboard at `http://localhost:8761` — you should see all services registered.

## Notes on the shared entity library

Several services depend on `com.rideflow:Rideflow-EntityService` for shared models. Keep the version consistent across consumers when you bump it — check each service's `build.gradle` for the current pin.
