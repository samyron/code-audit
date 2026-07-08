# Review Checklist

The per-dimension checklist for Phase 3. These are prompts for attention, not a form to fill out — apply the ones that fit the change and skip what's irrelevant. Depth beats breadth: a few verified, high-impact findings are worth more than a long list of speculation.

## Severity levels

Tag every finding so the reader can triage:

- **Critical** — must fix before merge. Bugs that corrupt data or crash, security vulnerabilities, breaking changes to a public contract, data loss.
- **Major** — should fix before merge. Incorrect behavior in a real (if narrower) case, a design decision that will be expensive to reverse, a meaningful performance regression.
- **Minor** — worth fixing but not blocking. Missing edge-case handling for unlikely input, readability problems, a small inefficiency.
- **Nit** — optional/subjective polish. Naming, phrasing, a marginally cleaner shape. Keep these few.

---

## Correctness

Does the code actually do what it claims, on every path — not just the happy one?

- **Logic** — does it implement the stated intent? Off-by-one errors, inverted conditions, wrong operator (`&&` vs `||`, `<` vs `<=`), incorrect boolean short-circuits.
- **Edge cases** — empty input, single element, zero, negative numbers, very large values, `null`/`nil`/`None`/`undefined`, empty string, unicode, duplicate keys, boundary values at the limits of a range.
- **Error handling** — are errors caught, propagated, and surfaced correctly? Are exceptions swallowed silently? Are failures left in a partial/inconsistent state? Is cleanup (files, locks, connections) guaranteed on the error path (finally/defer/RAII)?
- **Return values and contracts** — does the function honor its documented contract? Are all return values and error codes actually checked by callers?
- **State and mutation** — unintended shared/aliased mutable state, mutating a collection while iterating, stale caches, state that can be observed mid-update.
- **Concurrency** — data races, missing locks or over-broad locks, deadlock ordering, non-atomic check-then-act, assumptions about execution order, thread-unsafe library calls.
- **Resource lifecycle** — leaks of memory, file handles, sockets, DB connections, goroutines/threads; use-after-free/close; unbounded queues.
- **Type and conversion** — integer overflow/underflow, truncation, float precision and equality comparisons, signed/unsigned confusion, silent coercions, timezone/locale/encoding assumptions.
- **Tests** — do new/changed tests actually exercise the new behavior and its edge cases, or just assert the happy path? Do they test behavior rather than implementation? Would they catch a regression? Is anything important untested?
- **Backward compatibility** — does this break existing callers, serialized data, API consumers, or migrations? Is the change safe to roll out and roll back?

## Architecture

Does the change fit the system, and will it age well?

- **Fit with existing design** — does it follow the patterns and layering the codebase already uses, or introduce a parallel way of doing the same thing? Consistency usually beats local cleverness.
- **Separation of concerns / boundaries** — is logic in the right layer (no business logic in controllers/views, no DB calls in the domain layer)? Are module boundaries respected, or does this reach across them?
- **Coupling and cohesion** — does it add tight coupling to unrelated modules? Does a change here now force changes elsewhere? Are related things kept together and unrelated things kept apart?
- **Abstraction level** — is the abstraction earning its keep, or is it premature/speculative generality (YAGNI)? Conversely, is copy-paste duplication accreting where a small abstraction belongs? Is a leaky abstraction exposing internals?
- **Interfaces and API design** — are public interfaces minimal, hard to misuse, and named well? Is the surface area larger than it needs to be? Are defaults safe?
- **Complexity** — is there a materially simpler design that meets the same requirements? Deep nesting, sprawling functions, and control flow that's hard to follow are architecture smells, not just style.
- **Dependencies** — is a new third-party dependency justified given its size, maintenance, license, and security posture? Could the standard library or existing code do it? Does it introduce a circular dependency?
- **Extensibility and change** — how does this behave when requirements shift in the obvious next direction? Are the likely-to-change parts isolated from the stable ones?
- **Configuration and feature flags** — are new behaviors gated appropriately? Are magic constants hoisted to config where they'll need tuning?

## Performance

Is it fast enough for its context, and will it stay that way as data grows? Weigh findings against the realistic scale — micro-optimizing a rarely-run path is noise.

- **Algorithmic complexity** — hidden O(n²) (nested loops over the same collection, `.includes()`/linear scan inside a loop), quadratic string building, repeated sorting. What happens at 10x or 100x the data?
- **Data structures** — is the right one used for the access pattern? List where a set/map is needed (membership tests), array where a queue is needed, repeated linear lookups that a hash/index would make O(1).
- **N+1 and data access** — a query inside a loop, lazy-loading in a loop, fetching per-item instead of batching. Are joins/`IN`/prefetch used instead? Are the right DB indexes present for the query's filters and sorts? Is `SELECT *` pulling more than needed?
- **Unnecessary work** — recomputing invariants inside loops, redundant network/DB round-trips, work that could be cached/memoized, eager computation of values that may not be used, re-parsing/re-serializing.
- **Memory** — loading an entire large dataset into memory instead of streaming/paginating, unbounded caches or buffers, retaining references that prevent GC, large allocations in hot paths, copies that could be moves/references.
- **I/O and network** — synchronous/blocking I/O on a hot or latency-sensitive path, missing timeouts, no batching, no connection pooling, chatty protocols, missing pagination on list endpoints.
- **Concurrency and contention** — lock contention on a hot path, false sharing, thread-pool starvation, serializing work that could be parallel (or vice versa — parallelizing trivially cheap work).
- **Hot path awareness** — is this code on a hot path (per-request, per-row, per-frame) or a cold one (startup, once a day)? Spend the scrutiny where it's actually executed a lot.

## Security

Assume input is hostile and the caller is untrusted.

- **Input validation** — is all external input (request params, headers, uploads, env, files, third-party API responses) validated, typed, and bounded before use? Allow-list over deny-list. Size/length limits to prevent abuse.
- **Injection** — SQL/NoSQL injection (parameterized queries / prepared statements, never string-concatenated queries), command injection (avoid shelling out with user data; no `shell=True`), path traversal (`../`), LDAP/XPath/template/log injection, deserialization of untrusted data.
- **Output encoding / XSS** — is untrusted data escaped for its output context (HTML, attribute, JS, URL)? Any `dangerouslySetInnerHTML`/`innerHTML`/`eval`/raw template with user data?
- **AuthN / AuthZ** — is every sensitive action authenticated? Is authorization checked on the server for *this* resource and *this* user (not just UI-hidden)? Watch for IDOR — object references a user can change to access others' data. Missing ownership checks. Privilege escalation paths.
- **CSRF** — do state-changing endpoints require anti-CSRF protection (token, SameSite cookies, or a non-cookie auth scheme)? Are unsafe methods reachable via GET?
- **CORS** — is the allowed-origin list actually restrictive, or reflecting `Origin` / using `*` together with credentials? Overly permissive CORS is a real leak.
- **Secrets and sensitive data** — hardcoded credentials, API keys, or tokens in code/config/logs? Secrets in error messages or committed files? Are they sourced from a vault/env instead? Is sensitive data (PII, tokens) logged?
- **Crypto** — home-rolled crypto, weak/broken algorithms (MD5/SHA1 for passwords, ECB, static IVs), missing TLS verification, predictable randomness for security purposes (use a CSPRNG), passwords not hashed with a slow salted KDF (bcrypt/scrypt/argon2).
- **Session and tokens** — secure/HttpOnly/SameSite cookie flags, token expiry and rotation, JWT signature actually verified (and `alg:none` rejected), session fixation.
- **Access control on data** — mass assignment / over-posting (binding request fields straight to a model), missing field-level restrictions, leaking internal fields in serializers/responses.
- **Denial of service** — unbounded loops/recursion on user input, regex catastrophic backtracking (ReDoS), zip/decompression bombs, missing rate limits or request-size limits on expensive operations.
- **Dependencies and supply chain** — newly added dependency with known CVEs, unpinned versions, or from an untrusted source.
- **Error handling as a leak** — stack traces, SQL, or internal paths returned to clients; verbose errors that aid an attacker.

---

## Cross-cutting reminders

- **Verify before reporting.** Trace the path. Distinguish "this is wrong" (Critical/Major) from "I'm not sure this is handled" (phrase as a question).
- **Prefer concrete fixes.** "This is O(n²)" is weaker than "this nested loop over `users` is O(n²); index `orders` by `user_id` into a map first."
- **Respect documented conventions.** If `CLAUDE.md`/`CONTRIBUTING.md` sanctions the pattern, it isn't a finding.
- **Don't do the linter's job.** Formatting and mechanical nits belong to tooling, not the review.
