# Assignment 1: KeyCloak — Intern Learning Guide

> **Audience:** Java student with coding background
> **Goal:** Understand, integrate, and deliver KeyCloak for SSO authentication
> **Estimated time:** 3–4 days

---

## Overview: What is This Assignment Actually Asking?

You need to:
1. **Understand** what KeyCloak is and how it handles login (SSO)
2. **Draw** a diagram showing how it connects with Angular + Spring Boot
3. **Build** two login APIs using Custom Providers
4. **Write** sample code tying it all together

---

## Day-by-Day Learning Plan

---

### Day 1 — Understand KeyCloak Core Concepts

**What to learn:**

#### 1. What is KeyCloak?
- KeyCloak is an **open-source Identity and Access Management (IAM)** tool
- It handles login, logout, registration, and token management **for you** — so your app doesn't need to build auth from scratch
- Think of it as a dedicated "security server" that sits between your user and your app

#### 2. What is SSO (Single Sign-On)?
- SSO = log in **once**, access **multiple apps**
- Example: Log into Google → you're automatically logged into Gmail, Drive, YouTube
- KeyCloak enables SSO across your Spring Boot and Angular apps

#### 3. Key Terms You Must Know

| Term | Simple Explanation |
|---|---|
| **Realm** | A "tenant" or isolated space in KeyCloak. Like a namespace for your project |
| **Client** | An app registered in KeyCloak (e.g., your Angular app, your Spring Boot app) |
| **User** | A person with credentials stored in KeyCloak |
| **Role** | Permissions assigned to users (e.g., ADMIN, USER) |
| **Token** | A JWT string KeyCloak issues after login. Your app uses this to verify identity |
| **Access Token** | Short-lived token used to call APIs |
| **Refresh Token** | Long-lived token used to get a new Access Token without re-login |
| **Authorization Code Flow** | The secure login flow used by web apps (Angular → KeyCloak → Spring Boot) |

#### 4. Built-in APIs to Know
KeyCloak exposes REST endpoints. The most important ones:

| Endpoint | Purpose |
|---|---|
| `POST /realms/{realm}/protocol/openid-connect/token` | Get access token (login) |
| `POST /realms/{realm}/protocol/openid-connect/logout` | Logout |
| `GET /realms/{realm}/protocol/openid-connect/userinfo` | Get user info from token |
| `GET /realms/{realm}/.well-known/openid-configuration` | Discover all endpoints |

**Why learn this?** You need to know what KeyCloak does out of the box before building on top of it.

**Resources:**
- KeyCloak official docs: https://www.keycloak.org/docs/latest/server_admin/
- Run KeyCloak locally via Docker: `docker run -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:latest start-dev`

---

### Day 2 — Draw the Architecture Diagram + Setup Spring Boot Integration

**What to learn:**

#### 1. How the 3 Components Connect

```
User (Browser)
    |
    | (1) Login Request
    v
Angular App  ──────────────────────────────────────────────────►  KeyCloak Server
    |                                                                    |
    | (2) Redirects to KeyCloak login page                              |
    |◄─────────────── (3) Returns JWT Access Token ─────────────────────|
    |
    | (4) Calls Spring Boot API with JWT in Header
    v
Spring Boot API
    |
    | (5) Validates JWT with KeyCloak
    v
Returns Data
```

**Draw this using Draw.io** — save as `.drawio` or export as PNG for your document.

#### 2. Spring Boot + KeyCloak Setup

Add this dependency to `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

Add to `application.yml`:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/your-realm-name
```

Protect your endpoints:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

**Why learn this?** This is the foundation. Spring Boot needs to know how to validate tokens from KeyCloak.

---

### Day 3 — Build the Two Custom Provider Login APIs

This is the core deliverable. You need two login flows:

---

#### API 1: Login using User Provider (Database)

**What it means:** KeyCloak authenticates users from **your own database** instead of its internal storage.

**How it works:**
1. You create a custom `UserStorageProvider` in Java
2. KeyCloak calls your provider to look up the user
3. If the user exists in your DB → KeyCloak issues a token

**Steps:**
1. Create a Maven project that implements `UserStorageProvider`
2. Package it as a JAR and deploy it into KeyCloak's `providers/` folder
3. Enable it in KeyCloak Admin Console under your Realm → User Federation

**Sample skeleton:**
```java
public class MyUserStorageProvider implements UserStorageProvider, UserLookupProvider, CredentialInputValidator {

    private final EntityManager em;

    @Override
    public UserModel getUserByUsername(RealmModel realm, String username) {
        // Query your DB here
        MyUser user = em.createQuery("SELECT u FROM MyUser u WHERE u.username = :username", MyUser.class)
                        .setParameter("username", username)
                        .getSingleResult();
        return new UserAdapter(session, realm, model, user);
    }

    @Override
    public boolean isValid(RealmModel realm, UserModel user, CredentialInput input) {
        // Validate password against your DB (BCrypt etc.)
        String inputPassword = input.getChallengeResponse();
        return BCrypt.checkpw(inputPassword, storedHash);
    }
}
```

---

#### API 2: Login using Remote User Federation

**What it means:** KeyCloak calls an **external REST API** (your service) to validate the user.

**How it works:**
1. You build a REST endpoint that accepts username/password
2. KeyCloak calls this endpoint during login
3. If your endpoint returns success → KeyCloak issues a token

**Your REST endpoint (Spring Boot):**
```java
@PostMapping("/auth/validate")
public ResponseEntity<?> validateUser(@RequestBody LoginRequest request) {
    boolean valid = userService.checkCredentials(request.getUsername(), request.getPassword());
    if (valid) {
        return ResponseEntity.ok(Map.of("status", "valid", "username", request.getUsername()));
    }
    return ResponseEntity.status(401).body(Map.of("status", "invalid"));
}
```

**Why learn both?** These are the two most common enterprise patterns for integrating existing user databases with KeyCloak. The assignment explicitly asks for both.

---

### Day 4 — Wire Up Angular + Write Sample Code + Document

**What to learn:**

#### 1. Angular + KeyCloak Setup

Install the KeyCloak Angular library:
```bash
npm install keycloak-angular keycloak-js
```

Initialize in `app.module.ts`:
```typescript
import { KeycloakAngularModule, KeycloakService } from 'keycloak-angular';

function initializeKeycloak(keycloak: KeycloakService) {
    return () => keycloak.init({
        config: {
            url: 'http://localhost:8080',
            realm: 'your-realm',
            clientId: 'angular-client'
        },
        initOptions: { onLoad: 'login-required' }
    });
}
```

Protect a route with `AuthGuard`:
```typescript
{ path: 'dashboard', component: DashboardComponent, canActivate: [AuthGuard] }
```

Call Spring Boot API with token automatically attached:
```typescript
// keycloak-angular auto-attaches the Bearer token to HTTP requests
this.http.get('http://localhost:8081/api/data').subscribe(data => console.log(data));
```

#### 2. Document Structure for Your Word/PDF Deliverable

```
1. Introduction — What is KeyCloak, SSO, and why it matters
2. Architecture Diagram — Angular + KeyCloak + Spring Boot
3. Setup Guide — Step-by-step KeyCloak server setup
4. API 1: User Provider Database — Code + explanation
5. API 2: Remote User Federation — Code + explanation
6. Angular Integration — Code + explanation
7. Demo Screenshots or Video Link
8. References
```

---

## Summary Checklist

- [ ] Can explain what KeyCloak and SSO are in simple terms
- [ ] Know the 5 key terms: Realm, Client, Token, Role, Flow
- [ ] Know which KeyCloak built-in REST endpoints do what
- [ ] Drew architecture diagram (Angular → KeyCloak → Spring Boot)
- [ ] Spring Boot configured as OAuth2 Resource Server
- [ ] API 1 working: Custom User Provider reading from your DB
- [ ] API 2 working: Remote User Federation calling your REST endpoint
- [ ] Angular app configured with keycloak-angular
- [ ] Document written (Word or PDF)
- [ ] Video demo recorded (optional but recommended)

---

## Quick Tips

- **Start KeyCloak locally with Docker** — fastest way to get a running server in 2 minutes
- **Use Postman** to test the token endpoint before writing any Java code
- **Realm = your project namespace** — create one Realm, register both Angular and Spring Boot as Clients inside it
- **The JWT token is just a Base64 string** — paste it into https://jwt.io to inspect it and understand the claims
- **Custom Providers are JARs** — you deploy them like plugins, not like normal Spring Boot apps
