```
Act as a Principal Software Architect. I want to brainstorm a technical design for a new project, but I want to do it in structured phases. Do not give me a final solution yet. 

For Phase 1, please review my project details below and ask me 3 to 5 highly targeted clarifying questions regarding scale, edge cases, data structures, or constraints that we must resolve before choosing an architecture.

Here is my project context:
- What I am building: [Insert a brief description of the feature or system]
- Our Tech Stack: [Insert languages, databases, or cloud providers you must use, e.g., Node.js, PostgreSQL, AWS]
- Key Constraints: [Insert constraints, e.g., 50ms max latency, low budget, tight deadline, or legacy system integration]
- Biggest Concern: [Insert what keeps you up at night about this project, e.g., data consistency, high traffic concurrency]

Once I answer your questions, we will move to Phase 2 to compare different architectural approaches. Please acknowledge you understand and reply with your clarifying questions.
```

```
each target should have app_id and project_id fields to comply with the existing project that serving multiple users and projects. For now, You don't need to care about this.
```


```
- > _"Compile our technical design discussion into a clean, comprehensive Markdown file. Include headings, bullet points, system architecture details, and fenced code blocks. Save it as `DESIGN.md` in the root directory."_
```

### Simplify
- make it simple, break it down to problem (a few bullet points, give example to clarify the problem)

### Implement Technical Design by Phases

```
Read PLAN.md and the design doc at docs/design.md. Implement only Phase 1: <name>. Do not touch code related to Phase 2+. Look at the existing codebase structure first (package.json / relevant modules) before writing anything, and match existing patterns.
```

```
❯ I imagine the registration flow would look like this:
  Base URL: https://api.staging.modun.vn/v1
  Environment: "Staging"
  Authentication: Selecting a stored "API Key" or "Token" from the existing credentials or create new.
```