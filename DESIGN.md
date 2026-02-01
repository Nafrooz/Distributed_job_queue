# Design Document - Junior Developer Edition
## Distributed Task Scheduler & Job Queue
 
**Version:** 1.0  
**Last Updated:** February 1, 2026  
**Team:** 2 SDE-1 Developers (1 year experience)  
**Status:** Simplified for Learning

---

## 1. What Are We Building???

A **task scheduler** is a system that:
1. Accepts work to be done (tasks)
2. Stores them in a queue
3. Processes them in the background
4. Tracks their status

**Real-world example:** When you upload a video to YouTube, you don't wait for it to process. YouTube queues your video, processes it in the background, and notifies you when it's ready.

---

## 2. High-Level Architecture (Simple)

```
┌─────────────────┐
│  User/Client    │  ← Someone wants work done
└────────┬────────┘
         │ (1) Submit task via HTTP
         ▼
┌─────────────────────────────────────┐
│      Spring Boot API                │
│  - Receives task                    │
│  - Saves to PostgreSQL              │
│  - Adds to Redis queue              │
└────────┬────────────────────────────┘
         │ (2) Task in queue
         ▼
┌─────────────────────────────────────┐
│         Redis Queue                 │
│  [Task1] [Task2] [Task3] ...        │
└────────┬────────────────────────────┘
         │ (3) Worker polls queue
         ▼
┌─────────────────────────────────────┐
│         Worker Thread               │
│  - Gets task from queue             │
│  - Executes the work                │
│  - Updates status in PostgreSQL     │
└────────┬────────────────────────────┘
         │ (4) Task complete
         ▼
┌─────────────────────────────────────┐
│      PostgreSQL Database            │
│  - Stores all task information      │
│  - Task status: COMPLETED           │
└─────────────────────────────────────┘
```

---

## 3. Component Breakdown

### 3.1 API Service (Spring Boot)

**What it does:**
- Accepts HTTP requests to submit tasks
- Validates the request data
- Saves tasks to PostgreSQL database
- Adds task IDs to Redis queue
- Returns task ID to the user

**Files:**
- `TaskController.java` - Handles HTTP requests
- `TaskService.java` - Business logic
- `TaskRepository.java` - Database access

**Example Flow:**
```
1. User sends: POST /api/v1/tasks {"name":"send-email", "payload":"..."}
2. Controller receives request
3. Service validates and saves to database
4. Service adds task ID to Redis queue
5. Return task ID to user
```

---

### 3.2 PostgreSQL Database

**What it does:**
- Stores ALL task information permanently
- Acts as the "source of truth"

**Why PostgreSQL and not just Redis?**
- PostgreSQL persists data to disk (won't lose data on restart)
- Redis is fast but primarily in-memory
- We use both: PostgreSQL for permanence, Redis for speed

**Main Tables:**

**Tasks Table:**
```
┌─────────┬──────────────┬──────────┬────────────┬─────────────┐
│   ID    │     Name     │  Status  │  Payload   │ Created At  │
├─────────┼──────────────┼──────────┼────────────┼─────────────┤
│ uuid-1  │ send-email   │ QUEUED   │ {...}      │ 2026-02-01  │
│ uuid-2  │ generate-pdf │ COMPLETE │ {...}      │ 2026-02-01  │
│ uuid-3  │ send-sms     │ FAILED   │ {...}      │ 2026-02-01  │
└─────────┴──────────────┴──────────┴────────────┴─────────────┘
```

**Why we need this:**
- Know what tasks exist
- Track their current status
- See history of all tasks
- Handle failures and retries

---

### 3.3 Redis Queue

**What it does:**
- Acts as a fast, in-memory queue
- Workers poll this queue for work

**Why Redis?**
- **Fast**: In-memory = microsecond latency
- **Simple**: Built-in list operations
- **Blocking polls**: Workers can wait for new tasks

**How it works:**
```
Redis List: task_queue
┌────────┬────────┬────────┬────────┐
│ uuid-1 │ uuid-2 │ uuid-3 │ uuid-4 │  ← Left (new tasks added here)
└────────┴────────┴────────┴────────┘
                            Right (workers take from here) →

Operations:
LPUSH task_queue uuid-5    # Add new task to left
RPOP task_queue            # Worker takes from right (FIFO)
```

**Why not just database?**
- Database queries are slower (disk I/O)
- Redis is optimized for queuing
- Separate concerns: PostgreSQL for storage, Redis for coordination

---

### 3.4 Worker

**What it does:**
- Runs in a background thread
- Continuously polls Redis queue for tasks
- Executes tasks using appropriate executors
- Updates task status in database

**Pseudocode:**
```java
while (running) {
    1. Poll Redis queue (wait 5 seconds if empty)
    2. If task found:
       a. Load full task details from PostgreSQL
       b. Update status to PROCESSING
       c. Execute the task
       d. Update status to COMPLETED or FAILED
    3. If error:
       a. Check if should retry
       b. If yes, schedule retry
       c. If no, move to dead letter queue
}
```

**Why background thread?**
- Don't block HTTP requests
- Can run for minutes/hours
- Multiple workers can process tasks in parallel

---

## 4. Data Flow Diagrams

### 4.1 Submit Task Flow

```
User                 API               PostgreSQL        Redis
 │                    │                     │              │
 │ POST /tasks        │                     │              │
 ├───────────────────>│                     │              │
 │                    │                     │              │
 │                    │ INSERT task         │              │
 │                    ├────────────────────>│              │
 │                    │<────────────────────┤              │
 │                    │   (task_id)         │              │
 │                    │                     │              │
 │                    │ LPUSH task_id       │              │
 │                    ├─────────────────────────────────────>│
 │                    │                     │              │
 │ 201 Created        │                     │              │
 │ {id: uuid}         │                     │              │
 │<───────────────────┤                     │              │
```

**What happens:**
1. User submits task via HTTP
2. API saves task to PostgreSQL (gets ID back)
3. API adds task ID to Redis queue
4. API responds to user with task ID
5. User can check status later using this ID

---

### 4.2 Process Task Flow

```
Worker              Redis           PostgreSQL         Task Executor
 │                    │                  │                    │
 │ RPOP task_queue    │                  │                    │
 ├───────────────────>│                  │                    │
 │<───────────────────┤                  │                    │
 │   task_id          │                  │                    │
 │                    │                  │                    │
 │ SELECT * WHERE id=task_id            │                    │
 ├──────────────────────────────────────>│                    │
 │<──────────────────────────────────────┤                    │
 │   (full task data) │                  │                    │
 │                    │                  │                    │
 │ UPDATE status=PROCESSING              │                    │
 ├──────────────────────────────────────>│                    │
 │                    │                  │                    │
 │ execute(task)      │                  │                    │
 ├───────────────────────────────────────────────────────────>│
 │                    │                  │                    │
 │                    │                  │    (do the work)   │
 │                    │                  │                    │
 │<───────────────────────────────────────────────────────────┤
 │   result           │                  │                    │
 │                    │                  │                    │
 │ UPDATE status=COMPLETED              │                    │
 ├──────────────────────────────────────>│                    │
```

**What happens:**
1. Worker polls Redis queue (RPOP = right pop)
2. Gets task ID from queue
3. Loads full task from PostgreSQL
4. Updates status to PROCESSING
5. Executes the task using appropriate executor
6. Updates status to COMPLETED

---

### 4.3 Retry Flow (When Task Fails)

```
Worker                                PostgreSQL
 │                                        │
 │ Task execution failed!                │
 │                                        │
 │ Check: retryCount < maxRetries?       │
 ├───────────────────────────────────────>│
 │<───────────────────────────────────────┤
 │   retryCount=1, maxRetries=3 → YES    │
 │                                        │
 │ UPDATE retryCount=2                   │
 │        status=PENDING                 │
 │        nextRetryAt=now+4seconds       │
 ├───────────────────────────────────────>│
 │                                        │
 │ (wait 4 seconds - exponential backoff)│
 │                                        │
 │ LPUSH task_id to queue                │
 ├───────────────────────────────────────>│
```

**Why retry?**
- Network hiccups are common
- External services can be temporarily down
- Retrying often fixes transient failures

**Exponential backoff:**
- Retry 1: Wait 2^1 = 2 seconds
- Retry 2: Wait 2^2 = 4 seconds
- Retry 3: Wait 2^3 = 8 seconds
- This prevents overwhelming failing systems

---

## 5. Key Design Patterns (Simplified)

### 5.1 Producer-Consumer Pattern

```
Producer (API)          Queue (Redis)       Consumer (Worker)
     │                       │                      │
     │ Add task              │                      │
     ├──────────────────────>│                      │
     │                       │                      │
     │                       │   Poll for task      │
     │                       │<─────────────────────┤
     │                       │                      │
     │                       │   Return task        │
     │                       ├─────────────────────>│
```

**Why this pattern?**
- Decouples task submission from task execution
- API can accept tasks quickly without waiting
- Workers process at their own pace

---

### 5.2 Strategy Pattern (Task Executors)

```java
// Interface
interface TaskExecutor {
    void execute(Task task);
    String getTaskType();
}

// Different implementations
class EmailExecutor implements TaskExecutor {
    String getTaskType() { return "send-email"; }
    void execute(Task task) { /* send email */ }
}

class PDFExecutor implements TaskExecutor {
    String getTaskType() { return "generate-pdf"; }
    void execute(Task task) { /* generate PDF */ }
}

// Worker picks the right one
TaskExecutor executor = registry.get(task.getName());
executor.execute(task);
```

**Why this pattern?**
- Easy to add new task types
- Each executor focuses on one thing
- Clean separation of concerns

---

### 5.3 Repository Pattern (Database Access)

```java
// Repository interface
interface TaskRepository extends JpaRepository<Task, UUID> {
    List<Task> findByStatus(TaskStatus status);
}

// Service uses repository
class TaskService {
    @Autowired
    TaskRepository taskRepository;
    
    Task getTask(UUID id) {
        return taskRepository.findById(id)
            .orElseThrow(() -> new NotFoundException());
    }
}
```

**Why this pattern?**
- Hides database complexity
- Service layer doesn't know about SQL
- Easy to switch databases later

---

## 6. Important Concepts Explained

### 6.1 Idempotency (Advanced - Phase 2)

**Problem:** What if the same task is submitted twice by accident?

**Solution:** Idempotency key

```java
// User provides a unique key
TaskSubmitRequest {
    name: "send-email",
    payload: "...",
    idempotencyKey: "user-123-signup-email"  // Unique identifier
}

// On submission:
if (idempotencyStore.exists(idempotencyKey)) {
    return existingTask;  // Already processed!
}
```

**Why this matters:**
- Prevents duplicate emails/charges/actions
- Safe to retry API calls
- Common in payment systems

---

### 6.2 Task States (Status Lifecycle)

```
PENDING ────┐
            │
            ▼
         QUEUED ────┐
            │       │
            ▼       │
       PROCESSING   │ (on retry)
            │       │
     ┌──────┴───────┘
     │
     ├─────> COMPLETED (success!)
     │
     └─────> FAILED (gave up after retries)
```

**State transitions:**
1. **PENDING**: Created but not queued yet (for scheduled tasks)
2. **QUEUED**: In Redis queue, waiting for worker
3. **PROCESSING**: Worker is executing now
4. **COMPLETED**: Success!
5. **FAILED**: Failed after all retries

---

### 6.3 Dead Letter Queue (DLQ)

**What is it?**
A special queue for tasks that failed permanently

```
Normal Queue: [task1] [task2] [task3]
                │
                │ (fails 3 times)
                ▼
Dead Letter Queue: [task2:reason]
```

**Why we need it:**
- Don't lose failed tasks
- Can investigate what went wrong
- Option to manually retry later
- Prevents infinite retry loops

---

## 7. Database Schema Design Decisions

### Why UUID instead of Auto-increment ID?

**Auto-increment (1, 2, 3, ...):**
- Simple
- Sequential
- But predictable and can expose business data

**UUID (550e8400-e29b-41d4-a716-446655440000):**
- Globally unique
- No conflicts across multiple databases
- Can't guess other task IDs
- Industry standard for distributed systems

### Why JSONB for Payload?

```sql
-- Instead of creating columns for every possible field:
CREATE TABLE tasks (
    id UUID,
    email_to VARCHAR,
    email_subject VARCHAR,
    email_body TEXT,
    pdf_template VARCHAR,
    pdf_data TEXT,
    sms_number VARCHAR,
    sms_message TEXT,
    ...  -- Gets messy!
);

-- We use flexible JSONB:
CREATE TABLE tasks (
    id UUID,
    name VARCHAR,  -- Task type
    payload JSONB   -- Any JSON structure!
);
```

**Benefits:**
- Flexible: Different tasks need different data
- PostgreSQL can query JSON efficiently
- Easy to add new task types

---

## 8. Scaling Strategy (Simple)

### Current: Single Server

```
┌──────────────────────────────────────┐
│         One Server                   │
│                                      │
│  ┌────────────┐    ┌──────────────┐ │
│  │    API     │    │   Worker     │ │
│  └────────────┘    └──────────────┘ │
│                                      │
│  ┌────────────┐    ┌──────────────┐ │
│  │ PostgreSQL │    │    Redis     │ │
│  └────────────┘    └──────────────┘ │
└──────────────────────────────────────┘
```

**Good for:**
- Learning
- Development
- Small workloads (< 1000 tasks/minute)

---

### Future: Multiple Workers (Phase 2)

```
┌──────────┐
│   API    │
└────┬─────┘
     │
     ▼
┌────────────┐
│   Redis    │
└──────┬─────┘
       │
   ┌───┴───┬─────────┬─────────┐
   │       │         │         │
   ▼       ▼         ▼         ▼
┌──────┐┌──────┐┌──────┐┌──────┐
│Worker││Worker││Worker││Worker│
└──────┘└──────┘└──────┘└──────┘
```

**Benefits:**
- More workers = process more tasks in parallel
- If one worker crashes, others continue
- Easy to add more workers when needed

---

## 9. Error Handling Strategy

### 9.1 Retry Logic

```
Attempt 1: Execute ──> FAIL (network error)
           Wait 2 seconds
           
Attempt 2: Execute ──> FAIL (timeout)
           Wait 4 seconds
           
Attempt 3: Execute ──> FAIL (service down)
           Wait 8 seconds
           
Attempt 4: Give up ──> Move to Dead Letter Queue
```

**Why exponential backoff?**
- Give systems time to recover
- Don't hammer failing services
- Industry best practice

---

### 9.2 Exception Handling

```java
try {
    executor.execute(task);
    task.setStatus(COMPLETED);
} catch (NetworkException e) {
    // Retryable error
    scheduleRetry(task);
} catch (ValidationException e) {
    // Not retryable - bad data
    task.setStatus(FAILED);
    task.setErrorMessage(e.getMessage());
} catch (Exception e) {
    // Unknown error - log and decide
    log.error("Unexpected error", e);
    if (shouldRetry(task)) {
        scheduleRetry(task);
    } else {
        markAsFailed(task);
    }
}
```

**Key principles:**
- Catch specific exceptions
- Decide: retry or fail permanently
- Always log errors
- Update task status

---

## 10. Monitoring Strategy (Simple)

### 10.1 What to Monitor

```
Health Checks (Spring Actuator):
- Database connected? ✓
- Redis connected? ✓
- Disk space OK? ✓

Metrics:
- Tasks submitted today: 1,523
- Tasks completed today: 1,498
- Tasks failed: 5
- Current queue size: 20
- Average processing time: 2.3 seconds

Logs:
[INFO] Task submitted: id=uuid-123
[INFO] Task processing: id=uuid-123
[INFO] Task completed: id=uuid-123 duration=2.1s
[ERROR] Task failed: id=uuid-456 error="Connection timeout"
```

### 10.2 Endpoints to Check

```
GET /actuator/health
{
  "status": "UP",
  "components": {
    "db": {"status": "UP"},
    "redis": {"status": "UP"}
  }
}

GET /actuator/metrics
{
  "names": [
    "tasks.submitted",
    "tasks.completed",
    "tasks.failed"
  ]
}
```

---

## 11. Security Basics

### 11.1 API Key Authentication (Simple)

```
User Request:
POST /api/v1/tasks
Headers:
  X-API-Key: sk_test_abc123xyz789

Server:
1. Extract API key from header
2. Check if exists in database
3. Check if enabled
4. If valid, allow request
5. If invalid, return 401 Unauthorized
```

**Why API keys?**
- Simple to implement
- Easy for clients to use
- Can revoke if compromised
- Track usage per client

---

### 11.2 Rate Limiting (Simple)

```java
// In Redis:
ratelimit:api-key-xyz → counter

On each request:
1. Increment counter: INCR ratelimit:api-key-xyz
2. Set expiry (if first request): EXPIRE ratelimit:api-key-xyz 60
3. If counter > 100: Reject (429 Too Many Requests)
4. Else: Allow request
```

**Why rate limiting?**
- Prevent abuse
- Ensure fair usage
- Protect system resources

---

## 12. Testing Strategy

### 12.1 Unit Tests

Test individual methods:

```java
@Test
void shouldRetry_whenBelowMaxRetries() {
    Task task = new Task();
    task.setRetryCount(1);
    task.setMaxRetries(3);
    
    boolean result = retryService.shouldRetry(task);
    
    assertTrue(result);
}
```

---

### 12.2 Integration Tests

Test full flow with real database:

```java
@SpringBootTest
@Testcontainers
class TaskIntegrationTest {
    
    @Test
    void shouldProcessTaskEndToEnd() {
        // 1. Submit task
        TaskSubmitRequest request = new TaskSubmitRequest();
        request.setName("send-email");
        Task task = taskService.submitTask(request);
        
        // 2. Wait for worker to process
        Thread.sleep(3000);
        
        // 3. Check status
        Task updated = taskService.getTask(task.getId());
        assertEquals(TaskStatus.COMPLETED, updated.getStatus());
    }
}
```

---

## 13. Common Questions

### Q: Why not use just database for queuing?

**A:** Database queries are slower than Redis operations:
- Redis: Microseconds
- PostgreSQL: Milliseconds
- For high-throughput, Redis is better for queuing

### Q: What if Redis crashes?

**A:** Tasks are in PostgreSQL too!
1. Redis crashes
2. Worker can't poll queue
3. Restart Redis
4. Re-queue pending tasks from PostgreSQL
5. Resume processing

### Q: Can multiple workers process the same task?

**A:** No, because RPOP is atomic:
- Worker 1: RPOP → gets task-123
- Worker 2: RPOP → gets task-456
- Each task is removed when polled

### Q: What if worker crashes mid-execution?

**A:** Task status helps:
- Task was PROCESSING
- After timeout, mark as PENDING
- Re-queue for retry
- This is called "visibility timeout" (Phase 2 feature)

---

## 14. What You'll Learn

By building this project, you'll learn:

1. **Spring Boot fundamentals**
   - REST APIs
   - Dependency injection
   - Application structure

2. **Database skills**
   - JPA and Hibernate
   - Database design
   - Transactions

3. **Redis basics**
   - List operations
   - Key-value store
   - Caching concepts

4. **Concurrency**
   - Background threads
   - Thread safety
   - Async processing

5. **System design**
   - Queue-based architecture
   - Producer-consumer pattern
   - Error handling strategies

6. **Testing**
   - Unit tests
   - Integration tests
   - Test-driven development

7. **DevOps basics**
   - Docker
   - docker-compose
   - Deployment

---

## 15. Next Steps

After completing this project:

1. **Add features:**
   - Priority queues (critical, high, normal, low)
   - Scheduled tasks (cron)
   - Task cancellation
   - Webhooks for completion

2. **Improve observability:**
   - Prometheus metrics
   - Grafana dashboards
   - Distributed tracing

3. **Scale up:**
   - Multiple workers
   - Kubernetes deployment
   - Load balancing

4. **Enterprise features:**
   - Multi-tenancy
   - Advanced authentication
   - Audit logging

---

## 16. Architecture Decisions

### Why Spring Boot?
- Industry standard
- Great documentation
- Large ecosystem
- Easy to learn

### Why PostgreSQL?
- Free and open source
- Excellent JSON support
- Reliable and mature
- Great for learning SQL

### Why Redis?
- Fast in-memory operations
- Simple data structures
- Built for queuing
- Easy to set up

### Why Maven over Gradle?
- XML is more explicit
- Easier for beginners
- More examples online
- Still industry standard

---

## Summary

This design creates a **simple but complete** task scheduling system that:

✅ Accepts tasks via HTTP API  
✅ Queues them in Redis  
✅ Processes them in background  
✅ Tracks status in PostgreSQL  
✅ Handles failures with retries  
✅ Provides monitoring endpoints  
✅ Secured with API keys  
✅ Easy to test and deploy  

**Most importantly:** It teaches you fundamental concepts you'll use throughout your career!

Good luck building! 🚀
