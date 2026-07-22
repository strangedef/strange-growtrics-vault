**Task 1 — Scenario generation**

- What does the app-knowledge representation (`app_graph_reader.py`) actually capture?
- Can API test generation reuse it as-is, or does it need its own model?
- Does `knowledge_base/` already cover "product knowledge," or is that still unbuilt?

**Task 2 — Generating test cases**

- How would you generate test cases from Task 1's model + a client's API spec?
- Does today's GitHub integration expose route/handler code, or just PR metadata?
- How much better is generation with codebase access vs. without — worth pushing clients to connect GitHub?
