# Movie Recommendation Platform

A personalized movie recommendation platform that generates recommendations based on user preferences and feedback.

The primary business goal is to provide users with relevant movie suggestions by analyzing their likes, dislikes, and explicit recommendation preferences.

The system uses a microservice-based architecture and integrates with **GigaChat** to generate personalized recommendations.

## System Architecture

![System Architecture](architecture.png)

The platform consists of several independent services responsible for authentication, movie delivery, user feedback, recommendation generation, and data processing.

### User Flow

The typical user journey looks as follows:

1. **Registration and Authentication** — a user registers, logs in, and receives an access token.
2. **Browse Recommendations** — the user opens the main page and receives a list of recommended movies from the Async API Service.
3. **Provide Feedback** — the user can like or dislike a movie, request more movies similar to a particular one, or explicitly exclude a movie from future recommendations.
4. **Generate Recommendations** — the recommendation pipeline processes user preferences and generates personalized suggestions.
5. **Receive Updated Recommendations** — the user receives recommendations based on their preferences and available movie metadata.

### Service Interactions

Authentication and authorization are handled by the Auth Service. Other services communicate with it to validate user permissions and associate requests with the corresponding user.

The UGC Service (User-Generated Content) is responsible for storing user feedback, including likes, dislikes, and explicit recommendation preferences.

The Recommendation Service processes this feedback and generates personalized recommendations using GigaChat. Since LLM requests are relatively expensive and time-consuming, recommendation generation is performed periodically rather than on every user request.

Generated recommendations are stored in the Recommendation Service database.

When a user requests recommendations, the Async API Service retrieves the relevant data and matches recommended movies against the movie catalog stored in Elasticsearch.

## Microservices and Infrastructure

The application is orchestrated using Docker Compose.

The current local environment consists of 14 application and infrastructure containers, excluding the additional test container.

| Container | Responsibility |
|---|---|
| `etl_transporter` | Transfers the initial movie dataset from SQLite to PostgreSQL. |
| `etl_db` | PostgreSQL database used by the ETL pipeline to store movie data. |
| `etl` | Executes the movie data extraction, transformation, and loading pipeline. |
| `etl_es` | Elasticsearch instance containing the movie catalog. |
| `auth_service` | Handles user registration, authentication, and authorization. |
| `postgres_auth` | PostgreSQL database for the Auth Service. |
| `redis_async_api` | Redis instance used by the Async API Service. |
| `async_api_service` | Main API responsible for serving movie data and recommendations to users. |
| `postgres_recommendation` | PostgreSQL database storing generated recommendations. |
| `scheduler` | Periodically executes the recommendation generation pipeline. |
| `nginx` | Reverse proxy and entry point for incoming requests. |
| `like_service` | UGC Service responsible for storing user feedback and preferences. |
| `mongodb` | MongoDB instance used by the UGC Service. |

## Recommendation Generation Pipeline

The current implementation uses a dedicated `scheduler` container that continuously executes `fetch_recommendations.py`.

The script periodically performs the following operations:

1. **Collect user feedback**

   Requests the UGC Service to retrieve likes, dislikes, and other user preferences.

2. **Retrieve movie metadata**

   Sends movie UUIDs to the Async API Service to resolve them into movie titles and other relevant information.

   The corresponding endpoint is configured through `ASYNC_API_URL_TITLE_BY_UUIDS`.

3. **Generate recommendations**

   Sends the collected preferences and movie metadata to GigaChat and receives personalized movie recommendations.

4. **Persist recommendations**

   Stores the generated results in the Recommendation Service database.

5. **Serve recommendations**

   The Async API Service is intended to retrieve pre-generated recommendations from the Recommendation Service database. If no generated recommendations are available, it falls back to the existing IMDb-based recommendation mechanism.

### Scheduling Strategy

Recommendation generation is an expensive operation due to the latency and cost of LLM requests.

For this reason, recommendations are generated periodically rather than synchronously during user requests.

The current approach favors implementation simplicity and is sufficient for an MVP, although it has scalability and efficiency limitations.

## Data Processing (ETL)

The project includes a dedicated ETL pipeline responsible for preparing the movie catalog.

Its main components are:

- **SQLite** — source of the initial movie dataset.
- **PostgreSQL** — intermediate storage used during data processing.
- **Elasticsearch** — search and retrieval engine containing the processed movie catalog.

The `etl` service handles data processing, while `etl_transporter` is responsible for transferring the initial dataset into PostgreSQL.

## Technology Stack

The project uses the following technologies:

| Technology | Purpose |
|---|---|
| Python | Backend development and data processing |
| FastAPI | REST API development |
| PostgreSQL | Relational data storage |
| MongoDB | NoSQL storage for user-generated content |
| Redis | Caching and supporting Async API operations |
| Elasticsearch | Movie catalog search and retrieval |
| Docker & Docker Compose | Containerization and service orchestration |
| Nginx | Reverse proxy |
| GigaChat | LLM-based recommendation generation |

## Project Structure

```text
.
├── README.md
├── architecture.png
├── async_api_service/
├── auth_service/
├── etl_service/
├── recommendation_service/
├── scheduled_tasks/
├── ugc_service/
├── nginx/
├── docker-compose.yml
├── docker-compose.override.prod.yml
├── docker-compose.override.test.yml
├── dev_run.sh
├── local_run.sh
├── mypy.ini
└── setup.cfg
```

### Main Components

- **`async_api_service/`** — API responsible for delivering movie data and recommendations.
- **`auth_service/`** — authentication and authorization.
- **`ugc_service/`** — user feedback and recommendation preferences.
- **`recommendation_service/`** — storage and retrieval of generated recommendations.
- **`scheduled_tasks/`** — background recommendation generation tasks.
- **`etl_service/`** — movie catalog data processing and indexing.
- **`nginx/`** — reverse proxy configuration.

## Getting Started

### Prerequisites

Make sure the following tools are installed:

- Docker
- Docker Compose

### Running Locally

To start the application in the local development environment, run:

```bash
chmod +x local_run.sh
./local_run.sh
```

The repository also includes `dev_run.sh` for the development workflow.

Docker Compose configuration is split into several files:

- `docker-compose.yml` — base service configuration.
- `docker-compose.override.prod.yml` — production-specific overrides.
- `docker-compose.override.test.yml` — test environment overrides.

For environment-specific configuration, refer to the example `.env` files provided in the corresponding service directories.

**Note:** The project requires appropriate environment variables and external API credentials to be configured before all services can operate correctly.

## Current Limitations and Future Improvements

The current architecture represents an MVP implementation. Several areas have been identified for further development.

### 1. Asynchronous Recommendation Processing

The current scheduler-based approach relies on a continuously running script that periodically processes recommendation requests.

This solution has several limitations:

- Recommendation generation is tightly coupled to a single background process.
- Processing large numbers of users may result in a significant number of sequential LLM requests.
- Long-running operations can delay the completion of the entire pipeline.
- There is no dedicated mechanism for distributing and managing background tasks.

**Planned improvement:**

Introduce a message broker (e.g. Apache Kafka) to decouple recommendation generation from the API layer and enable asynchronous, distributed processing.

An initial implementation was explored, but the current MVP uses the simpler scheduler-based approach.

### 2. User Context Propagation

Currently, some API endpoints use a hardcoded `user_id` when accessing user-specific data.

This is a temporary implementation that needs to be replaced with proper user context propagation.

**Planned improvement:**

Extract the authenticated user's ID from the authentication context and make it available to endpoint handlers.

One possible implementation is to use FastAPI's `Request` object and its `request.state` mechanism to pass user information from the authentication validation layer to the endpoint.

### 3. Recommendation Generation Efficiency

The current implementation processes user preferences and generates recommendations through multiple service-to-service requests.

Potential improvements include:

- Reducing unnecessary API calls.
- Processing users in batches.
- Introducing caching where appropriate.
- Tracking recommendation generation status.
- Implementing retry mechanisms for failed LLM requests.
- Generating recommendations only when user preferences have changed.

### 4. Scalability and Reliability

Further improvements may include:

- Distributed background task processing.
- Better service health monitoring.
- Centralized logging and observability.
- Improved error handling between microservices.
- Automated integration and end-to-end testing.

## Project Status

The project is an MVP demonstrating a microservice-based movie recommendation platform with LLM-powered personalization.

The current implementation prioritizes a working end-to-end system while leaving room for architectural improvements, particularly in asynchronous processing, scalability, and user context management.