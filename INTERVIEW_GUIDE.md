# RideFlow — Complete Interview Explanation

## 1. Project title

**RideFlow: A ride-hailing backend built with Java and Spring Boot**

---

## 2. One-line explanation

RideFlow is a backend system for booking rides, finding nearby drivers, sending ride requests in real time, assigning a driver, and managing the booking review flow.

---

## 3. 30-second introduction

> RideFlow is a ride-hailing backend that I built using Java and Spring Boot. The main idea was to divide the application into separate services, where each service has one clear responsibility. The Auth Service handles passenger signup and login. The Booking Service creates and manages rides. The Location Service uses Redis GEO to find drivers near the passenger. The Socket Server sends ride requests to nearby drivers using WebSocket. Eureka helps the services discover each other. After the ride is completed, the Review Service handles the booking status and review process.

---

## 4. Why did I build this project?

I built this project to understand how a real-world ride-hailing backend works internally.

A ride-hailing application is not only a simple CRUD application. It needs to solve several problems:

- A passenger must be able to create an account and log in.
- A passenger must be able to request a ride.
- The system must find drivers close to the passenger.
- The ride request should reach drivers quickly.
- One driver should be assigned to the booking.
- The booking should move through different stages.
- The passenger or driver should be able to submit a review.

I wanted to learn how these different responsibilities can work together while still keeping the code organized.

---

## 5. What real-world problem does RideFlow solve?

The project solves the basic coordination problem between passengers, drivers, and the ride-booking system.

Without a system like RideFlow:

1. A passenger would not have a simple way to request a ride.
2. The platform would not know which drivers are nearby.
3. Drivers would not receive ride requests immediately.
4. The system would not have a clear way to track whether a ride is requested, scheduled, started, or completed.
5. Reviews would not be connected to completed rides.

RideFlow connects these steps into one backend flow.

---

## 6. Core idea behind the project

The core idea is to divide the ride-booking process into small services.

Each service owns one main responsibility:

| Service | Responsibility |
| --- | --- |
| Service Discovery | Helps services find each other |
| Auth Service | Handles passenger signup and login |
| Entity Service | Provides shared database models |
| Location Service | Stores driver locations and finds nearby drivers |
| Booking Service | Creates bookings and coordinates driver assignment |
| Socket Server | Sends real-time ride requests to drivers |
| Review Service | Handles booking status changes and reviews |

The goal was not to build a complete commercial Uber replacement. The goal was to build a smaller system that demonstrates the important backend ideas behind a ride-hailing application.

---

## 7. Simple architecture explanation

```text
Passenger Client
      |
      v
Auth Service --------------------> MySQL
      |
      v
Booking Service -----------------> MySQL
      |
      +----> Location Service ----> Redis GEO
      |
      +----> Socket Server --------> WebSocket/STOMP
                                      |
                                      v
                                Nearby Drivers

Review Service ------------------> MySQL

All services can find each other through Eureka Service Discovery.
```

In simple words:

- MySQL stores permanent information such as passengers, drivers, and bookings.
- Redis stores frequently changing driver coordinates.
- WebSocket is used to send ride requests to drivers immediately.
- Eureka stores the locations of running services.

---

## 8. Full end-to-end booking flow

### Step 1: Passenger signup

The passenger sends signup information to the Auth Service.

Example endpoint:

```text
POST /api/v1/auth/signUp
```

The service:

1. Validates the request.
2. Checks whether the email already exists.
3. Encrypts the password using BCrypt.
4. Creates a passenger account.
5. Stores the passenger in MySQL.

The password is not stored as plain text.

### Step 2: Passenger login

The passenger sends an email and password.

```text
POST /api/v1/auth/login
```

The service:

1. Finds the passenger by email.
2. Compares the entered password with the encrypted password.
3. Creates a JWT token.
4. Sends the token in an HTTP-only cookie.

For protected requests, the JWT filter reads the cookie and checks whether the token is valid.

### Step 3: Passenger creates a booking

The passenger sends the pickup location and destination.

```text
POST /api/v1/booking
```

Example request:

```json
{
  "passengerId": 101,
  "startLocation": {
    "latitude": 17.385,
    "longitude": 78.4867
  },
  "endLocation": {
    "latitude": 17.4435,
    "longitude": 78.3772
  }
}
```

Booking Service:

1. Validates the passenger ID and pickup coordinates.
2. Checks that the passenger exists.
3. Creates a booking.
4. Sets the initial status to `ASSIGNING_DRIVER`.
5. Saves the booking in MySQL.
6. Returns the booking ID to the passenger.

The booking is saved before the system starts contacting other services. This gives the ride a permanent ID even if a downstream service has a temporary problem.

### Step 4: Location Service finds nearby drivers

Booking Service asks Location Service to find drivers near the pickup point.

```text
POST /api/location/nearby/drivers
```

Location Service uses Redis GEO.

When a driver sends a location, the location is stored in Redis:

```text
driverId -> longitude, latitude
```

Redis uses:

- `x` as longitude
- `y` as latitude

The service searches within a configurable radius, which is 5 kilometres by default.

Example response:

```json
[
  {
    "driverId": "201",
    "latitude": 17.385,
    "longitude": 78.4867
  }
]
```

### Step 5: Booking Service sends nearby driver IDs

Booking Service takes the driver IDs returned by Location Service and sends them to Socket Server.

The important point is that the complete list of nearby driver IDs is passed forward. The socket service does not need to search for locations again.

### Step 6: Socket Server sends the ride request

Socket Server uses WebSocket and STOMP to send the ride request.

Each driver receives the message on a driver-specific topic:

```text
/topic/rideRequest/{driverId}
```

For example:

```text
/topic/rideRequest/201
```

This is better than sending the request to every connected driver because unrelated drivers should not receive a ride request that is outside their area.

### Step 7: Driver accepts the ride

The driver sends a response through:

```text
/app/rideResponse/{driverId}
```

The Socket Server receives the response and calls Booking Service through Eureka-discovered service communication.

### Step 8: Booking is scheduled

Booking Service:

1. Checks that the driver exists.
2. Finds the booking.
3. Assigns the driver.
4. Changes the booking status to `SCHEDULED`.
5. Returns a small driver summary instead of exposing the complete database entity.

### Step 9: Booking status changes

The Review Service demonstrates the later booking lifecycle.

A simple status flow is:

```text
ASSIGNED_DRIVER
        |
        v
CAB_ARRIVED
        |
        v
STARTED
        |
        v
IN_RIDE
        |
        v
COMPLETED
```

A booking can also be canceled from the allowed stages.

### Step 10: Review is created

After the booking is completed, a passenger review can be created.

```text
POST /api/v1/reviews
```

The service checks:

1. The booking exists.
2. The booking status is `COMPLETED`.
3. A review does not already exist.
4. The rating is between 0 and 5.
5. The review text is not empty.

This prevents a review from being added to an unfinished ride.

---

## 9. Why did I choose these technologies?

### Java and Spring Boot

I used Java and Spring Boot because they provide a clear structure for building backend services. Spring Boot makes it easier to create controllers, services, repositories, security filters, and database integrations.

### Spring Data JPA

I used JPA to work with MySQL using Java classes instead of writing every SQL query manually.

For example, `Passenger`, `Driver`, and `Booking` are represented as database entities.

### MySQL

MySQL stores permanent business data:

- Passenger accounts
- Driver details
- Bookings
- Booking status
- Reviews

### Redis GEO

Redis GEO is useful for finding nearby drivers. Driver locations change frequently, so Redis is a good place for quick location searches.

### WebSocket and STOMP

Normal REST APIs usually follow a request-and-response pattern. For driver notifications, the server needs to push a message to the driver. WebSocket supports this real-time communication.

STOMP gives the WebSocket messages a simple destination structure, such as:

```text
/topic/rideRequest/201
/app/rideResponse/201
```

### Eureka

Eureka helps one service find another service by its registered name. For example, Socket Server can find Booking Service without depending on a fixed localhost URL.

### Kafka

Kafka is included as a learning example for message-based communication. In the current project, it is not the main source of truth for booking status.

### Flyway

Flyway keeps database changes as versioned migration files. This makes database changes easier to track than changing the schema manually every time.

---

## 10. How the services communicate

| Communication | Example | Why it is used |
| --- | --- | --- |
| REST | Booking Service → Location Service | Request-response operation |
| REST | Socket Server → Booking Service | Assign the accepted driver |
| WebSocket/STOMP | Socket Server → Driver | Real-time ride notification |
| Redis command | Location Service → Redis | Nearby-driver search |
| Kafka | Kafka producer → Kafka consumer | Learning example for asynchronous messages |
| Eureka | Service → Eureka | Service discovery |

---

## 11. Important design decisions

### Why is the booking saved before driver search?

Saving first creates a stable booking ID. If the location or socket service fails, the booking can be marked as canceled or retried later instead of losing the request completely.

### Why are driver locations stored in Redis instead of only MySQL?

Driver locations may change many times. Redis is better suited for quick location searches, while MySQL is better for permanent business information.

### Why use separate services?

Separating services makes each responsibility easier to understand and change. For example, the location-search logic can evolve without changing the authentication code.

### Why use DTOs?

DTOs control the data exchanged between services and clients. They help avoid exposing complete database entities, passwords, relationships, or unnecessary internal fields.

### Why use an HTTP-only cookie for the JWT?

An HTTP-only cookie prevents normal browser JavaScript from reading the token. This is a safer default than storing the token in regular browser storage, although production applications need additional cookie and CSRF decisions.

---

## 12. Main challenges and how I solved them

### Challenge 1: Finding nearby drivers

The system needed to search by distance, not just retrieve all drivers.

**Solution:** I stored driver coordinates in Redis GEO and searched within a configurable radius.

### Challenge 2: Sending notifications in real time

A driver should not repeatedly ask the server whether a new ride is available.

**Solution:** I used WebSocket/STOMP so the server can push ride requests to driver-specific topics.

### Challenge 3: Connecting multiple services

Hardcoding service URLs becomes difficult when services move to different machines or ports.

**Solution:** I used Eureka so services can find each other by service name.

### Challenge 4: Keeping booking states meaningful

A booking should not jump randomly from one state to another.

**Solution:** Review Service defines allowed status transitions and only permits a review after `COMPLETED`.

### Challenge 5: Handling downstream failures

Location Service or Socket Server may be unavailable after the booking is created.

**Solution:** The improved Booking Service logs the failure and marks the booking as `CANCELED` when driver dispatch cannot continue.

### Challenge 6: Avoiding incorrect location results

Redis expects longitude first and latitude second, while users usually provide latitude first.

**Solution:** The service now converts the input clearly before storing and searching.

---

## 13. What I would say about my contribution

Use this only if it accurately describes your work:

> I worked on the backend design and implementation of the RideFlow platform. I created the service structure, REST APIs, authentication flow, booking flow, Redis-based driver search, and WebSocket driver communication. I also worked on database entities, validation, service-to-service communication, and the review workflow. While testing the flow, I found and fixed issues such as incorrect JWT filter routing, wrong Redis coordinate order, hardcoded service communication, and broadcasting ride requests to all drivers.

If an interviewer asks for one area in detail, choose the booking-to-driver flow because it connects the most parts of the project.

---

## 14. Best 2-minute explanation

> RideFlow is a ride-hailing backend built with Java and Spring Boot. I built it to understand how a real ride request moves through a distributed backend.
>
> The passenger first signs up and logs in through the Auth Service. The password is encrypted and login creates a JWT cookie. When the passenger requests a ride, Booking Service saves the booking with the status `ASSIGNING_DRIVER` and generates a booking ID.
>
> Booking Service then asks Location Service for nearby drivers. Location Service stores driver coordinates in Redis GEO and searches within a configured radius. It returns the nearby driver IDs. Booking Service passes those IDs to Socket Server, which sends a real-time request to each driver through WebSocket/STOMP.
>
> When a driver accepts, Socket Server calls Booking Service using Eureka service discovery. Booking Service assigns the driver and changes the booking to `SCHEDULED`. The Review Service handles later status changes such as `STARTED`, `IN_RIDE`, and `COMPLETED`, and allows a review only after the ride is completed.
>
> The main things I learned were service separation, JWT authentication, JPA relationships, Redis GEO search, WebSocket communication, Eureka discovery, and handling failures between services.

---

## 15. Best 30-second explanation

> RideFlow is a Spring Boot ride-hailing backend. Auth handles passenger login, Booking handles ride creation, Location uses Redis GEO to find nearby drivers, and Socket Server sends real-time ride requests to those drivers through WebSocket. When a driver accepts, the booking is updated. Eureka helps the services find each other, and Review Service manages the completed ride and review flow.

---

## 16. Questions the interviewer may ask

### Q1. Why did you choose a ride-hailing system?

**Answer:**

> It is a practical domain because it includes authentication, database operations, location search, real-time communication, service-to-service calls, status management, and reviews. It helped me learn several backend concepts through one connected use case.

### Q2. Why not build everything as one service?

**Answer:**

> A single service would be simpler to start with, but I wanted to understand how responsibilities can be separated. Authentication, location search, booking, and real-time communication have different concerns. Separating them makes each part easier to change and explain.

### Q3. Why Redis?

**Answer:**

> The main location question is “which drivers are near this coordinate?” Redis GEO is designed for this type of nearby search. Driver locations can change often, so Redis provides a quick lookup while MySQL stores permanent booking data.

### Q4. Why WebSocket instead of REST polling?

**Answer:**

> With REST polling, the driver would repeatedly ask whether a ride is available. WebSocket allows the server to push the ride request immediately, which is more suitable for real-time notifications.

### Q5. What happens if no driver is nearby?

**Answer:**

> The booking is marked as canceled when no nearby driver is found. A future improvement would retry with a larger radius or notify the passenger and ask them to try again.

### Q6. What happens if Location Service is down?

**Answer:**

> The booking is already saved, so it is not lost. The dispatch callback logs the failure and marks the booking as canceled. A production version could add retries and a timeout.

### Q7. How do you prevent two drivers from accepting the same booking?

**Answer:**

> The current learning version does not fully protect that race condition. My next improvement would be a transaction that updates the booking only if it is still in `ASSIGNING_DRIVER` or `SCHEDULED`-eligible state. If the update affects zero rows, another driver already won.

### Q8. Why use Eureka?

**Answer:**

> Eureka allows services to discover each other by name instead of depending on fixed host names and ports. It is useful when services run on different machines or when more than one instance exists.

### Q9. Why do services share one database?

**Answer:**

> I kept one MySQL database to make the learning project easier to run and understand. In a more mature microservice design, each service would own its data and communicate through clear APIs or events.

### Q10. What is the biggest limitation?

**Answer:**

> The project is not a production-ready ride platform yet. It does not include payments, map routing, driver authentication, strong retry handling, distributed tracing, or deployment automation. The core booking and driver-notification flow is implemented as a learning project.

### Q11. What did you do to protect passwords?

**Answer:**

> Passwords are encoded using BCrypt before being stored. The API does not store the original password. Login compares the entered password with the encoded database value.

### Q12. What is the purpose of DTOs?

**Answer:**

> DTOs define exactly what data crosses an API boundary. They prevent us from exposing complete entities and make the request and response formats easier to change safely.

---

## 17. Demo plan for an interview

Do not try to start every feature if the interview time is short. Demonstrate the most important path.

### Short demo

1. Open Location Service Swagger.
2. Save one driver coordinate.
3. Search for nearby drivers.
4. Show the returned driver ID.
5. Open Booking Service Swagger.
6. Create a booking near the same coordinate.
7. Explain that Booking Service passes the returned driver ID to Socket Server.
8. Show the WebSocket subscription topic.
9. Accept the ride.
10. Show the booking status as `SCHEDULED`.

### Full demo

1. Start MySQL.
2. Start Redis.
3. Start Kafka if demonstrating the Kafka example.
4. Start Eureka.
5. Start Location Service.
6. Start Socket Server.
7. Start Booking Service.
8. Start Auth Service.
9. Start Review Service if demonstrating reviews.
10. Publish the Entity Service to Maven Local before starting services that depend on it.

### What to show in Swagger

| Service | Swagger URL |
| --- | --- |
| Auth | `http://localhost:8080/swagger-ui.html` |
| Location | `http://localhost:7777/swagger-ui.html` |
| Booking | `http://localhost:8001/swagger-ui.html` |
| Review | `http://localhost:8081/swagger-ui.html` |
| Socket | `http://localhost:3002/swagger-ui.html` |
| Eureka | `http://localhost:8761` |

---

## 18. What I should not claim

Do not say the project has features that are not fully implemented.

Avoid saying:

- “This is production-ready.”
- “Kafka controls the complete booking flow.”
- “The system guarantees only one driver can accept.”
- “Every service has its own database.”
- “The system has complete payment and map integration.”
- “The project has full distributed tracing.”

Instead say:

> These are planned improvements. I focused first on understanding and implementing the core booking, location, authentication, and real-time notification flow.

---

## 19. Future improvements

### High priority

1. Add integration tests for the complete booking flow.
2. Prevent two drivers from accepting the same booking.
3. Add retry and timeout handling for service calls.
4. Add driver authentication.
5. Add Spring Boot Actuator health endpoints.

### Medium priority

1. Add API Gateway.
2. Give each service ownership of its database.
3. Add centralized logging.
4. Add metrics and tracing.
5. Add Docker Compose for local setup.

### Product features

1. Add fare calculation.
2. Add map routing and estimated arrival time.
3. Add ride cancellation.
4. Add driver availability updates.
5. Add payments.
6. Add notifications and ride history.

---

## 20. Learning plan

### First: understand the normal Spring Boot flow

Learn:

- Controller
- Service
- Repository
- Entity
- DTO
- Validation

Trace one request from the controller to the database.

### Second: understand authentication

Learn:

- Password hashing
- JWT creation
- JWT validation
- Security filter chain
- HTTP-only cookies

### Third: understand persistence

Learn:

- JPA relationships
- Lazy loading
- Transactions
- Flyway migrations

### Fourth: understand nearby-driver search

Learn:

- Longitude and latitude
- Redis GEO commands
- Radius searches
- Frequently changing location data

### Fifth: understand real-time communication

Learn:

- WebSocket connection
- STOMP destination
- Publish and subscribe
- Driver-specific topics

### Sixth: understand service discovery

Learn:

- Eureka registration
- Service names
- Finding an available service instance

### Seventh: understand reliability

Learn:

- Retries
- Timeouts
- Transactions
- Race conditions
- Idempotency

---

## 21. Final answer to memorize

> RideFlow is a ride-hailing backend that I built to understand how a real ride request is handled across multiple backend responsibilities. A passenger signs up and logs in through the Auth Service. When the passenger requests a ride, Booking Service saves the booking and asks Location Service to find nearby drivers using Redis GEO. The nearby driver IDs are sent to Socket Server, which sends real-time notifications through WebSocket/STOMP. When a driver accepts, Socket Server calls Booking Service through Eureka service discovery, and the booking is updated with that driver. Review Service then handles the booking status flow and allows a review after completion.
>
> The main things I learned from this project were Spring Boot service design, JWT authentication, JPA and MySQL, Redis nearby search, WebSocket communication, Eureka service discovery, validation, and failure handling between services. The project is a learning version, so payment, map routing, production deployment, and advanced concurrency protection are future improvements.

---

## 22. One final interview tip

Do not try to explain every file.

Explain one complete story:

```text
Passenger logs in
        ↓
Passenger creates booking
        ↓
Location Service finds nearby drivers
        ↓
Socket Server notifies selected drivers
        ↓
Driver accepts
        ↓
Booking becomes scheduled
        ↓
Ride completes
        ↓
Review is created
```

If you can explain this flow clearly, explain why each technology was chosen, and honestly describe the limitations, you can present the project confidently.
