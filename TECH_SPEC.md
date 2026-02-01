# Technical Specification Document - Junior Developer Edition
## Distributed Task Scheduler & Job Queue

**Version:** 1.0  
**Last Updated:** February 1, 2026  
**Team:** 2 SDE-1 Developers (1 year experience)  
**Status:** Simplified for Learning

---

## 1. Technology Stack (Simplified)  ,,,

### 1.1 Core Technologies

| Component | Technology | Version | Why This Choice |
|-----------|-----------|---------|-----------------|
| Programming Language | Java | 17 LTS | Stable, lots of learning resources, good job market |
| Framework | Spring Boot | 3.1.x | Industry standard, beginner-friendly, great documentation |
| Build Tool | Maven | 3.9.x | Easier than Gradle for beginners, XML-based |
| Database | PostgreSQL | 14 | Free, powerful, widely used in industry |
| Cache/Queue | Redis | 7 | Simple to learn, fast, versatile |
| Testing | JUnit 5 + Testcontainers | Latest | Standard testing tools |
| IDE | IntelliJ IDEA Community | Latest | Best Java IDE, free version is enough |

### 1.2 Key Libraries

**Spring Boot Starters:**
```xml
<dependencies>
    <!-- Web APIs -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Database -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Monitoring -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Lombok (reduce boilerplate) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Why these libraries?**
- All official Spring libraries = well documented
- Lombok = less code to write
- Testcontainers = easy integration testing

---

## 2. Project Structure (Simplified)

```
task-scheduler/
├── pom.xml                           # Maven configuration
├── docker-compose.yml                # Local PostgreSQL + Redis
├── README.md                         # How to run the project
└── src/
    ├── main/
    │   ├── java/com/taskscheduler/
    │   │   ├── TaskSchedulerApplication.java    # Main class
    │   │   │
    │   │   ├── controller/                      # REST endpoints
    │   │   │   ├── TaskController.java
    │   │   │   └── ScheduleController.java
    │   │   │
    │   │   ├── service/                         # Business logic
    │   │   │   ├── TaskService.java
    │   │   │   ├── QueueService.java
    │   │   │   ├── RetryService.java
    │   │   │   └── ApiKeyService.java
    │   │   │
    │   │   ├── repository/                      # Database access
    │   │   │   ├── TaskRepository.java
    │   │   │   ├── ScheduleRepository.java
    │   │   │   └── ApiKeyRepository.java
    │   │   │
    │   │   ├── entity/                          # Database tables
    │   │   │   ├── Task.java
    │   │   │   ├── Schedule.java
    │   │   │   └── ApiKey.java
    │   │   │
    │   │   ├── dto/                             # Request/Response objects
    │   │   │   ├── TaskSubmitRequest.java
    │   │   │   ├── TaskResponse.java
    │   │   │   └── ErrorResponse.java
    │   │   │
    │   │   ├── worker/                          # Background worker
    │   │   │   ├── Worker.java
    │   │   │   ├── WorkerStarter.java
    │   │   │   └── executor/
    │   │   │       ├── TaskExecutor.java (interface)
    │   │   │       ├── EmailTaskExecutor.java
    │   │   │       └── LogTaskExecutor.java
    │   │   │
    │   │   ├── config/                          # Configuration
    │   │   │   ├── RedisConfig.java
    │   │   │   └── WebConfig.java
    │   │   │
    │   │   └── exception/                       # Custom exceptions
    │   │       ├── TaskNotFoundException.java
    │   │       └── GlobalExceptionHandler.java
    │   │
    │   └── resources/
    │       ├── application.yml                   # Main config
    │       └── application-prod.yml              # Production config
    │
    └── test/
        └── java/com/taskscheduler/
            ├── TaskServiceTest.java              # Unit tests
            └── TaskIntegrationTest.java          # Integration tests
```

**Key Principles:**
- One class per file
- Clear package separation
- Simple, descriptive names

---

## 3. Database Schema (Simplified)

### 3.1 Tasks Table (Main Table)

```sql
CREATE TABLE tasks (
    -- Primary key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- What to do
    name VARCHAR(255) NOT NULL,           -- Task type (e.g., "send-email")
    payload TEXT NOT NULL,                 -- JSON string with task data
    
    -- Status tracking
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    
    -- Retry configuration
    max_retries INTEGER DEFAULT 3,
    retry_count INTEGER DEFAULT 0,
    next_retry_at TIMESTAMP,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    -- Error handling
    error_message TEXT
);

-- Index for finding pending tasks
CREATE INDEX idx_tasks_status ON tasks(status);

-- Index for retry scheduling
CREATE INDEX idx_tasks_retry ON tasks(next_retry_at) 
    WHERE status = 'PENDING' AND next_retry_at IS NOT NULL;
```

**Explanation:**
- `id`: Unique identifier for each task (UUID = universally unique)
- `name`: What type of task (links to TaskExecutor)
- `payload`: The actual data needed to execute (stored as JSON string)
- `status`: Current state (PENDING → QUEUED → PROCESSING → COMPLETED/FAILED)
- Retry fields: Track retries
- Timestamps: When created, when completed
- Indexes: Make queries fast

### 3.2 Schedules Table (Cron Jobs)

```sql
CREATE TABLE schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    cron_expression VARCHAR(255) NOT NULL,  -- e.g., "0 9 * * *" (daily at 9am)
    task_name VARCHAR(255) NOT NULL,        -- What task to create
    task_payload TEXT NOT NULL,             -- Payload for the task
    enabled BOOLEAN DEFAULT TRUE,
    last_run_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_schedules_enabled ON schedules(enabled);
```

**Explanation:**
- Creates tasks automatically based on cron schedule
- `cron_expression`: When to run (learn cron syntax)
- `enabled`: Can turn schedules on/off
- `last_run_at`: Track when it last ran

### 3.3 API Keys Table (Authentication)

```sql
CREATE TABLE api_keys (
    key_value VARCHAR(255) PRIMARY KEY,     -- The actual API key
    tenant_id VARCHAR(255),                 -- Who owns this key
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_api_keys_enabled ON api_keys(enabled);
```

**Explanation:**
- Simple API key authentication
- `key_value`: The secret key clients send
- `tenant_id`: Who this key belongs to (for future multi-tenancy)

---

## 4. Redis Data Structures (Simplified)

### 4.1 Task Queue (List)

```redis
# Queue name
task_queue

# Operations
LPUSH task_queue "task-uuid-123"      # Add task to queue
RPOP task_queue                        # Remove task from queue (worker polls)
LLEN task_queue                        # Get queue size
```

**Why a list?**
- Simple FIFO queue (First In, First Out)
- Fast push and pop operations
- Perfect for job queues

### 4.2 Dead Letter Queue (List)

```redis
# DLQ for failed tasks
dead_letter_queue

# Add failed task
LPUSH dead_letter_queue "task-uuid-456:reason"
```

### 4.3 Rate Limiting (String)

```redis
# Key per API key
ratelimit:api-key-xyz

# Increment counter
INCR ratelimit:api-key-xyz           # Returns count
EXPIRE ratelimit:api-key-xyz 60      # Expires in 60 seconds
```

**How it works:**
- Each API key gets a counter
- Increment on each request
- If counter > limit (100), reject request
- Counter resets after 60 seconds

---

## 5. Core Java Classes (Simplified)

### 5.1 Task Entity

```java
package com.taskscheduler.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "tasks")
@Data  // Lombok: generates getters, setters, toString, etc.
public class Task {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(columnDefinition = "TEXT", nullable = false)
    private String payload;  // JSON string
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TaskStatus status = TaskStatus.PENDING;
    
    private Integer maxRetries = 3;
    private Integer retryCount = 0;
    private LocalDateTime nextRetryAt;
    
    private LocalDateTime createdAt;
    private LocalDateTime completedAt;
    
    @Column(columnDefinition = "TEXT")
    private String errorMessage;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

**Key Annotations:**
- `@Entity`: This is a database table
- `@Table`: Table name
- `@Data`: Lombok generates boilerplate code
- `@Id`: Primary key
- `@GeneratedValue`: Auto-generate UUID
- `@Enumerated`: Store enum as string
- `@PrePersist`: Run before saving to database

### 5.2 TaskStatus Enum

```java
package com.taskscheduler.entity;

public enum TaskStatus {
    PENDING,      // Created but not queued yet
    QUEUED,       // In Redis queue
    PROCESSING,   // Worker is executing
    COMPLETED,    // Success!
    FAILED        // Failed after all retries
}
```

### 5.3 TaskRepository (Database Access)

```java
package com.taskscheduler.repository;

import com.taskscheduler.entity.Task;
import com.taskscheduler.entity.TaskStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Repository
public interface TaskRepository extends JpaRepository<Task, UUID> {
    
    // Spring Data JPA auto-implements these based on method names!
    List<Task> findByStatus(TaskStatus status);
    
    List<Task> findByStatusAndNextRetryAtBefore(
        TaskStatus status, 
        LocalDateTime time
    );
}
```

**Magic of Spring Data JPA:**
- You just write method signatures
- Spring generates the SQL queries automatically!
- `findByStatus` → `SELECT * FROM tasks WHERE status = ?`

### 5.4 TaskService (Business Logic)

```java
package com.taskscheduler.service;

import com.taskscheduler.dto.TaskSubmitRequest;
import com.taskscheduler.entity.Task;
import com.taskscheduler.entity.TaskStatus;
import com.taskscheduler.repository.TaskRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@Slf4j  // Logging
public class TaskService {
    
    @Autowired
    private TaskRepository taskRepository;
    
    @Autowired
    private QueueService queueService;
    
    @Transactional  // Database transaction
    public Task submitTask(TaskSubmitRequest request) {
        log.info("Submitting task: {}", request.getName());
        
        // Create task entity
        Task task = new Task();
        task.setName(request.getName());
        task.setPayload(request.getPayload());
        task.setStatus(TaskStatus.PENDING);
        
        // Save to database
        task = taskRepository.save(task);
        
        // Add to Redis queue
        queueService.addToQueue(task.getId());
        task.setStatus(TaskStatus.QUEUED);
        taskRepository.save(task);
        
        log.info("Task queued: {}", task.getId());
        return task;
    }
    
    public Task getTask(UUID id) {
        return taskRepository.findById(id)
            .orElseThrow(() -> new TaskNotFoundException(id));
    }
}
```

**Key Concepts:**
- `@Service`: Business logic layer
- `@Autowired`: Dependency injection (Spring gives us instances)
- `@Transactional`: If method fails, database changes are rolled back
- `log.info()`: Write to logs for debugging

### 5.5 QueueService (Redis Operations)

```java
package com.taskscheduler.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class QueueService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String QUEUE_KEY = "task_queue";
    private static final String DLQ_KEY = "dead_letter_queue";
    
    // Add task to queue
    public void addToQueue(UUID taskId) {
        redisTemplate.opsForList().leftPush(QUEUE_KEY, taskId.toString());
        log.info("Task added to queue: {}", taskId);
    }
    
    // Poll queue (blocking, waits 5 seconds)
    public String pollQueue() {
        String taskId = redisTemplate.opsForList()
            .rightPop(QUEUE_KEY, 5, TimeUnit.SECONDS);
        
        if (taskId != null) {
            log.info("Task polled from queue: {}", taskId);
        }
        
        return taskId;
    }
    
    // Get queue size
    public Long getQueueSize() {
        return redisTemplate.opsForList().size(QUEUE_KEY);
    }
    
    // Add to dead letter queue
    public void addToDeadLetter(UUID taskId, String reason) {
        String value = taskId + ":" + reason;
        redisTemplate.opsForList().leftPush(DLQ_KEY, value);
        log.warn("Task moved to DLQ: {} - {}", taskId, reason);
    }
}
```

**Redis Operations:**
- `leftPush`: Add to left side of list (queue)
- `rightPop`: Remove from right side (FIFO)
- Blocking pop: Waits up to 5 seconds if queue is empty

### 5.6 Worker (Background Processor)

```java
package com.taskscheduler.worker;

import com.taskscheduler.entity.Task;
import com.taskscheduler.entity.TaskStatus;
import com.taskscheduler.repository.TaskRepository;
import com.taskscheduler.service.QueueService;
import com.taskscheduler.service.RetryService;
import com.taskscheduler.worker.executor.TaskExecutor;
import com.taskscheduler.worker.executor.TaskExecutorRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.UUID;

@Component
@Slf4j
public class Worker {
    
    @Autowired
    private QueueService queueService;
    
    @Autowired
    private TaskRepository taskRepository;
    
    @Autowired
    private TaskExecutorRegistry executorRegistry;
    
    @Autowired
    private RetryService retryService;
    
    private volatile boolean running = false;
    
    public void start() {
        running = true;
        Thread workerThread = new Thread(this::run);
        workerThread.setName("TaskWorker");
        workerThread.start();
        log.info("Worker started");
    }
    
    private void run() {
        while (running) {
            try {
                // Poll queue (blocks for 5 seconds)
                String taskId = queueService.pollQueue();
                
                if (taskId != null) {
                    processTask(UUID.fromString(taskId));
                }
            } catch (Exception e) {
                log.error("Worker error", e);
            }
        }
        log.info("Worker stopped");
    }
    
    private void processTask(UUID taskId) {
        log.info("Processing task: {}", taskId);
        
        // Load task from database
        Task task = taskRepository.findById(taskId).orElse(null);
        if (task == null) {
            log.warn("Task not found: {}", taskId);
            return;
        }
        
        // Update status
        task.setStatus(TaskStatus.PROCESSING);
        taskRepository.save(task);
        
        try {
            // Find the right executor
            TaskExecutor executor = executorRegistry.getExecutor(task.getName());
            
            // Execute the task
            executor.execute(task);
            
            // Mark as completed
            task.setStatus(TaskStatus.COMPLETED);
            task.setCompletedAt(LocalDateTime.now());
            log.info("Task completed: {}", taskId);
            
        } catch (Exception e) {
            log.error("Task execution failed: {}", taskId, e);
            task.setErrorMessage(e.getMessage());
            
            // Retry or fail
            if (retryService.shouldRetry(task)) {
                retryService.scheduleRetry(task);
                log.info("Task scheduled for retry: {}", taskId);
            } else {
                task.setStatus(TaskStatus.FAILED);
                queueService.addToDeadLetter(task.getId(), task.getErrorMessage());
                log.error("Task failed permanently: {}", taskId);
            }
        }
        
        taskRepository.save(task);
    }
    
    public void stop() {
        running = false;
    }
}
```

**Worker Flow:**
1. Poll queue (wait 5 seconds if empty)
2. Get task from database
3. Update status to PROCESSING
4. Execute task using appropriate executor
5. If success → COMPLETED
6. If failure → Retry or move to DLQ

### 5.7 TaskExecutor Interface

```java
package com.taskscheduler.worker.executor;

import com.taskscheduler.entity.Task;

public interface TaskExecutor {
    
    /**
     * Execute the task
     * @param task The task to execute
     * @throws Exception if execution fails
     */
    void execute(Task task) throws Exception;
    
    /**
     * Get the task type this executor handles
     * @return Task type (e.g., "send-email")
     */
    String getTaskType();
}
```

### 5.8 Sample Executor

```java
package com.taskscheduler.worker.executor;

import com.taskscheduler.entity.Task;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class EmailTaskExecutor implements TaskExecutor {
    
    @Override
    public void execute(Task task) throws Exception {
        log.info("Executing email task: {}", task.getId());
        log.info("Payload: {}", task.getPayload());
        
        // Simulate sending email
        Thread.sleep(1000);
        
        log.info("Email sent successfully");
    }
    
    @Override
    public String getTaskType() {
        return "send-email";
    }
}
```

---

## 6. API Endpoints (Simplified)

### 6.1 Submit Task

```http
POST /api/v1/tasks
Content-Type: application/json
X-API-Key: your-api-key-here

{
  "name": "send-email",
  "payload": "{\"to\":\"user@example.com\",\"subject\":\"Hello\"}"
}

Response: 201 Created
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "send-email",
  "status": "QUEUED",
  "createdAt": "2026-02-01T10:00:00"
}
```

### 6.2 Get Task Status

```http
GET /api/v1/tasks/550e8400-e29b-41d4-a716-446655440000
X-API-Key: your-api-key-here

Response: 200 OK
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "send-email",
  "status": "COMPLETED",
  "createdAt": "2026-02-01T10:00:00",
  "completedAt": "2026-02-01T10:00:05"
}
```

### 6.3 Health Check

```http
GET /actuator/health

Response: 200 OK
{
  "status": "UP",
  "components": {
    "db": {"status": "UP"},
    "redis": {"status": "UP"},
    "ping": {"status": "UP"}
  }
}
```

---

## 7. Configuration Files

### 7.1 application.yml (Development)

```yaml
spring:
  application:
    name: task-scheduler
  
  # PostgreSQL
  datasource:
    url: jdbc:postgresql://localhost:5432/taskscheduler
    username: taskuser
    password: taskpass
    hikari:
      maximum-pool-size: 10
  
  # JPA/Hibernate
  jpa:
    hibernate:
      ddl-auto: update  # Auto-create tables
    show-sql: true      # Show SQL in console
    properties:
      hibernate:
        format_sql: true
  
  # Redis
  data:
    redis:
      host: localhost
      port: 6379

# Logging
logging:
  level:
    com.taskscheduler: DEBUG
    org.springframework: INFO

# Server
server:
  port: 8080
```

### 7.2 docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    container_name: taskscheduler-db
    environment:
      POSTGRES_USER: taskuser
      POSTGRES_PASSWORD: taskpass
      POSTGRES_DB: taskscheduler
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7
    container_name: taskscheduler-redis
    ports:
      - "6379:6379"

volumes:
  postgres-data:
```

---

## 8. Testing Examples

### 8.1 Unit Test

```java
package com.taskscheduler;

import com.taskscheduler.entity.Task;
import com.taskscheduler.entity.TaskStatus;
import com.taskscheduler.service.RetryService;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class RetryServiceTest {
    
    @Test
    void shouldRetry_whenRetryCountBelowMax() {
        // Arrange
        RetryService retryService = new RetryService();
        Task task = new Task();
        task.setMaxRetries(3);
        task.setRetryCount(1);
        
        // Act
        boolean result = retryService.shouldRetry(task);
        
        // Assert
        assertTrue(result);
    }
    
    @Test
    void shouldNotRetry_whenRetryCountEqualsMax() {
        // Arrange
        RetryService retryService = new RetryService();
        Task task = new Task();
        task.setMaxRetries(3);
        task.setRetryCount(3);
        
        // Act
        boolean result = retryService.shouldRetry(task);
        
        // Assert
        assertFalse(result);
    }
}
```

### 8.2 Integration Test with Testcontainers

```java
package com.taskscheduler;

import com.taskscheduler.dto.TaskSubmitRequest;
import com.taskscheduler.entity.Task;
import com.taskscheduler.entity.TaskStatus;
import com.taskscheduler.service.TaskService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Testcontainers
class TaskServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:14")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @Autowired
    private TaskService taskService;
    
    @Test
    void shouldCreateAndQueueTask() {
        // Arrange
        TaskSubmitRequest request = new TaskSubmitRequest();
        request.setName("test-task");
        request.setPayload("{\"test\":\"data\"}");
        
        // Act
        Task task = taskService.submitTask(request);
        
        // Assert
        assertNotNull(task.getId());
        assertEquals(TaskStatus.QUEUED, task.getStatus());
        assertEquals("test-task", task.getName());
    }
}
```

---

## 9. Common Issues & Solutions

### Issue 1: "Port 5432 already in use"
**Solution:** PostgreSQL is already running. Stop it:
```bash
# On Mac
brew services stop postgresql

# On Linux
sudo systemctl stop postgresql
```

### Issue 2: "Connection refused to Redis"
**Solution:** Start Redis with docker-compose:
```bash
docker-compose up -d redis
```

### Issue 3: "Lombok not working"
**Solution:** 
1. Install Lombok plugin in IntelliJ
2. Enable annotation processing:
   - File → Settings → Build, Execution, Deployment → Compiler → Annotation Processors
   - Check "Enable annotation processing"

### Issue 4: "Tests failing with database errors"
**Solution:** Testcontainers needs Docker running:
```bash
docker ps  # Check Docker is running
```

---

## 10. Deployment (Simple)

### 10.1 Create JAR file

```bash
# Build project
mvn clean package

# JAR created at:
# target/task-scheduler-0.0.1-SNAPSHOT.jar
```

### 10.2 Create Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk-alpine

WORKDIR /app

COPY target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 10.3 Run with Docker Compose

Update docker-compose.yml:

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/taskscheduler
      SPRING_DATA_REDIS_HOST: redis
    depends_on:
      - postgres
      - redis
  
  postgres:
    # ... existing config ...
  
  redis:
    # ... existing config ...
```

Start everything:
```bash
mvn clean package
docker-compose up --build
```

---

## 11. Learning Resources

### Spring Boot
- Official Guides: https://spring.io/guides
- Spring Boot Reference: https://docs.spring.io/spring-boot/docs/current/reference/html/

### Java
- Java 17 Tutorial: https://dev.java/learn/
- Effective Java (book)

### PostgreSQL
- PostgreSQL Tutorial: https://www.postgresqltutorial.com/
- SQL Practice: https://sqlbolt.com/

### Redis
- Redis University: https://university.redis.com/
- Try Redis: https://try.redis.io/

### Testing
- JUnit 5 Guide: https://junit.org/junit5/docs/current/user-guide/
- Testcontainers: https://www.testcontainers.org/

---

## 12. Glossary (Terms You'll Learn)

| Term | Simple Explanation |
|------|-------------------|
| **Entity** | A class that maps to a database table |
| **Repository** | Interface for database operations |
| **Service** | Business logic layer |
| **Controller** | Handles HTTP requests |
| **DTO** | Data Transfer Object - for sending data between layers |
| **Dependency Injection** | Spring gives you instances of classes you need |
| **Transaction** | Database operations that succeed or fail together |
| **Queue** | Ordered list of items waiting to be processed |
| **Redis** | In-memory data store (very fast) |
| **JPA** | Java Persistence API (database framework) |
| **Lombok** | Library that generates boilerplate code |
| **Testcontainers** | Run Docker containers for tests |
| **UUID** | Universally Unique Identifier |
| **CRUD** | Create, Read, Update, Delete |

---

## 13. Next Steps After This Project

Once you complete this project, you'll be ready to:
1. Add more complex features (workflow orchestration)
2. Learn Kubernetes for scaling
3. Add Prometheus + Grafana for monitoring
4. Implement distributed tracing
5. Build a React frontend
6. Apply for mid-level positions!

**Remember:** This project teaches you the fundamentals. Master these before moving to advanced topics.

---

Good luck with your learning journey! 🚀
