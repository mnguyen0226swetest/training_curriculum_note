# Training Curriculum Note
- Assignments 1–3 in 2 weeks = feasible. Final assignment alone could take 3–4 weeks given the scope (full-stack portal + all integrations).
- KeyCloak 3–4 days: SSO concepts are non-trivial; Custom Provider adds complexity
- Jasper 2–3 day: sMore mechanical once Spring Boot setup works
- Camunda 3–4 days: BPMN modeling + API writing takes time
- Final — Should NOT be in 2 weeks

## Connection
Assignment 1 (KeyCloak) → Final: Auth layer
- Every screen in the Admin Portal sits behind KeyCloak. The login, logout, register, forgot password, and role-based access control you built in Assignment 1 becomes the security foundation the entire Final app runs on. Without it, anyone can hit your APIs.

Assignment 2 (Jasper) → Final: Import/Export feature
- The PDF/XLSX export pattern and the JDBC datasource connection you built in Assignment 2 gets dropped directly into the Final as the reporting module. The only new thing is the import direction (reading Excel via Apache POI) and making the template look professional.

Assignment 3 (Camunda) → Final: Approval workflow
- The maker/checker pattern in the Final is exactly the BPMN workflow you built in Assignment 3. Start process, complete task, and Java Delegate for email — same APIs, just now triggered by real business actions like order approval instead of a test flow.

```
Assignment 1 ──► Security layer (who can log in, what they can access)
Assignment 2 ──► Data layer    (get data in and out as files)
Assignment 3 ──► Process layer (how work moves between people)
                     │
                     ▼
              Final Assignment
         (one app that needs all three)
```

**So the 2-week training period isn't just learning — it's pre-building the three hardest integrations so the Final is mostly assembly, not research. If you skip or rush any of the three, you'll hit that gap again during the Final under time pressure**
