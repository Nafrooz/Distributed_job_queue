# Distributed Task Scheduler & Job Queue

A distributed, horizontally-scalable task scheduling and job queue system built with Java 21, Spring Boot 3.2.x, PostgreSQL, and Redis.

## Project Structure

This is a Maven multi-module project organized into the following modules:

### API Module (`api/`)

REST API layer for task submission and management.

```
api/
├── src/
│   ├── main/
│   │   ├── java/com/taskscheduler/api/
│   │   │   ├── audit/              # Audit logging
│   │   │   ├── config/             # Configuration classes
│   │   │   ├── constants/          # API constants
│   │   │   ├── controller/         # REST controllers
│   │   │   ├── dto/                # Data Transfer Objects
│   │   │   ├── exception/          # Exception handlers
│   │   │   ├── mapper/             # MapStruct mappers
│   │   │   ├── security/           # Security components
│   │   │   ├── service/            # Business logic services
│   │   │   └── validation/         # Custom validators
│   │   └── resources/               # Configuration files
│   └── test/
│       └── java/                    # Test classes
```

### Common Module (`common/`)

Shared utilities and services used across modules.

```
common/
├── src/
│   ├── main/
│   │   └── java/com/taskscheduler/common/
│   │       ├── cache/              # Caching configurations
│   │       ├── config/              # Shared configurations
│   │       ├── constants/           # Common constants
│   │       └── service/            # Shared services
│   └── test/
│       └── java/                   # Test classes
```

### Domain Module (`domain/`)

Domain models, entities, and data access layer.

```
domain/
├── src/
│   ├── main/
│   │   ├── java/com/taskscheduler/domain/
│   │   │   ├── entity/             # JPA entities
│   │   │   ├── enums/               # Enumerations
│   │   │   ├── exception/           # Domain exceptions
│   │   │   ├── repository/          # Data repositories
│   │   │   └── valueobject/         # Value objects
│   │   └── resources/
│   │       └── db/
│   │           └── migration/       # Flyway migrations
│   └── test/
│       └── java/                    # Test classes
```

### Scheduler Module (`scheduler/`)

Cron-based task scheduling service.

```
scheduler/
├── src/
│   ├── main/
│   │   ├── java/com/taskscheduler/scheduler/  # Scheduler services
│   │   └── resources/               # Configuration files
│   └── test/
│       └── java/                    # Test classes
```

### Worker Module (`worker/`)

Background task execution service.

```
worker/
├── src/
│   ├── main/
│   │   ├── java/com/taskscheduler/worker/
│   │   │   ├── constants/           # Worker constants
│   │   │   ├── executor/            # Task executors
│   │   │   ├── health/              # Health checks
│   │   │   ├── pool/                # Worker pool management
│   │   │   └── service/             # Worker services
│   │   └── resources/                # Configuration files
│   └── test/
│       └── java/                    # Test classes
```

### Additional Folders

```
├── k8s/                             # Kubernetes deployment manifests
└── .github/
    └── workflows/                   # CI/CD pipeline configurations
```

## Features

- **Task Submission**: Submit tasks via REST API with priority levels
- **Distributed Processing**: Horizontal scaling with multiple workers
- **Fault Tolerance**: Automatic retry with exponential backoff, circuit breakers
- **Scheduling**: Cron-based recurring task scheduling
- **Security**: API key authentication, RBAC, rate limiting
- **Observability**: Prometheus metrics, structured logging, distributed tracing
- **Reliability**: Dead letter queues, visibility timeout, distributed locking

## Technology Stack

- **Java**: 21
- **Framework**: Spring Boot 3.2.x
- **Database**: PostgreSQL 15
- **Cache/Queue**: Redis 7
- **Build Tool**: Maven
- **Containerization**: Docker
- **Orchestration**: Kubernetes

## Quick Start

### Prerequisites

- Java 21
- Maven 3.9+
- Docker and Docker Compose

### Local Development

1. Start infrastructure:
```bash
docker-compose up -d
```

2. Build and run:
```bash
mvn clean install
cd api
mvn spring-boot:run
```

3. Submit a task:
```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "name": "send-email",
    "payload": "{\"to\":\"user@example.com\",\"subject\":\"Hello\"}"
  }'
```

## Documentation

- [Product Requirements Document](PRD.md)
- [Design Document](DESIGN.md)
- [Technical Specification](TECH_SPEC.md)
- [TODO List](TODO.md)

## License

MIT License
