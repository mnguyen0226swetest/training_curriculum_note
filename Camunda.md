# Assignment 3: Camunda — Intern Learning Guide

> **Audience:** Java student with coding background
> **Goal:** Understand Camunda BPMN, integrate it with Spring Boot + Angular, build workflow APIs
> **Estimated time:** 3–4 days

---

## Overview: What is This Assignment Actually Asking?

You need to:
1. **Understand** what Camunda is and how it runs business workflows
2. **Draw** a diagram showing how Camunda connects with Angular + Spring Boot
3. **Design** a BPMN diagram with 5 specific element types in Camunda Modeler
4. **Build** 3 APIs: start a workflow, change task status, complete a task
5. **Write** sample code for the full integration

---

## Day-by-Day Learning Plan

---

### Day 1 — Understand Camunda Core Concepts

**What to learn:**

#### 1. What is Camunda?

- Camunda is an **open-source workflow and decision automation platform**
- It lets you model a business process visually as a diagram, then **execute** that diagram as real running code
- Think of it as: instead of hardcoding `if/else` business logic in Java, you draw the process as a flowchart — and Camunda runs it

**Real-world example:**
> A leave approval process: Employee submits → Manager reviews → HR approves → System sends email
> Instead of coding all this logic manually, you draw it in Camunda and it manages the flow automatically

#### 2. What is BPMN?

- **BPMN** = Business Process Model and Notation
- It is a **standardized visual language** for drawing workflows — like UML but for business processes
- A `.bpmn` file is XML under the hood, but you design it visually in Camunda Modeler

#### 3. Key Terms You Must Know

| Term | Simple Explanation |
|---|---|
| **Process Definition** | The `.bpmn` diagram you deploy — the "blueprint" of the workflow |
| **Process Instance** | One running execution of the blueprint (like one leave request) |
| **Task** | A step in the process that needs to be done |
| **User Task** | A task assigned to a human — pauses the process and waits for user action |
| **Service Task** | A task executed automatically by Java code |
| **Gateway** | A decision point — routes the flow based on conditions (like an if/else) |
| **Token** | An invisible marker that moves through the diagram, representing where the process currently is |
| **Candidate Group** | A group of users who can claim and complete a User Task |
| **Process Variables** | Key-value data attached to a running instance (e.g., `approved = true`) |
| **Deployment** | Uploading a `.bpmn` file to Camunda so it can be executed |

#### 4. The 5 BPMN Elements Required by the Assignment

| Element | Symbol | Purpose |
|---|---|---|
| **User Task** | Rectangle with person icon | Human does this step manually |
| **Service Task** | Rectangle with gear icon | Java code runs automatically |
| **Conditional Gateway** | Diamond with X | Routes flow based on a process variable |
| **Expression** | Used inside gateways/tasks | EL expression like `${approved == true}` |
| **Java Delegate** | Implements `JavaDelegate` | Java class that runs when a Service Task executes |

**Why learn these 5 specifically?** The assignment explicitly asks you to include all of them in your BPMN diagram.

#### 5. Camunda's Built-in REST APIs

Camunda exposes REST endpoints out of the box. Most important ones:

| Endpoint | Method | Purpose |
|---|---|---|
| `/process-definition/key/{key}/start` | POST | Start a new process instance |
| `/task?processInstanceId={id}` | GET | Get tasks for a running instance |
| `/task/{taskId}/claim` | POST | Assign a task to a user |
| `/task/{taskId}/complete` | POST | Complete a User Task + pass variables |
| `/process-instance/{id}/variables` | GET | Get all variables of an instance |

**Why learn these?** Your 3 required APIs are wrappers around these Camunda REST calls.

---

### Day 2 — Draw the Architecture + Setup Spring Boot

**What to learn:**

#### 1. How the Components Connect

```
Angular App
    |
    | HTTP calls
    v
Spring Boot API (your custom endpoints)
    |
    | Uses Camunda Java API or REST API
    v
Camunda Engine (embedded in Spring Boot OR standalone server)
    |
    | Reads/executes
    v
.bpmn Process Definition
    |
    | When Service Task fires
    v
Java Delegate (your Java class runs)
```

**Two deployment options — pick Embedded for internship:**

| Mode | How | Best for |
|---|---|---|
| **Embedded** | Camunda runs inside your Spring Boot app | Simple, no separate server needed |
| **Standalone** | Camunda runs as a separate server | Production, microservices |

#### 2. Spring Boot + Camunda Setup

Add to `pom.xml`:
```xml
<dependency>
    <groupId>org.camunda.bpm.springboot</groupId>
    <artifactId>camunda-bpm-spring-boot-starter-rest</artifactId>
    <version>7.20.0</version>
</dependency>

<dependency>
    <groupId>org.camunda.bpm.springboot</groupId>
    <artifactId>camunda-bpm-spring-boot-starter-webapp</artifactId>
    <version>7.20.0</version>
</dependency>

<!-- In-memory H2 database for development -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

Add to `application.yml`:
```yaml
camunda.bpm:
  admin-user:
    id: admin
    password: admin
    firstName: Admin
  filter:
    create: All Tasks

spring:
  datasource:
    url: jdbc:h2:mem:camunda;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver

server:
  port: 8080
```

Place your `.bpmn` file in:
```
src/main/resources/processes/leave-approval.bpmn
```

Camunda **auto-deploys** any `.bpmn` files it finds in `resources/processes/` on startup.

**Verify it works:** Open `http://localhost:8080/camunda/app/cockpit` → you should see your process deployed.

#### 3. Architecture Diagram to Draw (Draw.io)

```
Angular App
    |── POST /api/process/start ──────────────────────────────────►
    |── GET  /api/process/{id}/tasks ────────────────────────────►   Spring Boot
    |── POST /api/task/{taskId}/complete ────────────────────────►   Controller
                                                                          |
                                                                          v
                                                                    ProcessService
                                                                          |
                                                                          | Camunda Java API
                                                                          v
                                                                   Camunda Engine
                                                                    (Embedded)
                                                                          |
                                                              ┌───────────┴───────────┐
                                                              v                       v
                                                       User Task              Service Task
                                                    (waits for user)        (runs JavaDelegate)
                                                                                      |
                                                                                      v
                                                                              MyJavaDelegate.java
                                                                           (send email, update DB...)
```

---

### Day 3 — Design the BPMN + Build the 3 APIs

**What to learn:**

#### 1. Design Your BPMN in Camunda Modeler

Download Camunda Modeler: https://camunda.com/download/modeler/

**Example: Leave Approval Process**

```
[Start Event]
     |
     v
[User Task: Submit Request]         ← Employee fills form
     |
     v
[User Task: Manager Review]         ← Manager approves/rejects
     |
     v
[Conditional Gateway]               ← Check: approved == true?
     |                    |
     | Yes                | No
     v                    v
[Service Task:        [Service Task:
 Send Approval]        Send Rejection]    ← Java Delegate runs here
     |                    |
     └────────┬───────────┘
              v
         [End Event]
```

**How to build this in Camunda Modeler:**
1. Drag a Start Event onto the canvas
2. Add User Tasks by clicking the shape menu on the event
3. Add a Gateway (Exclusive/XOR) after the last User Task
4. Set conditions on the outgoing arrows: `${approved == true}` and `${approved == false}`
5. Add Service Tasks for each path
6. On each Service Task → set Implementation = `Java Class` → enter your delegate class name
7. Connect to End Events
8. Save as `.bpmn`

**Setting up a Java Delegate (for Service Task):**
```java
@Component("sendApprovalDelegate")  // name must match what you put in Modeler
public class SendApprovalDelegate implements JavaDelegate {

    @Override
    public void execute(DelegateExecution execution) throws Exception {
        // Read process variables
        String employeeName = (String) execution.getVariable("employeeName");
        Boolean approved = (Boolean) execution.getVariable("approved");

        // Do business logic here
        System.out.println("Sending approval email to: " + employeeName);

        // Set new variables if needed
        execution.setVariable("emailSent", true);
    }
}
```

In Camunda Modeler, set the Service Task's Java Class to:
`com.example.delegate.SendApprovalDelegate`

---

#### 2. API 1: Start a Business Process

```java
@RestController
@RequestMapping("/api/process")
public class ProcessController {

    @Autowired
    private RuntimeService runtimeService;

    @PostMapping("/start")
    public ResponseEntity<?> startProcess(@RequestBody Map<String, Object> variables) {
        // "leave-approval" must match the Process ID in your .bpmn file
        ProcessInstance instance = runtimeService.startProcessInstanceByKey(
            "leave-approval",
            variables  // e.g. { "employeeName": "John", "days": 3 }
        );

        return ResponseEntity.ok(Map.of(
            "processInstanceId", instance.getId(),
            "status", "STARTED"
        ));
    }
}
```

**Sample request body:**
```json
{
  "employeeName": "John Doe",
  "requestedDays": 3,
  "reason": "Family vacation"
}
```

---

#### 3. API 2: Change Task Status (Claim a Task)

"Change status" in Camunda means claiming or reassigning a User Task:

```java
@Autowired
private TaskService taskService;

// Get tasks for a process instance
@GetMapping("/{processInstanceId}/tasks")
public ResponseEntity<?> getTasks(@PathVariable String processInstanceId) {
    List<Task> tasks = taskService.createTaskQuery()
            .processInstanceId(processInstanceId)
            .list();

    List<Map<String, Object>> result = tasks.stream().map(task -> Map.of(
        "taskId", task.getId(),
        "taskName", task.getName(),
        "assignee", task.getAssignee() != null ? task.getAssignee() : "unassigned",
        "created", task.getCreateTime()
    )).collect(Collectors.toList());

    return ResponseEntity.ok(result);
}

// Claim (assign) a task to a user
@PostMapping("/task/{taskId}/claim")
public ResponseEntity<?> claimTask(
        @PathVariable String taskId,
        @RequestParam String userId) {

    taskService.claim(taskId, userId);
    return ResponseEntity.ok(Map.of(
        "taskId", taskId,
        "claimedBy", userId,
        "status", "CLAIMED"
    ));
}
```

---

#### 4. API 3: Complete a Task

```java
// Complete a task and move the process forward
@PostMapping("/task/{taskId}/complete")
public ResponseEntity<?> completeTask(
        @PathVariable String taskId,
        @RequestBody Map<String, Object> variables) {
    // variables can include decisions, e.g. { "approved": true }
    taskService.complete(taskId, variables);

    return ResponseEntity.ok(Map.of(
        "taskId", taskId,
        "status", "COMPLETED",
        "variablesPassed", variables
    ));
}
```

**Sample request body:**
```json
{
  "approved": true,
  "managerComment": "Approved. Enjoy your vacation!"
}
```

**What happens after this call:**
- Camunda receives `approved = true`
- The token moves to the Gateway
- Gateway evaluates `${approved == true}` → routes to Service Task
- Service Task fires `SendApprovalDelegate.execute()`
- Process continues automatically

---

### Day 4 — Wire Angular + Document + Polish

**What to learn:**

#### 1. Angular Integration (Simple HTTP calls)

Angular just calls your Spring Boot REST APIs — no special Camunda library needed:

```typescript
@Injectable({ providedIn: 'root' })
export class WorkflowService {

  constructor(private http: HttpClient) {}

  startProcess(variables: any): Observable<any> {
    return this.http.post('/api/process/start', variables);
  }

  getTasks(processInstanceId: string): Observable<any> {
    return this.http.get(`/api/process/${processInstanceId}/tasks`);
  }

  completeTask(taskId: string, variables: any): Observable<any> {
    return this.http.post(`/api/task/${taskId}/complete`, variables);
  }
}
```

Simple component usage:
```typescript
this.workflowService.startProcess({ employeeName: 'John', days: 3 })
  .subscribe(response => {
    this.processInstanceId = response.processInstanceId;
    console.log('Process started:', response);
  });
```

#### 2. Document Structure for Your Word/PDF Deliverable

```
1. Introduction — What is Camunda, BPMN, and workflow automation
2. Key Concepts — The 5 BPMN element types used
3. Architecture Diagram — Angular + Spring Boot + Camunda Engine
4. BPMN Diagram — Screenshot of your process in Camunda Modeler
5. Setup Guide — pom.xml, application.yml, auto-deploy
6. API 1: Start Process — Code + request/response examples
7. API 2: Change Task Status — Code + request/response examples
8. API 3: Complete Task — Code + request/response examples
9. Java Delegate — Code + explanation
10. Demo Screenshots or Video Link
11. References
```

---

## Summary Checklist

- [ ] Can explain what Camunda and BPMN are in simple terms
- [ ] Know the 10 key terms: Process Definition, Process Instance, User Task, Service Task, Gateway, Token, Java Delegate, Variables, Deployment, Candidate Group
- [ ] Camunda Modeler installed and BPMN drawn with all 5 required elements
- [ ] Spring Boot project set up with Camunda embedded dependency
- [ ] `.bpmn` file auto-deploys on Spring Boot startup
- [ ] `JavaDelegate` class implemented and linked to Service Task in Modeler
- [ ] API 1 working: Start a process instance with variables
- [ ] API 2 working: Get and claim tasks for a process instance
- [ ] API 3 working: Complete a task and pass decision variables
- [ ] Angular service calling all 3 APIs
- [ ] Architecture diagram drawn in Draw.io
- [ ] Document written (Word or PDF)
- [ ] Video demo recorded (optional but recommended)

---

## Quick Tips

- **Camunda Cockpit is your best friend** — open `localhost:8080/camunda/app/cockpit` to see running process instances, active tasks, and variables in real time. Use it to debug
- **The Process ID in `.bpmn` must match your Java code** — if your `startProcessInstanceByKey("leave-approval")` fails, check the ID attribute on the `<bpmn:process>` tag in your `.bpmn` XML
- **Use H2 in-memory DB for development** — no PostgreSQL setup needed. Switch to PostgreSQL only if required
- **Expressions use `${}` syntax** — `${approved == true}` on a gateway arrow, `${employeeName}` to reference a variable in a task name
- **`JavaDelegate` class name in Modeler must be the full package path** — e.g. `com.example.delegate.SendApprovalDelegate`, not just the class name
- **User Task vs Service Task** — if a human does it → User Task. If code does it automatically → Service Task. Get this right in your diagram
- **Camunda vs KeyCloak vs Jasper complexity** — Camunda has the steepest learning curve of the three because you're working with two tools (Modeler + Spring Boot) that must stay in sync
