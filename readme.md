# Node.js Social Media Backend (Microservices)

A microservices-based backend for a social media–style application built with Node.js, Express, Docker, and RabbitMQ. Each service is fully isolated with its own models, controllers, middleware, utilities, and Docker configuration.

---

## Architecture Overview

The system is composed of four main backend services plus an API gateway, all orchestrated via Docker Compose. 

- **API Gateway (`api-gateway`)**
  - Single entry point for clients.
  - Handles routing, authentication, and centralized error handling.
  - Uses `authMiddleware.js`, `errorHandler.js`, and `logger.js` in `src/middleware` and `src/utils`. 

- **Identity Service (`identity-service`)**
  - Manages user registration, login, and authentication-related operations.
  - `controllers/identity-controller.js` contains the core auth logic.
  - `models/` holds user-related models.
  - `utils/generateToken.js`, `validation.js`, and `logger.js` provide JWT generation, request validation, and logging.
  - `routes/identity-service.js` exposes identity-related HTTP endpoints. 

- **Post Service (`post-service`)**
  - Handles CRUD operations for posts.
  - `models/Post.js` defines the post schema.
  - `controllers/post-controller.js` implements business logic.
  - `routes/post-routes.js` maps HTTP routes to controllers.
  - `middleware/authMiddleware.js` protects post routes; `errorHandler.js` standardizes error responses.
  - `utils/rabbitmq.js`, `validation.js`, and `logger.js` support messaging, validation, and logging. 

- **Media Service (`media-service`)**
  - Responsible for media upload and processing.
  - `eventHandlers/media-event-handlers.js` consumes media-related events (e.g., from RabbitMQ).
  - `models/` contains media-related persistence models.
  - `utils/cloudinary.js`, `rabbitmq.js`, and `logger.js` integrate with Cloudinary, messaging, and logging.
  - `routes/` and `controllers/` expose and implement media APIs. 

- **Search Service (`search-service`)**
  - Provides search functionality over posts (and potentially users/media).
  - `models/Search.js` defines search-related data structures.
  - `controllers/search-controller.js` implements search logic.
  - `eventHandlers/search-event-handlers.js` listens to message-broker events to update search indexes.
  - `middleware/authMiddleware.js` and `errorHandler.js` secure and normalize search requests.
  - `utils/rabbitmq.js` and `logger.js` handle messaging and logging. 

Each service has its own `server.js`, `package.json`, `.env`, `Dockerfile`, and `src/models` folder, aligning with the “database-per-service” microservices pattern. 

---


## Technologies Used

This project combines several backend, infrastructure, and integration technologies to implement a realistic microservices-based social platform.  

- **Runtime & Language**
  - Node.js + Express in each service (`api-gateway`, `identity-service`, `post-service`, `media-service`, `search-service`). 
  - JavaScript for all application code (controllers, models, middleware, utils). 

- **API & Middleware**
  - Custom `authMiddleware.js` in gateway, post, search, etc. for JWT-based authentication and route protection. 
  - Centralized `errorHandler.js` in each service for consistent error responses. 
  - Validation utilities (`validation.js`) for request payload checks. 

- **Data & Models**
  - Per-service `models/` folder (e.g., `Post.js`, `Search.js`, user model in identity service) following database-per-service design. 

- **Messaging & Event-Driven Components**
  - RabbitMQ integration in `utils/rabbitmq.js` (post-service, media-service, search-service) to publish and consume domain events such as new posts or media updates.  

- **Media & Cloud Integration**
  - Cloudinary integration via `utils/cloudinary.js` in the media service for file storage and CDN-style media delivery.  

- **Infrastructure & DevOps**
  - Docker and per-service `Dockerfile` for containerization. 
  - `docker-compose.yml` for orchestrating multiple services, networks, and dependencies in local development. 
  - Service-specific `.env` files for configuration and secrets. 

- **Logging & Observability**
  - Custom `logger.js` utility in each service for structured logging and debugging. 

---

## System Design Concepts

The project intentionally applies several modern system-design patterns to mirror real-world backend systems. 

### Microservices & Bounded Contexts

- Each domain (identity, posts, media, search) is implemented as an independent microservice with its own controllers, models, middleware, utilities, and configuration. 
- Services communicate over HTTP through the API Gateway and over **asynchronous messages** using RabbitMQ for cross-service events.  

### API Gateway Pattern

- The `api-gateway` service acts as the single public entry point, routing incoming requests to the correct internal service.  
- It centralizes:
  - Authentication and token verification (`authMiddleware.js`).
  - Global error handling (`errorHandler.js`).
  - Logging of external requests (`logger.js`). 

### Database-Per-Service & Encapsulated Models

- Each service owns its models in its own `models/` directory (e.g., `Post.js`, `Search.js`, user model), which reflects the **database-per-service** pattern.  
- This keeps data schemas isolated and aligns with bounded contexts, reducing coupling between services.

### Event-Driven Architecture

- The post, media, and search services integrate with RabbitMQ to implement an **event-driven** workflow.  
  - The Post Service publishes events (via `utils/rabbitmq.js`) when posts are created or updated.
  - Media and Search services subscribe and react via `media-event-handlers.js` and `search-event-handlers.js`, updating media metadata or search indexes without synchronous coupling.  
- This improves scalability and resilience because services only need the broker to be available, not each other directly.

### Separation of Concerns & Layered Design

- Each service follows a clean layering:
  - **Routes**: Define HTTP endpoints and map them to controllers.
  - **Controllers**: Orchestrate business logic and validation.
  - **Models**: Encapsulate data layer and schemas.
  - **Middleware**: Cross-cutting concerns like auth and error handling.
  - **Utils**: Shared helpers such as logging, JWT generation, Cloudinary, RabbitMQ, and validation.  
- This structure makes the codebase easier to maintain, test, and extend with additional services or features.

---

## Running the Project

1. **Clone the repository**
git clone https://github.com/VISHALGAURAV95/backend_microservices.git
cd backend_microservices

text

2. **Configure environment variables**
- Duplicate the `.env` templates in each service (`api-gateway`, `identity-service`, `post-service`, `media-service`, `search-service`).
- Fill in values such as database URLs, JWT secrets, RabbitMQ URL, and Cloudinary credentials where required. 

3. **Start all services with Docker**
docker-compose up --build

text

4. **Access the API Gateway**
- Open the gateway base URL (for example `http://localhost:<GATEWAY_PORT>`) and hit routes like:
  - `POST /auth/register` or `POST /auth/login` (proxied to identity-service).
  - `POST /posts` and `GET /posts` (proxied to post-service).
  - `POST /media/upload` (proxied to media-service).
  - `GET /search` (proxied to search-service). 

---

## Request Flow Example

- A client sends a request to the **API Gateway** with a JWT; `authMiddleware.js` validates the token and forwards the request. 
- The **Post Service** creates a post via `post-controller.js`, persists it with `Post.js`, and emits an event using `rabbitmq.js`. 
- The **Media Service** or **Search Service** consumes that event through their `*-event-handlers.js` files to store media metadata or update search indexes. 
- Logs are recorded by each service’s `logger.js`, and errors are normalized by shared-style `errorHandler.js` middleware. 

---

## Possible Extensions

- Add per-service databases (MongoDB/PostgreSQL) wired via `docker-compose`.
- Implement richer search capabilities (e.g., full-text or fuzzy search).
- Add role-based authorization in the identity service and enforce it in the gateway.
- Integrate monitoring and tracing (Prometheus, Grafana, OpenTelemetry) for cross-service observability. 
