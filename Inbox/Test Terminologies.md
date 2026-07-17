### Fuzzing
Negative/edge-case fuzzing just means: **instead of testing that the API works correctly, you test that it fails safely.**

Think of it like this — normal testing checks "if I order 1 ticket, do I get 1 ticket confirmed?" That's the happy path.

Fuzzing does the opposite. It deliberately sends the API bad, weird, or extreme input to see what happens:

- **Malformed** — send garbage data where a proper value is expected (e.g., sending `"banana"` where a date is expected, or a broken/incomplete JSON body)
- **Boundary** — push values to their limits (e.g., order **0** tickets, order **-1** tickets, order **999999999** tickets, send a name that's 10,000 characters long)
- **Wrong-type** — send the wrong kind of data entirely (e.g., sending text `"abc"` into a field that expects a number, or sending `true` where a price is expected)

**Why it matters:** a well-built API should reject this stuff gracefully — return a clean error message like "invalid input" — instead of crashing, freezing, or (worse) silently accepting the bad data and corrupting something in the database. If it _does_ crash or misbehave, that's a real bug you just found before a real user (or attacker) stumbled into it accidentally.

That's also why the spec calls this a "high-value" capability — real-world bugs cluster heavily around this kind of unexpected input, way more than around the happy path everyone already tests.
### BOLA/BFLA
This is testing two different things that people often mix up: **"are you logged in?"** vs. **"are you allowed to do this specific thing?"**

**1. The credential checks (missing/invalid/expired/wrong-scope)**

This is the basic "who are you" layer — authentication. You systematically hit every endpoint with:

- **Missing** credentials — no token at all
- **Invalid** credentials — a fake or malformed token
- **Expired** credentials — a token that used to work but shouldn't anymore
- **Wrong-scope** credentials — a token that's valid, but only for limited permissions (e.g., a "read-only" token trying to do a "delete" action)

The API should reject all of these consistently. Sounds obvious, but it's easy for one endpoint to get missed when a team is manually building these checks by hand.

**2. BOLA/BFLA — the more dangerous one**

This is authoriz_ation_, not authentic_ation_ — meaning: you _are_ logged in validly, but should you be allowed to see or touch _this particular thing_?

- **BOLA** (Broken Object Level Authorization) — can User A access User B's _data_? Example: you're logged into your own account, but you change `/orders/1234` to `/orders/1235` in the URL and suddenly you're looking at a stranger's order.
- **BFLA** (Broken Function Level Authorization) — can a regular user access an _admin-only action_? Example: a normal user calls the "delete user" endpoint that should only work for admins.

**"Multi-tenant object-level authorization replay"** just means the actual test technique: log in as Account B, try to request Account A's object, and confirm the API says **403 Forbidden** — not confirm it just returns the data.

**Why this is called out specially:** the spec notes this is the single most common and most severe real-world API bug class — roughly 40% of real API attacks exploit BOLA specifically — and it's the one category that automated scanners tend to miss, because it requires understanding _whose data is whose_ (business context), not just checking if a token is valid. That's exactly why it fits TaloTrace's "business-knowledge-grounded testing" pitch rather than being commodity work.