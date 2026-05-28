# Assignment 4: Final Assignment — Intern Learning Guide

> **Audience:** Java student with coding background (Assignments 1–3 completed)
> **Goal:** Build a full-stack Admin Portal WebApp integrating everything learned
> **Estimated time:** 5–6 weeks
> **Interview at the end:** Demo + Q&A on Backend / Frontend / Database / Trend Techs

---

## Overview: What is This Assignment Actually Asking?

You need to build a **real full-stack application** that combines everything:

| Layer | What to Build |
|---|---|
| **Frontend** | Angular Admin Portal (Kiosk / Booking / Shopping / User Management) |
| **Backend** | Spring Boot REST APIs with CRUD, auth, import/export, workflow |
| **Database** | PostgreSQL with MyBatis or JPA/Hibernate ORM |
| **Auth** | KeyCloak for login/logout/register/forgot password + JWT |
| **Reports** | Jasper Report for import/export with good structure |
| **Workflow** | Camunda BPMN for maker/checker tasks + email service |
| **Source Control** | Git with `main` and `develop` branches |
| **Advanced (optional)** | Kafka / Message Queue / Cache + Design Patterns |

This is not a small project. Treat it like a **mini production system**.

---

## Recommended Week-by-Week Plan

| Week | Focus |
|---|---|
| **Week 1** | Project setup, DB schema, base CRUD backend |
| **Week 2** | KeyCloak auth integration (re-use Assignment 1) |
| **Week 3** | Angular frontend — layout, routing, CRUD screens |
| **Week 4** | Jasper Report import/export (re-use Assignment 2) |
| **Week 5** | Camunda BPMN workflow (re-use Assignment 3) |
| **Week 6** | Polish, Git cleanup, document, interview prep |

---

## Topic 1 — Project Architecture & Setup

**What to learn:**

### 1. Decide Your App Domain First
Pick ONE domain for your portal — this keeps scope manageable:

| Domain | Modules |
|---|---|
| **Kiosk** | Product catalog, order management, user management |
| **Booking** | Room/service booking, approval workflow, calendar |
| **Shopping** | Product, cart, order, category management |
| **Learning** | Course catalog, enrollment, progress tracking |

> Recommendation: **Shopping** or **Booking** — they naturally have CRUD, workflow (order approval), and reports (invoice export).

### 2. Recommended Project Structure

```
my-portal/
├── backend/                        ← Spring Boot
│   ├── src/main/java/com/example/
│   │   ├── config/                 ← Security, KeyCloak, Camunda config
│   │   ├── controller/             ← REST endpoints
│   │   ├── service/                ← Business logic
│   │   ├── repository/             ← MyBatis mappers or JPA repositories
│   │   ├── model/                  ← Entity classes
│   │   ├── dto/                    ← Request/Response objects
│   │   └── delegate/               ← Camunda JavaDelegates
│   ├── src/main/resources/
│   │   ├── processes/              ← .bpmn files
│   │   ├── reports/                ← .jrxml files
│   │   └── application.yml
│   └── pom.xml
│
└── frontend/                       ← Angular
    ├── src/app/
    │   ├── core/                   ← Auth guard, interceptors, services
    │   ├── shared/                 ← Reusable components, pipes
    │   ├── features/
    │   │   ├── auth/               ← Login, register pages
    │   │   ├── dashboard/
    │   │   ├── category/           ← CRUD screens
    │   │   ├── user-management/
    │   │   └── reports/
    │   └── app-routing.module.ts
    └── package.json
```

### 3. Key `pom.xml` Dependencies

```xml
<!-- Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Security + KeyCloak -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<!-- JPA (choose one ORM) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- MyBatis (alternative to JPA) -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>

<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Camunda -->
<dependency>
    <groupId>org.camunda.bpm.springboot</groupId>
    <artifactId>camunda-bpm-spring-boot-starter-rest</artifactId>
    <version>7.20.0</version>
</dependency>

<!-- JasperReports -->
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports</artifactId>
    <version>6.21.0</version>
</dependency>

<!-- XLSX export -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.2.3</version>
</dependency>
```

**Why set this up first?** A messy project structure becomes unmanageable fast. Set it up cleanly on Day 1 and you'll thank yourself in Week 5.

---

## Topic 2 — Database Design (PostgreSQL + ORM)

**What to learn:**

### 1. Design Your Schema First (Before Writing Any Code)

Example for a Shopping portal:

```sql
-- Users (managed by KeyCloak, mirrored here for app data)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    role VARCHAR(50) NOT NULL,  -- ADMIN, USER
    created_at TIMESTAMP DEFAULT NOW()
);

-- Categories
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT DEFAULT 0,
    category_id INT REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Orders (has workflow)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    total_amount DECIMAL(10,2),
    status VARCHAR(50) DEFAULT 'PENDING',  -- PENDING, APPROVED, REJECTED
    camunda_instance_id VARCHAR(255),       -- link to Camunda process
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 2. JPA vs MyBatis — Which to Use?

The assignment mentions both. Here's the difference:

| | JPA / Hibernate | MyBatis |
|---|---|---|
| **Style** | Object-first: you define Java classes, JPA generates SQL | SQL-first: you write SQL, MyBatis maps results to Java |
| **Best for** | Standard CRUD, simple queries | Complex queries, full SQL control |
| **Learning curve** | Lower for basic CRUD | Higher but more explicit |
| **Assignment says** | Use both if possible | Use both if possible |

**Recommendation:** Use **JPA for simple CRUD** (users, categories) and **MyBatis for complex reports/queries** (orders with joins). This satisfies the assignment requirement to use both.

**JPA Entity example:**
```java
@Entity
@Table(name = "categories")
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @CreationTimestamp
    private LocalDateTime createdAt;

    // getters/setters
}
```

**JPA Repository:**
```java
public interface CategoryRepository extends JpaRepository<Category, Long> {
    List<Category> findByNameContaining(String keyword);
}
```

**MyBatis Mapper Interface:**
```java
@Mapper
public interface OrderMapper {
    @Select("""
        SELECT o.*, u.username, u.email
        FROM orders o
        JOIN users u ON o.user_id = u.id
        WHERE o.status = #{status}
        ORDER BY o.created_at DESC
    """)
    List<OrderDto> findOrdersByStatus(@Param("status") String status);
}
```

**Why learn both?** The assignment explicitly requires MyBatis, JPA, and Hibernate. Understanding when to use each is a real interview question.

---

## Topic 3 — CSS Libraries (Frontend Styling)

**What to learn:**

The assignment lists: Bootstrap, TailwindCSS, AntD, Material Design, SCSS.

**You do NOT need to use all of them.** Pick one primary library:

| Library | Best for | Angular package |
|---|---|---|
| **Angular Material** | Clean admin UIs, Google-style components | `@angular/material` |
| **NG-ZORRO (AntD)** | Feature-rich admin panels, tables, forms | `ng-zorro-antd` |
| **Bootstrap** | Familiar, widely known, grid system | `ngx-bootstrap` |
| **TailwindCSS** | Utility-first, highly customizable | `tailwindcss` |

> **Recommendation: NG-ZORRO (AntD)** — it has the best pre-built admin components (tables with pagination, forms, modals, sidebars) that match exactly what an Admin Portal needs.

**Install NG-ZORRO:**
```bash
ng add ng-zorro-antd
```

**Use SCSS** for custom styles on top. Add to `angular.json`:
```json
"schematics": {
  "@schematics/angular:component": {
    "style": "scss"
  }
}
```

**Why learn this?** The frontend is what the interviewer sees first. A polished UI using a proper component library makes a strong impression.

---

## Topic 4 — CRUD + User Management

**What to learn:**

### 1. Standard CRUD Pattern (Spring Boot)

Every feature follows the same 4-layer pattern:

```
Controller (HTTP) → Service (business logic) → Repository (DB) → Entity (data)
```

**Example: Category CRUD**

```java
// Controller
@RestController
@RequestMapping("/api/categories")
@PreAuthorize("hasRole('ADMIN')")  // KeyCloak role check
public class CategoryController {

    @Autowired private CategoryService categoryService;

    @GetMapping
    public ResponseEntity<List<CategoryDto>> getAll() {
        return ResponseEntity.ok(categoryService.findAll());
    }

    @PostMapping
    public ResponseEntity<CategoryDto> create(@RequestBody @Valid CategoryDto dto) {
        return ResponseEntity.status(201).body(categoryService.create(dto));
    }

    @PutMapping("/{id}")
    public ResponseEntity<CategoryDto> update(@PathVariable Long id,
                                               @RequestBody @Valid CategoryDto dto) {
        return ResponseEntity.ok(categoryService.update(id, dto));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        categoryService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 2. Use DTOs, Not Entities Directly

Never expose your JPA Entity directly in the API response — use a DTO:

```java
// Entity stays internal
@Entity
public class Category { ... }

// DTO is what the API sends/receives
public class CategoryDto {
    private Long id;

    @NotBlank(message = "Name is required")
    private String name;

    private String description;
}
```

**Why?** Entities can have circular references (lazy loading), sensitive fields, or fields you don't want exposed. DTOs give you control.

### 3. Angular CRUD Table (NG-ZORRO example)

```typescript
// category.component.html
<nz-table [nzData]="categories" nzBordered>
  <thead>
    <tr>
      <th>ID</th>
      <th>Name</th>
      <th>Description</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    <tr *ngFor="let row of categories">
      <td>{{ row.id }}</td>
      <td>{{ row.name }}</td>
      <td>{{ row.description }}</td>
      <td>
        <button nz-button (click)="edit(row)">Edit</button>
        <button nz-button nzDanger (click)="delete(row.id)">Delete</button>
      </td>
    </tr>
  </tbody>
</nz-table>
```

---

## Topic 5 — Auth with KeyCloak + JWT

**What to learn (re-applying Assignment 1):**

This is Assignment 1 applied to your full app. Key additions for the Final:

### 1. Password Hashing (BCrypt)

When storing any password in your DB (not via KeyCloak):
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

// Usage
String hashed = passwordEncoder.encode(rawPassword);
boolean valid = passwordEncoder.matches(rawPassword, hashed);
```

### 2. Forgot Password Flow with KeyCloak

KeyCloak handles this out of the box:
1. In KeyCloak Admin → Realm Settings → Login tab → enable **Forgot Password**
2. KeyCloak sends a reset email automatically
3. Your Angular app just links to KeyCloak's reset URL:
   ```
   http://localhost:8080/realms/{realm}/login-actions/reset-credentials
   ```

### 3. JWT Token Interceptor in Angular

Automatically attach the Bearer token to every API call:
```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {

  constructor(private keycloak: KeycloakService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.keycloak.getKeycloakInstance().token;
    if (token) {
      const cloned = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
      return next.handle(cloned);
    }
    return next.handle(req);
  }
}
```

Register in `app.module.ts`:
```typescript
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
]
```

**Why learn this?** Every API call after login needs the token. Without the interceptor, you'd manually attach it everywhere.

---

## Topic 6 — Import/Export with Jasper Report

**What to learn (re-applying Assignment 2):**

For the Final, "Good Structure Template" means your Jasper report should look professional:

### 1. Good Report Template Structure

```
┌─────────────────────────────────────────┐
│  COMPANY LOGO    │    REPORT TITLE       │  ← Title Band
│                  │    Generated: {date}  │
├─────────────────────────────────────────┤
│  Filter Applied: Category = {param}     │  ← Page Header
├────┬──────────────┬────────┬────────────┤
│ ID │ Product Name │ Price  │ Stock      │  ← Column Header
├────┼──────────────┼────────┼────────────┤
│ 1  │ Widget A     │ $10.00 │ 50         │  ← Detail (repeats)
│ 2  │ Widget B     │ $20.00 │ 30         │
├────┴──────────────┴────────┴────────────┤
│                      Total: $30.00      │  ← Summary Band
└─────────────────────────────────────────┘
```

### 2. Import Feature (Excel/CSV → Database)

Import is the reverse of export — read uploaded file, save to DB:

```java
@PostMapping("/import")
public ResponseEntity<?> importProducts(@RequestParam("file") MultipartFile file) throws Exception {
    List<Product> products = new ArrayList<>();

    // Read Excel with Apache POI
    Workbook workbook = new XSSFWorkbook(file.getInputStream());
    Sheet sheet = workbook.getSheetAt(0);

    for (Row row : sheet) {
        if (row.getRowNum() == 0) continue; // skip header
        Product p = new Product();
        p.setName(row.getCell(0).getStringCellValue());
        p.setPrice(BigDecimal.valueOf(row.getCell(1).getNumericCellValue()));
        p.setStock((int) row.getCell(2).getNumericCellValue());
        products.add(p);
    }
    workbook.close();

    productRepository.saveAll(products);
    return ResponseEntity.ok(Map.of("imported", products.size()));
}
```

**Why learn import separately?** The assignment says "Import, Export using Jasper Report with Good Structure Template" — export uses Jasper, but import typically uses Apache POI (reading Excel) or OpenCSV (reading CSV).

---

## Topic 7 — Camunda BPMN Workflow (Maker/Checker)

**What to learn (re-applying Assignment 3):**

For the Final, the specific pattern is **Maker/Checker** — a common financial/enterprise approval pattern:

### 1. What is Maker/Checker?

```
Maker (creates/submits) → Checker (reviews/approves) → System Action (execute)
```

Example: Order workflow
```
[Employee creates order]  →  [Manager approves]  →  [System sends confirmation email]
     Maker                        Checker                    Service Task
```

### 2. BPMN Design for Maker/Checker

```
[Start]
   |
[User Task: Create Order]         ← Maker submits
   |
[User Task: Review Order]         ← Checker approves/rejects
   |
[Gateway: approved?]
   |              |
  Yes             No
   |              |
[Service Task:  [Service Task:
 Send confirm]   Send rejection + notify maker]
   |              |
  [End]          [End]
```

### 3. Email Service with JavaDelegate

```java
@Component("sendConfirmationDelegate")
public class SendConfirmationDelegate implements JavaDelegate {

    @Autowired
    private JavaMailSender mailSender;

    @Override
    public void execute(DelegateExecution execution) {
        String email = (String) execution.getVariable("userEmail");
        String orderId = (String) execution.getVariable("orderId");

        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(email);
        message.setSubject("Order Confirmed: " + orderId);
        message.setText("Your order has been approved!");
        mailSender.send(message);

        execution.setVariable("emailSent", true);
    }
}
```

**Add to `pom.xml`:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

**Add to `application.yml`:**
```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: your-email@gmail.com
    password: your-app-password
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

---

## Topic 8 — Git Workflow (Branch Strategy)

**What to learn:**

The assignment requires Git with `main` and `develop` branches. This is the standard team workflow:

### 1. Branch Strategy

```
main        ← Production-ready code only. Never commit directly here.
  |
develop     ← Integration branch. All features merge here first.
  |
feature/login-page      ← One branch per feature
feature/category-crud
feature/order-workflow
```

### 2. Daily Git Workflow

```bash
# Start a new feature
git checkout develop
git pull origin develop
git checkout -b feature/category-crud

# Work on your feature...
git add .
git commit -m "feat: add category CRUD endpoints"
git push origin feature/category-crud

# Merge feature back to develop (via Pull Request or directly)
git checkout develop
git merge feature/category-crud
git push origin develop

# When develop is stable, merge to main
git checkout main
git merge develop
git push origin main
```

### 3. Commit Message Convention

Use this format — it shows professionalism:

| Prefix | When to use |
|---|---|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `docs:` | Documentation |
| `refactor:` | Code restructure |
| `test:` | Adding tests |

Example: `feat: add Jasper PDF export for orders`

**Why learn this?** The interviewer will look at your Git history. A clean commit history with `main`/`develop` separation shows you understand real team workflows.

---

## Topic 9 — Advanced Topics (Kafka / Cache / Design Patterns)

**What to learn (stretch goal — do after core features work):**

### 1. Message Queue / Kafka (Optional)

Use case: When an order is approved, publish an event. Other services (email, inventory) consume it.

```java
// Producer — publish event after order approved
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;

public void publishOrderApproved(String orderId) {
    kafkaTemplate.send("order-approved", orderId);
}

// Consumer — listen and react
@KafkaListener(topics = "order-approved", groupId = "notification-group")
public void handleOrderApproved(String orderId) {
    emailService.sendConfirmation(orderId);
}
```

Add to `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 2. Cache (Redis)

Cache frequently read data to reduce DB calls:

```java
@Cacheable(value = "categories", key = "#id")
public CategoryDto findById(Long id) {
    return categoryRepository.findById(id)
        .map(this::toDto)
        .orElseThrow();
}

@CacheEvict(value = "categories", key = "#id")
public void delete(Long id) {
    categoryRepository.deleteById(id);
}
```

Add to `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 3. Design Patterns Worth Knowing for the Interview

| Pattern | Where used in this project |
|---|---|
| **Repository Pattern** | JPA/MyBatis data access layer |
| **Service Layer Pattern** | Business logic separated from controllers |
| **DTO Pattern** | Separating API contracts from DB entities |
| **Factory Pattern** | Creating different report exporters (PDF/XLSX) |
| **Observer/Event Pattern** | Kafka event publishing |
| **Singleton** | Spring beans are singletons by default |

**Why learn Kafka/Cache?** The assignment specifically mentions them. Even a basic implementation shows initiative. In the final interview, being able to explain *why* you'd use Kafka (async, decoupled) vs a direct call is worth points.

---

## Interview Preparation

The final interview covers **Backend / Frontend / Database / Trend Techs**.

### Backend Questions to Prepare

- What is the difference between JPA and MyBatis? When would you use each?
- How does Spring Security work with KeyCloak JWT tokens?
- What is a JavaDelegate in Camunda and when does it execute?
- What is the difference between `@Service`, `@Repository`, `@Controller` in Spring?
- What is a DTO and why not expose entities directly?

### Frontend Questions to Prepare

- What is Angular's `HttpInterceptor` and what did you use it for?
- What is an `AuthGuard` and how does it work?
- What is the difference between `Observable` and `Promise`?
- How do you handle errors in Angular HTTP calls?

### Database Questions to Prepare

- What is the difference between `@OneToMany` and `@ManyToOne` in JPA?
- What is an N+1 query problem and how do you fix it?
- What is an index and why does it improve query speed?
- What is a transaction and why is `@Transactional` important?

### Trend Tech Questions to Prepare

- What is Kafka and why use a message queue instead of a direct API call?
- What is Redis cache and what problem does it solve?
- What is a Design Pattern? Give 2 examples from your project.
- What is the difference between REST and GraphQL?
- What is Docker and why would you containerize this app?

---

## Document Structure for Your Word/PDF Deliverable

```
1.  Executive Summary — What the portal does
2.  System Architecture Diagram — Full stack overview
3.  Database Schema — ERD diagram
4.  Module Overview — List of all features built
5.  Authentication — KeyCloak setup, JWT flow
6.  CRUD Modules — Category, User, Product (with screenshots)
7.  Import/Export — Jasper Report templates + screenshots
8.  Workflow — Camunda BPMN diagram + API docs
9.  Email Service — Configuration + demo
10. Git Strategy — Branch diagram + commit history screenshot
11. Advanced Features — Kafka/Cache (if implemented)
12. Demo Video Link
13. References
```

---

## Final Summary Checklist

**Foundation**
- [ ] Project structure set up (backend + frontend folders)
- [ ] PostgreSQL running, schema designed and created
- [ ] Spring Boot runs with all dependencies loaded
- [ ] Angular project initialized with chosen CSS library

**Auth**
- [ ] KeyCloak integrated — login/logout/register working
- [ ] Forgot Password flow enabled in KeyCloak
- [ ] JWT interceptor in Angular attaching token to all calls
- [ ] Role-based access (`ADMIN`, `USER`) enforced on APIs

**CRUD**
- [ ] Category management — full CRUD (JPA)
- [ ] User management — full CRUD
- [ ] At least one more domain entity (Product, Order, etc.)
- [ ] MyBatis used for at least one complex query
- [ ] DTOs used for all API request/response

**Reports**
- [ ] Jasper template designed with professional structure
- [ ] PDF export working
- [ ] XLSX export working
- [ ] Excel/CSV import working

**Workflow**
- [ ] Camunda BPMN with Maker/Checker pattern deployed
- [ ] Start process API working
- [ ] Complete task API working
- [ ] Email notification sent via JavaDelegate

**Git**
- [ ] `main` and `develop` branches created
- [ ] Feature branches used during development
- [ ] Meaningful commit messages with `feat:/fix:` prefixes

**Polish**
- [ ] Frontend looks clean and professional
- [ ] Error handling on APIs (try/catch, proper HTTP status codes)
- [ ] README written explaining how to run the project
- [ ] Document written (Word or PDF)
- [ ] Video demo recorded

---

## Quick Tips

- **Build vertically, not horizontally** — finish one full feature end-to-end (DB → API → UI) before starting the next. Don't build all DB tables first then all APIs then all UI
- **Reuse Assignments 1–3 directly** — don't rewrite KeyCloak, Jasper, and Camunda from scratch. Copy your working code and adapt it
- **The interviewer cares about "why"** — not just that it works, but why you chose JPA over MyBatis for this case, why you used Kafka here, what design pattern you applied
- **Git history is part of the submission** — commit regularly with clean messages, not one giant commit at the end
- **Camunda Cockpit + Jasper preview + KeyCloak Admin** are your three debugging UIs — keep all three open during development
- **Start the email service early** — SMTP configuration issues often take longer than expected to debug
