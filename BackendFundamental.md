*Q1. A Node API becomes slow under load, but CPU and DB look fine. How do you diagnose it?*

If CPU and DB are healthy but latency climbs under load, the time is going to waiting, not working. Something is queueing: the event loop, the libuv threadpool, a connection pool, an outbound socket limit, or GC. I diagnose by finding where requests wait, not by guessing at optimizations.

# Step 1: Characterize the problem

- Percentiles, not averages. Check p50/p95/p99 per endpoint. If only p99 is bad, suspect GC pauses or head-of-line blocking. If everything degrades, suspect saturation.

- Latency vs. concurrency. If throughput plateaus while latency keeps rising, requests are queueing somewhere. By Little’s Law, concurrency = throughput × latency, so a flat throughput ceiling means a fixed-size resource is exhausted.

- Scope. One endpoint or all? Since a deploy? Only on one instance?

# Step 2: Check the event loop first

Node runs your JS on one thread, so one blocked tick delays every in-flight request.

Per-core CPU hides this. A single Node process pinned at 100% shows as 12.5% on an 8-core machine, so “CPU looks fine” often isn’t true. Check per-process, per-core usage.

Measure event loop health directly:

const { monitorEventLoopDelay, performance } = require('perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

let last = performance.eventLoopUtilization();
setInterval(() => {
  const cur = performance.eventLoopUtilization();
  const elu = performance.eventLoopUtilization(cur, last).utilization;
  last = cur;
  console.log({
    p50_ms: h.percentile(50) / 1e6,
    p99_ms: h.percentile(99) / 1e6,
    elu, // >0.9 = saturated
  });
  h.reset();
}, 5000);

If loop delay p99 is in the tens or hundreds of ms, something is blocking it. Typical culprits are JSON.parse/stringify on large payloads, sync fs/crypto, catastrophic regex backtracking, big array sorts, or heavy template rendering. Confirm with a CPU profile (node --cpu-prof, or clinic flame / 0x for flame graphs).

# Step 3: Trace one slow request end to end

Add OpenTelemetry (or manual timing spans) and split request time into:

1. Queue time: time between arrival and handler start (event loop lag, backlog)
2. Handler time: your code
3. Wait on pool: time to acquire a DB or HTTP connection
4. Downstream time: DB query, external API, Redis

Whichever bucket grows with load is your bottleneck. This is the step that turns guessing into evidence.

# Step 4: Reproduce and verify

Use autocannon or k6 to ramp concurrency while watching event loop delay, pool queue depth, and p99 together. Change one variable at a time (pool size, threadpool size, keep-alive) and re-measure. A fix isn’t a fix until the load test shows it.

*Q2. Two concurrent requests try to deduct stock for the last item. How do you prevent overselling?*

I would prevent overselling using three things: atomic updates, proper concurrency control, and database transactions.

1. Atomic Updates: An atomic update means checking the stock quantity and deducting it in a single database operation. For example, I would use a conditional UPDATE query that deducts stock only when the available quantity is greater than zero. This prevents the application from checking and deducting stock in two separate steps.

2. Concurrency Control: When two requests arrive simultaneously, the database manages concurrent updates using mechanisms such as row-level locks and transaction isolation. If only one item is available, one request will successfully deduct it, while the other will find insufficient stock. This prevents both requests from purchasing the same item.

3. Database Transactions: I would wrap the stock deduction and order creation in a transaction. If both operations succeed, I commit the transaction. If order creation or another required operation fails, I roll back the transaction so the stock deduction is not left incomplete.

In summary, atomic updates prevent invalid stock deductions, concurrency control handles simultaneous requests, and transactions maintain consistency between inventory and orders. Together, these ensure that stock never becomes negative and overselling is prevented.

*Q3. HTTP and REST in depth. What do idempotency and safety mean for GET, PUT, PATCH, POST and DELETE? How do you make a POST payment endpoint safe to retry (idempotency keys)? Explain status code choices (400 vs 422, 401 vs 403, 409, 429), caching headers (ETag, Cache-Control) and API versioning strategies.*

In REST APIs, HTTP methods define how we interact with resources. Each method has specific behavior regarding **safety and idempotency**, which is important for designing reliable APIs.

### 1. Safety and Idempotency

**Safety** means that a request does not intentionally modify the server's application state. **Idempotency** means that sending the same request multiple times produces the same intended result as sending it once.

- **GET:** Safe and idempotent. It retrieves data without modifying application state. For example, fetching user details multiple times should not change the user.
- **PUT:** Not safe but idempotent. It replaces a resource or sets it to a specified state. Sending the same update repeatedly should produce the same final state.
- **PATCH:** Not safe and not necessarily idempotent. It partially updates a resource. For example, setting a user's name to "Dev" is idempotent, but incrementing a balance by 100 is not.
- **POST:** Not safe and not idempotent by default. Repeating a request to create an order may create multiple orders.
- **DELETE:** Not safe but idempotent in terms of its intended effect. Deleting the same resource repeatedly should leave it deleted, although the response codes may differ.

### 2. Making a Payment POST Endpoint Safe to Retry

Consider a payment API where a client sends a payment request, but the network times out before receiving the response. The payment might already have succeeded. If the client retries normally, the customer could be charged twice.

I would solve this using an **idempotency key**.

The client generates a unique key for each logical payment operation and sends it with the request, for example:

```http
POST /api/payments
Idempotency-Key: pay_abc123
Content-Type: application/json

{
  "orderId": 101,
  "amount": 1500,
  "currency": "INR"
}
```

On the backend, I would implement the following process:

1. Validate the request and authenticate the user.
2. Store the idempotency key in the database with a unique constraint, scoped to the appropriate user or merchant.
3. Associate the key with a hash of the request payload so that the same key cannot be reused for a different payment.
4. Process the payment and store its result using a reliable transaction and payment-provider integration.
5. If the same key is received again, return the previously stored result instead of creating another payment.
6. If the same key is used with a different payload, reject the request. If the original request is still processing, handle the duplicate according to the API's documented in-progress behavior.

For external payment gateways, I would also use their idempotency support where available and reconcile uncertain payment outcomes before attempting another charge.

**The key point is that idempotency must be enforced on the server, not just in the frontend.** A database uniqueness constraint and appropriate concurrency handling prevent two simultaneous requests from processing the same logical payment independently.

### 3. Choosing the Correct HTTP Status Code

I choose status codes according to what actually happened during request processing.

- **400 Bad Request:** The request is malformed or cannot be parsed. For example, invalid JSON or a missing required field, depending on the API's validation conventions.
- **422 Unprocessable Content:** The request is syntactically valid, but its contents fail validation or business rules. For example, an order quantity is negative.
- **401 Unauthorized:** Authentication is missing or invalid, such as an expired access token.
- **403 Forbidden:** The user is authenticated but does not have permission to access the resource.
- **409 Conflict:** The request conflicts with the current resource state. For example, trying to reserve a table that has already been booked, or updating a resource with a conflicting version.
- **429 Too Many Requests:** The client has exceeded a rate limit. I may include a `Retry-After` header to indicate when it can retry.

I would also use `201 Created` when a resource is successfully created, `200 OK` for a successful request returning a response, `204 No Content` when the operation succeeds without a response body, and `500 Internal Server Error` for unexpected server-side failures.

The distinction between `400` and `422` can vary by API convention, so consistency across the application is important.

### 4. Caching Headers: ETag and Cache-Control

Caching reduces unnecessary network requests and improves API performance.

**ETag** is a validator representing a particular version of a resource. The server returns it with a response:

```http
HTTP/1.1 200 OK
ETag: "user-v5"
```

When the client requests the resource again, it can send:

```http
If-None-Match: "user-v5"
```

If the resource has not changed, the server returns `304 Not Modified` without sending the response body again. If it has changed, the server returns the updated representation and a new ETag.

ETags can also support optimistic concurrency control through `If-Match`, helping prevent one client from overwriting another client's update.

**Cache-Control** defines how a response can be cached and for how long.

For example:

```http
Cache-Control: public, max-age=300
```

This allows a response to be cached for five minutes by shared or private caches, subject to the applicable caching rules.

For sensitive user-specific or payment responses, I would choose directives carefully. For example, `Cache-Control: no-store` prevents caches from storing the response. `private` restricts storage to private caches rather than shared caches, but does not itself prevent storage.

### 5. API Versioning Strategies

API versioning allows us to introduce changes without unexpectedly breaking existing clients.

The common approaches are:

- **URL versioning:** `/api/v1/users` and `/api/v2/users`. This is easy to understand, test and maintain.
- **Header versioning:** The client specifies the API version through a custom header or media type. This keeps URLs cleaner but makes debugging less obvious.
- **Query parameter versioning:** `/api/users?version=2`. It is simple to implement, although it is less commonly preferred for public APIs.

For most business applications, I prefer **URL versioning**, such as `/api/v1/orders`, because it is explicit and straightforward for frontend and mobile developers.

I would avoid creating a new version for every small change. Adding an optional response field is generally backward-compatible, while removing a field, changing its type, or changing existing behavior may be breaking changes.

When introducing a new major version, I would document the changes, maintain the previous version during a defined migration period, monitor its usage and communicate a deprecation timeline before removing it.

### Conclusion

In practice, I use HTTP methods according to their intended semantics, enforce idempotency for operations such as payments, return consistent status codes, use caching validators where appropriate, and version APIs when changes could break existing clients. These practices make REST APIs more reliable, secure and maintainable in production.

*Q4. Authentication and authorization. Compare session cookies, JWT access plus refresh tokens, and OAuth2/OIDC. How do you revoke a JWT, rotate refresh tokens and detect token reuse? Explain RBAC vs ABAC and where authorization checks belong (middleware, service layer, row level).*

Authentication verifies **who the user is**, while authorization determines **what the authenticated user is allowed to do**.

For example, in a restaurant management application, authentication verifies that an employee has logged in, while authorization determines whether that employee can view orders, update inventory, or manage users.

### 1. Session Cookies vs JWT vs OAuth2/OIDC

**Session-based authentication**

In session-based authentication, after a successful login, the server creates a session and stores its data in a database or a distributed cache such as Redis. The client receives a session ID through a cookie and sends it with subsequent requests.

The server validates the session ID and retrieves the associated user information.

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax
```

The main advantage is that sessions can be revoked immediately by deleting or invalidating the server-side session. However, the application needs shared session storage when running across multiple server instances.

I would use this approach for traditional web applications where server-side session management is practical.

**JWT access and refresh tokens**

JWT stands for JSON Web Token. It is a signed token containing claims such as the user ID, roles, issuer, audience, and expiration time.

After login, the server issues a short-lived access token and a longer-lived refresh token. The access token is sent with API requests:

```http
Authorization: Bearer <access_token>
```

The API validates the token's signature, issuer, audience, expiration, and other required claims without necessarily querying the database on every request.

When the access token expires, the client uses the refresh token to obtain a new access token.

The main advantage is efficient authentication across distributed APIs. The disadvantage is that a JWT access token generally remains valid until expiration unless the server implements an additional revocation mechanism.

**OAuth2 and OpenID Connect (OIDC)**

OAuth2 is an authorization framework that allows a client application to obtain limited access to a resource on behalf of a user or itself.

OpenID Connect builds on OAuth2 and adds an identity layer for authenticating users. It introduces the ID token, which communicates authentication information to the client.

For example, when a user signs in using Google:

- OAuth2 handles delegated authorization.
- OIDC provides identity information about the authenticated user.
- The application validates the identity information and establishes its own session or token-based authentication.

For user login, I would use OIDC rather than treating OAuth2 alone as an authentication protocol. For third-party application access, I would use OAuth2 with the appropriate flow, typically Authorization Code with PKCE for browser-based or mobile clients.

**Comparison:**

| Feature | Session cookies | JWT access + refresh | OAuth2/OIDC |
|---|---|---|---|
| State | Server-side session | Access token can be stateless | Depends on implementation |
| Revocation | Straightforward | Requires additional handling for immediate revocation | Depends on token type and provider |
| Scaling | Requires shared session storage | Convenient across distributed APIs | Suitable for delegated access and SSO |
| Main use case | First-party web applications | APIs and distributed services | SSO and third-party authorization |

These approaches are not mutually exclusive. An application can use OIDC for login and then establish a secure session cookie for its own web application.

### 2. How Do You Revoke a JWT?

A signed JWT is not normally revocable by itself because a server can validate it without checking a central session store.

I would consider three approaches:

1. **Short-lived access tokens:** Keep access tokens valid for a short period, such as 5–15 minutes, to limit the impact of a compromised token.
2. **Token denylist:** Store revoked token identifiers, such as the JWT `jti` claim, in Redis or a database until the tokens expire. The API checks the denylist during authentication.
3. **Token version or session version:** Maintain a token version or session status on the server. When the user logs out everywhere or an administrator disables the account, increment the version or invalidate the session. Requests must check the relevant server-side state.

Short-lived tokens reduce the exposure window, while a denylist or server-side state check enables more immediate revocation.

For a security-sensitive system, I would also revoke refresh tokens and terminate the associated token family when compromise is suspected.

### 3. Refresh Token Rotation and Reuse Detection

Refresh tokens usually have longer lifetimes than access tokens, so they need stronger protection.

I would implement **refresh token rotation**, where every successful refresh invalidates the old refresh token and issues a new one.

For example:

1. The client logs in and receives access token A and refresh token R1.
2. When access token A expires, the client sends R1 to the refresh endpoint.
3. The server validates R1, marks it as consumed, and issues access token B with refresh token R2.
4. The client replaces R1 with R2.
5. If R1 is presented again after it has already been consumed, the server detects possible token theft or replay.

To detect reuse, I would store refresh-token metadata server-side, including a hashed token value, token family identifier, expiration, and consumed or revoked status.

If an already-consumed refresh token is presented again, I would treat it as a potential compromise and revoke the active refresh-token family, requiring the user to authenticate again.

This requires careful concurrency handling. If two legitimate refresh requests arrive simultaneously, one may consume the token before the other. I would coordinate refresh operations atomically and handle duplicate requests according to a defined policy, rather than silently allowing an old token to create an unlimited number of new tokens.

Refresh tokens should be stored securely. For browser applications, an `HttpOnly`, `Secure`, appropriately configured `SameSite` cookie can reduce JavaScript access to the token. Cookie-based refresh endpoints also need appropriate CSRF protection.

### 4. RBAC vs ABAC

**RBAC — Role-Based Access Control**

In RBAC, permissions are assigned to roles, and users receive permissions through their assigned roles.

For example:

- Admin: manage users, roles, inventory, and orders.
- Manager: manage inventory and orders.
- Waiter: create orders and view assigned tables.

When a user requests an endpoint, the application checks whether their role grants the required permission.

RBAC is simple to understand and works well when permissions are organized around job responsibilities.

**ABAC — Attribute-Based Access Control**

In ABAC, access decisions depend on attributes of the user, resource, action, and environment.

For example, a restaurant employee may view an order only if the order belongs to their restaurant and their role permits viewing orders.

The authorization decision may depend on:

- User attributes: role, department, restaurant ID.
- Resource attributes: order ID, restaurant ID, ownership.
- Action: read, update, delete.
- Environment: time, device, or other contextual conditions.

RBAC is generally simpler for standard role permissions, while ABAC is more flexible for complex, context-dependent access rules. A system can combine both approaches.

### 5. Where Should Authorization Checks Belong?

I would implement authorization as a defense-in-depth process, with checks at multiple layers.

**1. Middleware or API policies**

Middleware validates authentication and establishes the current user's identity. Endpoint policies can then check broad permissions, such as whether the user has the `Orders.Read` permission.

For example, in ASP.NET Core:

```csharp
[Authorize(Policy = "Orders.Read")]
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    // Call the application service.
}
```

This prevents unauthenticated users or users without the required permission from reaching protected endpoints.

**2. Service layer**

The service layer enforces business authorization rules. For example, even if a manager has permission to update orders, the service must verify that the order belongs to the manager's restaurant.

I would not rely exclusively on the frontend or controller to enforce this rule, because the service may be called from other endpoints or background processes.

**3. Database or row-level security**

For stronger tenant isolation, I would ensure that database queries are scoped to the authorized tenant.

For example:

```sql
SELECT *
FROM Orders
WHERE Id = @OrderId
  AND RestaurantId = @RestaurantId;
```

The restaurant ID should come from a trusted, validated server-side identity or authorization context, not from an untrusted request parameter.

For systems requiring stronger isolation, database row-level security can provide an additional safeguard, ensuring queries cannot access rows outside the permitted tenant scope.

**My approach:** middleware handles authentication and broad permissions, the service layer enforces business rules and resource ownership, and database-level controls provide additional isolation where justified. This prevents users from accessing another restaurant's data simply by changing an order ID in the request.

### Conclusion

In production, I choose the authentication mechanism based on the application architecture. I use sessions when centralized server-side control is desirable, JWT access and refresh tokens when token-based API access is appropriate, and OAuth2/OIDC for delegated authorization and federated login.

For security, I implement short-lived access tokens, refresh-token rotation, reuse detection, and explicit revocation strategies. I combine RBAC for standard permissions with ABAC or resource-level checks when access depends on ownership or tenant boundaries. Most importantly, authorization must be enforced on the server at the appropriate layers, never trusted solely to the client.

*Q5. Database indexing and query performance. How does a B+ tree index work? Explain composite index column order, the leftmost-prefix rule, covering indexes and why a function on an indexed column prevents index use. How do you read an EXPLAIN plan and fix a slow query, including the N+1 problem?*

In a database, indexing improves query performance by allowing the database engine to locate rows without scanning the entire table. However, indexes also consume storage and add overhead to `INSERT`, `UPDATE`, and `DELETE` operations. So, I create indexes based on actual query patterns rather than indexing every column.

### 1. How Does a B+ Tree Index Work?

A B+ tree is a balanced tree data structure commonly used for database indexes in systems such as MySQL's InnoDB.

It contains three main components:

- **Root node:** The starting point for searching the tree.
- **Internal nodes:** Store separator keys and pointers to child nodes, helping the database navigate toward the required key range.
- **Leaf nodes:** Store indexed keys and references to the corresponding rows. In InnoDB, a clustered primary-key index stores the actual row data in its leaf pages, while secondary indexes store the indexed columns along with the primary-key value.

For example, suppose we have one million orders and an index on `order_id`. Without an index, the database may perform a full table scan and examine many rows. With a B+ tree index, it navigates through the tree to find the required key, typically requiring approximately logarithmic search work.

B+ trees are particularly useful for range queries because their leaf pages are ordered and linked, allowing efficient traversal through a range of values.

For example:

```sql
SELECT *
FROM Orders
WHERE order_id BETWEEN 1000 AND 1100;
```

The database can seek to the beginning of the range and scan the relevant leaf entries instead of scanning the entire table.

**The main advantage of a B+ tree is that it supports both efficient equality lookups and range queries while keeping the tree balanced.**

### 2. Composite Index and Column Order

A composite index contains multiple columns in a defined order. The order matters because the database organizes the index using the first column, then the second column within the first column's values, and so on.

Consider this index:

```sql
CREATE INDEX idx_orders_restaurant_status_date
ON Orders (RestaurantId, Status, CreatedAt);
```

This index is useful for queries such as:

```sql
SELECT *
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Pending'
  AND CreatedAt >= '2026-10-01';
```

The database can use the leading `RestaurantId` and `Status` columns to narrow down the search, followed by a range scan on `CreatedAt`.

However, if I frequently query orders using only `Status`, this index may not be an efficient access path because `Status` is not the leading column.

When deciding column order, I consider:

1. The actual `WHERE` conditions used by important queries.
2. Equality predicates before range predicates in many common index designs.
3. Columns used in joins, sorting, and grouping.
4. Column selectivity, workload frequency, and whether the index can cover the query.

I would not blindly put the most selective column first. The correct order depends on the workload and which query patterns the index must support.

### 3. What Is the Leftmost-Prefix Rule?

The leftmost-prefix rule means that a composite B-tree index can generally support searches beginning with its leading columns.

For the index:

```sql
CREATE INDEX idx_employee
ON Employees (DepartmentId, Role, Salary);
```

The index can generally support:

```sql
WHERE DepartmentId = 10

WHERE DepartmentId = 10
  AND Role = 'Developer'

WHERE DepartmentId = 10
  AND Role = 'Developer'
  AND Salary > 50000
```

But this query cannot generally use the index as an efficient direct lookup by `Role` alone:

```sql
WHERE Role = 'Developer'
```

Similarly, a condition on `Salary` alone does not normally benefit from this index as a direct search path.

There is an important detail: after the database encounters a range condition on an index column, later columns generally cannot further narrow the contiguous index range in the same way as leading equality columns. They may still help with filtering or covering the query, depending on the database and execution plan.

Therefore, I design composite indexes around the actual query patterns rather than simply combining every frequently queried column into one index.

### 4. What Is a Covering Index?

A covering index contains all the columns required by a query, including the columns needed for filtering and returning results.

For example:

```sql
CREATE INDEX idx_orders_lookup
ON Orders (RestaurantId, Status, CreatedAt);
```

Consider this query:

```sql
SELECT RestaurantId, Status, CreatedAt
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Pending';
```

The index contains the columns required by the query, so the database may retrieve the result directly from the index without accessing the underlying table rows. This is called a covering-index access path.

In MySQL InnoDB, secondary indexes also contain the primary-key value. This can make some queries covering even when the primary key is not explicitly included in the secondary-index definition.

Covering indexes reduce additional table lookups and can significantly improve read performance. However, adding too many columns increases index size, memory usage, and write overhead, so I create them selectively.

### 5. Why Can a Function on an Indexed Column Prevent Index Usage?

Consider an index on `CreatedAt`:

```sql
CREATE INDEX idx_orders_created_at
ON Orders (CreatedAt);
```

This query may prevent a normal index range scan:

```sql
SELECT *
FROM Orders
WHERE YEAR(CreatedAt) = 2026;
```

The database must evaluate `YEAR()` against candidate values, and the condition does not directly describe a contiguous range of the original indexed values. Depending on the optimizer and database, the index may not be used efficiently.

Instead, I rewrite the condition as a range:

```sql
SELECT *
FROM Orders
WHERE CreatedAt >= '2026-01-01'
  AND CreatedAt < '2027-01-01';
```

Now the database can perform an index range scan on `CreatedAt`.

Other examples include applying `LOWER()` to a text column or performing arithmetic on an indexed column. The exact behavior depends on the database optimizer. Some databases support expression indexes or functional indexes, which can make specific transformed expressions indexable.

**My general rule is to keep indexed columns directly searchable in predicates whenever possible.**

### 6. How Do You Read an EXPLAIN Plan?

When a query is slow, I first inspect its execution plan instead of immediately adding indexes or rewriting SQL.

In MySQL, I use:

```sql
EXPLAIN
SELECT *
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Pending';
```

For more detailed execution information, I can use:

```sql
EXPLAIN ANALYZE
SELECT *
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Pending';
```

`EXPLAIN ANALYZE` executes the query and reports actual execution statistics alongside the plan, so I use it carefully on production systems.

The important things I check are:

- **Access type (`type`):** `const`, `eq_ref`, `ref`, and `range` can indicate efficient access patterns. `ALL` indicates a full table scan, which may be expensive on a large table but can be reasonable for a small table or a query returning most rows.
- **Possible keys (`possible_keys`):** Indexes the optimizer considers candidates for the query.
- **Selected key (`key`):** The index actually selected by the optimizer. A `NULL` value means no index was selected for that table access.
- **Estimated rows (`rows`):** The estimated number of rows examined at that plan step. A large estimate can indicate an expensive operation.
- **Filtering (`filtered`):** The estimated percentage of rows that remain after filtering.
- **Extra:** Details such as `Using filesort`, `Using temporary`, or `Using index`. These provide clues about sorting, temporary processing, and covering-index access, but none should be treated as automatically bad without understanding the query.

With `EXPLAIN ANALYZE`, I compare estimated and actual row counts, loops, and execution times to find where the plan is spending its time.

### 7. How I Fix a Slow Query in Practice

Suppose an API retrieves pending orders for a restaurant, sorted by creation time:

```sql
SELECT Id, Status, CreatedAt
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Pending'
ORDER BY CreatedAt DESC
LIMIT 20;
```

I would follow these steps:

**Step 1: Measure the problem.**

I identify the slow endpoint, capture the actual SQL, check query execution time and frequency, and determine whether the delay comes from database execution, network transfer, or application-side processing.

**Step 2: Inspect the execution plan.**

I use `EXPLAIN` to check whether the database scans too many rows, chooses an unsuitable index, or performs unnecessary sorting.

**Step 3: Add or adjust the index based on the query pattern.**

For this example, I would evaluate:

```sql
CREATE INDEX idx_orders_restaurant_status_created
ON Orders (RestaurantId, Status, CreatedAt);
```

The equality filters come first, followed by the creation timestamp used for ordering. MySQL can potentially scan the matching index range in reverse order to satisfy `ORDER BY CreatedAt DESC`.

**Step 4: Retrieve only the necessary data.**

Instead of using `SELECT *`, I select the required columns. This reduces data transfer and may allow the query to become covering, depending on the index and the database engine.

**Step 5: Check pagination and sorting.**

For large datasets, I prefer keyset pagination over large `OFFSET` values. For example, I can request the next page using the last observed `CreatedAt` and a unique tie-breaker such as `Id`.

**Step 6: Validate the improvement.**

I compare execution plans, actual query timings, rows examined, and application response times. I also check the effect on write performance before keeping the index.

The important point is that an index is not automatically beneficial. The optimizer may correctly choose a table scan if most rows are required or the table is small.

### 8. What Is the N+1 Query Problem?

The N+1 problem occurs when an application executes one query to retrieve a collection of records and then executes another query for each record to retrieve related data.

For example, suppose an API fetches 100 orders and then retrieves each order's restaurant separately:

```sql
SELECT * FROM Orders LIMIT 100;

-- Executed once for every order:
SELECT * FROM Restaurants WHERE Id = @RestaurantId;
```

This produces 1 query for the orders plus 100 additional queries, resulting in **101 database queries**.

Even if each query is individually fast, the repeated database round trips increase latency and consume database resources.

I would solve this using eager loading, a join, or a batch query, depending on the data relationship and the ORM.

For example:

```sql
SELECT
    o.Id,
    o.Status,
    r.Name AS RestaurantName
FROM Orders o
JOIN Restaurants r
    ON r.Id = o.RestaurantId
LIMIT 100;
```

Alternatively, I could fetch the orders first, collect their distinct restaurant IDs, and retrieve all restaurants in one batch using `WHERE Id IN (...)`.

With Entity Framework Core, I would consider projecting directly into a DTO or using appropriate `Include` operations when entity loading is necessary. I would also inspect generated SQL to ensure that eager loading has not introduced excessive joins or duplicated result rows.

**My preference is to fetch the required data in a small, predictable number of queries rather than executing a query inside a loop.**

### Conclusion

When optimizing database performance, I start by understanding the query workload and inspecting the execution plan. I use B+ tree indexes for efficient lookups and range scans, choose composite index order based on query predicates, apply covering indexes selectively, and rewrite predicates to preserve index usability.

I also look beyond SQL itself: I address N+1 queries, unnecessary data retrieval, pagination, and excessive database round trips. Finally, I validate every change with actual execution measurements and monitor its impact on write performance.

*Q6. Transactions, isolation and concurrency. Explain ACID, then the isolation levels (read committed, repeatable read, serializable) and the anomalies each prevents (dirty read, non-repeatable read, phantom). What are pessimistic vs optimistic locking, and how do you prevent double-booking or overselling stock with SELECT ... FOR UPDATE or version columns?*

In a database, transactions and concurrency control ensure that multiple users can access and modify data without producing inconsistent results. For example, in a restaurant management system, two customers might try to book the same table, or two orders might attempt to purchase the last item in stock.

I handle these situations using database transactions, appropriate isolation levels, and pessimistic or optimistic locking, depending on the use case.

### 1. Explain ACID Properties

ACID stands for Atomicity, Consistency, Isolation, and Durability. These properties help ensure that database transactions execute reliably.

**Atomicity:** A transaction is completed entirely or not at all. If an order requires inserting an order record and deducting inventory, both operations should succeed together. If either fails, the transaction is rolled back.

**Consistency:** A transaction moves the database from one valid state to another while preserving constraints and business rules. For example, stock quantity should never become negative if negative inventory is prohibited. I enforce this through database constraints and appropriate application logic.

**Isolation:** Concurrent transactions should not interfere with one another in ways that violate the guarantees of the selected isolation level. For example, two requests should not both successfully reserve the same table.

**Durability:** Once a transaction is committed, its changes survive failures according to the database's durability guarantees.

In practice, I use transactions to group related database operations and rely on constraints and concurrency control to preserve data integrity.

### 2. Transaction Isolation Levels

Isolation levels determine how much one transaction can observe the intermediate or concurrent changes of another transaction.

**Read Uncommitted**

This is the weakest standard isolation level. A transaction may read data written by another transaction that has not committed yet.

For example, Transaction A changes a product's price from ₹500 to ₹800 but has not committed. Transaction B reads ₹800. If Transaction A rolls back, Transaction B has read a value that was never committed.

This is called a **dirty read**.

**Read Committed**

A transaction reads only committed data. It prevents dirty reads, but the same query executed twice within one transaction may return different values if another transaction commits an update between those reads.

For example, Transaction A reads stock as 10. Transaction B updates the stock to 8 and commits. Transaction A reads again and sees 8.

This is a **non-repeatable read**.

Read Committed is commonly used for business applications where committed data visibility is sufficient and explicit locking or atomic updates handle critical concurrent operations.

**Repeatable Read**

This level ensures that repeated reads of the same rows remain consistent within a transaction according to the database's implementation.

It prevents dirty reads and non-repeatable reads under the standard isolation model. However, phantom reads may still be possible under the SQL standard.

A phantom occurs when a transaction repeats a query for a range of rows and sees additional or missing matching rows because another transaction inserted or deleted qualifying records.

For example, Transaction A queries all bookings for table 5 on a particular date and finds none. Transaction B inserts a booking for that table and commits. If Transaction A executes the query again, it may see the newly inserted booking.

Database engines differ in their implementation. For example, InnoDB uses MVCC and consistent snapshots under Repeatable Read, with locking reads and other mechanisms having different behavior from ordinary snapshot reads.

**Serializable**

Serializable is the strongest standard isolation level. It aims to make concurrent transactions behave as though they had executed sequentially.

It prevents the standard dirty-read, non-repeatable-read, and phantom-read anomalies. The database may achieve this through locking, serialization checks, or a combination of techniques.

The trade-off is reduced concurrency and potentially more blocking, deadlocks, or transaction retries.

### 3. Isolation Levels and Anomalies

| Isolation level | Dirty reads | Non-repeatable reads | Phantom reads |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible under the SQL standard |
| Serializable | Prevented | Prevented | Prevented |

Actual behavior can differ by database engine and implementation. In particular, MySQL InnoDB's Repeatable Read provides stronger phantom protection for certain locking operations than the basic SQL-standard description suggests.

### 4. Pessimistic vs Optimistic Locking

**Pessimistic locking**

Pessimistic locking assumes that concurrent conflicts are likely. It locks the relevant rows before modifying them so that another transaction cannot simultaneously perform conflicting operations on those rows.

For example, in a stock management system, I can use `SELECT ... FOR UPDATE` to lock a product's inventory row while processing an order.

```sql
START TRANSACTION;

SELECT Quantity
FROM Inventory
WHERE ProductId = 101
FOR UPDATE;

-- Validate available stock and deduct it.

UPDATE Inventory
SET Quantity = Quantity - 2
WHERE ProductId = 101;

COMMIT;
```

The transaction must check that sufficient stock is available before executing the update. The row lock is held until commit or rollback.

This approach works well for operations such as inventory reservation or table booking, where conflicting requests must be coordinated. However, locks can cause waiting and deadlocks, so transactions should be kept short.

**Optimistic locking**

Optimistic locking assumes that conflicts are relatively uncommon. Instead of locking a row throughout the operation, the application checks whether the data has changed since it was read.

For example, suppose an inventory record contains:

```text
ProductId = 101
Quantity = 10
Version = 5
```

The application reads the record and attempts an update using the original version:

```sql
UPDATE Inventory
SET Quantity = 8,
    Version = Version + 1
WHERE ProductId = 101
  AND Version = 5;
```

If another transaction has already modified the row and incremented its version, this update affects zero rows. The application detects the conflict and retries or returns an appropriate response, such as `409 Conflict`.

Optimistic locking is useful for records that are read frequently but updated infrequently, such as profile settings or administrative forms.

### 5. How Do You Prevent Overselling Stock?

Suppose only one item remains in stock, and two requests arrive simultaneously. Both requests read `Quantity = 1`. If both independently check the quantity and then deduct one, the application can oversell unless the operations are coordinated.

I would use an atomic conditional update or a transaction with row locking.

**Approach A: Atomic conditional update**

```sql
UPDATE Inventory
SET Quantity = Quantity - 1
WHERE ProductId = 101
  AND Quantity >= 1;
```

Then I check the number of affected rows.

- If one row is affected, the stock was successfully reserved.
- If zero rows are affected, the product does not exist or insufficient stock is available.

The application can distinguish those cases with an additional query if necessary.

The database evaluates the condition as part of the update, preventing two concurrent requests from both successfully deducting the same final unit.

**Approach B: Pessimistic locking with a transaction**

```sql
START TRANSACTION;

SELECT Quantity
FROM Inventory
WHERE ProductId = 101
FOR UPDATE;

-- Application verifies Quantity >= 1.

UPDATE Inventory
SET Quantity = Quantity - 1
WHERE ProductId = 101;

COMMIT;
```

The first transaction locks the inventory row. A competing transaction must wait until the lock is released, then reads the current quantity and checks availability again.

For this approach, the validation and update must occur in the same transaction. I would also enforce a database constraint such as `CHECK (Quantity >= 0)` where supported, as an additional integrity safeguard.

For most simple stock deductions, I prefer the atomic conditional update because it is straightforward and minimizes the time locks are held.

### 6. How Do You Prevent Double-Booking?

Suppose two users attempt to book the same restaurant table for the same time slot. Checking availability in application code alone is insufficient because both requests might observe the table as available before either inserts its booking.

I would use a database-enforced strategy.

For a system with fixed booking slots, one approach is to represent each table and time slot as a unique reservable resource:

```sql
CREATE TABLE TableSlots (
    TableId INT NOT NULL,
    SlotStart DATETIME NOT NULL,
    BookingId INT NULL,
    PRIMARY KEY (TableId, SlotStart)
);
```

The composite primary key prevents the same table and slot from being represented more than once. Booking can then be performed using an atomic conditional update or an insert that relies on the unique constraint.

For example, if a slot already exists and is unbooked:

```sql
UPDATE TableSlots
SET BookingId = 5001
WHERE TableId = 5
  AND SlotStart = '2026-10-15 19:00:00'
  AND BookingId IS NULL;
```

If one row is affected, the booking succeeds. If zero rows are affected, the slot is already booked or does not exist.

For arbitrary booking durations, preventing overlapping intervals is more complex. I would use an appropriate database constraint where supported, or transaction-based locking with a carefully designed reservation model that serializes conflicting bookings. A simple unique constraint on the start time alone does not prevent overlapping bookings with different start times.

### 7. Deadlocks and Transaction Retries

A deadlock occurs when transactions wait on each other's locks and neither can proceed.

For example, Transaction A locks product 1 and waits for product 2, while Transaction B locks product 2 and waits for product 1.

I reduce deadlocks by:

- Acquiring locks in a consistent order.
- Keeping transactions short.
- Avoiding unnecessary queries while holding locks.
- Indexing queries so that they lock fewer rows where possible.
- Handling deadlock errors by retrying the entire transaction a limited number of times with backoff.

I do not retry a failed statement blindly when earlier operations in the transaction may already have executed. I retry the transaction according to the database's transaction semantics.

For payment processing or order creation, I also use idempotency keys so that a retry does not create duplicate business operations.

### Conclusion

I use ACID transactions to maintain data integrity, choose isolation levels according to the application's consistency requirements, and select pessimistic or optimistic locking based on the likelihood of concurrent conflicts.

For overselling, I prefer an atomic conditional update or row locking. For double-booking, I combine transaction control with database-enforced uniqueness or an appropriate reservation constraint. Finally, I handle deadlocks, retries, and duplicate requests explicitly rather than assuming that application-level checks alone are sufficient.

*Q7. Schema design and normalization. When do you normalize to 3NF and when do you denormalize on purpose? How do you model many-to-many relations, multi-tenancy (shared schema with tenant_id vs separate schemas), soft deletes, audit trails and money (integer minor units vs DECIMAL)? What makes a migration safe on a live table?*

Database schema design is about organizing data to maintain consistency, avoid unnecessary duplication, and support efficient queries. When designing a schema, I consider data integrity, query patterns, scalability, and how the application will evolve.

### 1. Normalization: 3NF vs Denormalization

**Normalization** means organizing tables to reduce redundant data and prevent insertion, update, and deletion anomalies.

I generally normalize transactional databases to **Third Normal Form (3NF)** because it helps maintain data consistency.

For example, consider a restaurant management system where every order stores the restaurant's name:

```sql
Orders(
    OrderId,
    RestaurantId,
    RestaurantName,
    TotalAmount
)
```

If thousands of orders belong to the same restaurant, its name is duplicated across many rows. If the restaurant changes its name, multiple records may need updating.

Instead, I separate the entities:

```sql
Restaurants(
    RestaurantId PRIMARY KEY,
    RestaurantName
)

Orders(
    OrderId PRIMARY KEY,
    RestaurantId FOREIGN KEY,
    TotalAmount
)
```

Now, the restaurant's name is stored in one place, and orders reference it through a foreign key.

In 3NF, the table is in 2NF, and non-key attributes do not depend transitively on a candidate key. In practical terms, each non-key attribute should depend on the key, the whole key, and nothing but the key.

**When I normalize:**
- When multiple records share the same entity information.
- When data is frequently updated.
- When consistency and referential integrity are important.
- When designing transactional systems such as orders, inventory, payments, and bookings.

**Denormalization** intentionally introduces redundancy to improve read performance or simplify frequently executed queries.

For example, I might store `TotalAmount` on an order rather than calculate it from all order items every time. This is also important for preserving the historical total at the time of purchase, provided the application maintains it consistently.

I might also maintain precomputed sales summaries for dashboards rather than aggregating millions of order records on every request.

Denormalization introduces additional consistency responsibilities. Therefore, I use it only when profiling or workload requirements justify it, and I define how the duplicated data will be updated.

### 2. How Do You Model Many-to-Many Relationships?

A many-to-many relationship exists when multiple records from one entity can be associated with multiple records from another entity.

For example, a product can belong to multiple categories, and each category can contain multiple products.

I model this using a junction table:

```sql
CREATE TABLE Products (
    Id INT PRIMARY KEY,
    Name VARCHAR(150) NOT NULL
);

CREATE TABLE Categories (
    Id INT PRIMARY KEY,
    Name VARCHAR(100) NOT NULL
);

CREATE TABLE ProductCategories (
    ProductId INT NOT NULL,
    CategoryId INT NOT NULL,

    PRIMARY KEY (ProductId, CategoryId),

    FOREIGN KEY (ProductId) REFERENCES Products(Id),
    FOREIGN KEY (CategoryId) REFERENCES Categories(Id)
);
```

The `ProductCategories` table resolves the many-to-many relationship into two one-to-many relationships.

The composite primary key prevents the same product-category association from being inserted twice.

I would also add indexes based on access patterns. The composite primary key supports lookups beginning with `ProductId`; if the application frequently retrieves products by `CategoryId`, I would add an appropriate index on that column.

If the relationship has additional attributes, such as `AssignedAt`, `Quantity`, or `CreatedBy`, I store those attributes in the junction table.

### 3. Multi-Tenancy: Shared Schema vs Separate Schemas

Multi-tenancy means supporting multiple customers or organizations within the same application while ensuring that each tenant's data remains isolated.

For a restaurant ERP SaaS, each restaurant or business tenant must be prevented from accessing another tenant's orders, employees, inventory, and reports.

**Approach A: Shared database and shared schema**

All tenants share the same tables, and each tenant-owned record contains a `TenantId`.

```sql
CREATE TABLE Orders (
    Id BIGINT PRIMARY KEY,
    TenantId BIGINT NOT NULL,
    TotalAmount DECIMAL(12, 2) NOT NULL
);

CREATE INDEX idx_orders_tenant
ON Orders (TenantId, Id);
```

Advantages:
- Lower infrastructure and maintenance costs.
- Easier centralized migrations.
- Efficient resource utilization.
- Straightforward support for a large number of small tenants.

Disadvantages:
- Every relevant query must enforce tenant isolation.
- A missing tenant filter can expose another tenant's data.
- Heavy tenants may compete for shared resources.

For a multi-tenant ERP serving many small restaurants, I would generally start with a shared schema and `TenantId`, combined with strong authorization checks and database constraints.

For tenant-owned relationships, I would consider composite foreign keys involving `TenantId` to prevent records from referencing entities belonging to another tenant.

**Approach B: Separate schema per tenant**

Each tenant has its own schema containing its tables.

Advantages:
- Better logical separation.
- Easier tenant-specific schema permissions in some architectures.
- Potentially simpler tenant-specific data operations.

Disadvantages:
- Migrations must be coordinated across many schemas.
- Schema management and monitoring become more complex.
- Database-level isolation still depends on permissions and configuration.

**Approach C: Separate database per tenant**

Each tenant has its own database. This can provide stronger isolation, independent backups, and tenant-specific scaling, but increases infrastructure and operational costs.

My choice depends on the isolation requirements, number of tenants, compliance needs, and operational budget. Shared schema is a practical starting point for a growing SaaS, while larger or regulated customers may justify dedicated databases.

### 4. Soft Deletes vs Hard Deletes

A hard delete permanently removes a record:

```sql
DELETE FROM Orders
WHERE Id = 101;
```

A soft delete marks the record as deleted without physically removing it.

For example:

```sql
ALTER TABLE Orders
ADD COLUMN DeletedAt DATETIME NULL;
```

Then:

```sql
UPDATE Orders
SET DeletedAt = CURRENT_TIMESTAMP
WHERE Id = 101;
```

Normal queries exclude deleted records:

```sql
SELECT *
FROM Orders
WHERE TenantId = @TenantId
  AND DeletedAt IS NULL;
```

**I use soft deletes when:**
- Users need to restore accidentally deleted records.
- Business records must remain available for auditing.
- Historical references should be preserved.
- The application needs a recoverable deletion workflow.

However, soft deletes do not automatically satisfy legal retention or deletion requirements. Sensitive data may still need to be permanently erased or anonymized according to applicable policies.

I also account for soft deletion in uniqueness rules. For example, if a restaurant's email address should be reusable after deletion, the database's unique-index design must allow that behavior. The exact solution depends on the database engine and its support for partial or generated-column-based unique indexes.

### 5. Audit Trails

An audit trail records who performed an action, what changed, and when it happened.

For basic auditing, I might add:

```sql
CreatedAt
CreatedBy
UpdatedAt
UpdatedBy
```

These fields help track record creation and the most recent update, but they do not provide a complete history of all changes.

For more detailed auditing, I would create a separate audit table:

```sql
CREATE TABLE AuditLogs (
    Id BIGINT PRIMARY KEY,
    TenantId BIGINT NOT NULL,
    UserId BIGINT NULL,
    EntityType VARCHAR(100) NOT NULL,
    EntityId BIGINT NOT NULL,
    Action VARCHAR(30) NOT NULL,
    OldValues JSON NULL,
    NewValues JSON NULL,
    CreatedAt DATETIME NOT NULL
);
```

For example, when a manager changes an inventory quantity, the audit record could store the previous value, the new value, the responsible user, and the timestamp.

For reliable business auditing, I would write the audit record in the same database transaction as the business change when both are stored in the same database.

I would also restrict modification and deletion of audit records, define a retention policy, and avoid storing unnecessary sensitive information in audit payloads.

For changes that must be captured even when they originate outside the application, database auditing or carefully designed triggers may be appropriate.

### 6. How Should Money Be Stored?

For monetary values, I avoid floating-point types such as `FLOAT` and `DOUBLE` because binary floating-point arithmetic can introduce rounding errors.

I generally consider two approaches.

**Approach A: Integer minor units**

Store amounts in the smallest relevant currency unit.

For example:

```text
₹150.50 = 15050 paise
```

A database column could be:

```sql
AmountMinorUnits BIGINT NOT NULL
```

Advantages:
- Exact integer arithmetic for values within the supported range.
- Convenient for many payment-provider integrations.
- No binary floating-point representation errors.

However, not all currencies use two decimal places, and some calculations can produce fractional minor units. Tax, discounts, foreign exchange, and rounding policies must therefore be handled explicitly.

**Approach B: DECIMAL**

For financial applications, I often use `DECIMAL` with an appropriate precision and scale:

```sql
Amount DECIMAL(12, 2) NOT NULL
```

`DECIMAL` stores exact decimal values within its defined precision and scale, making it suitable for prices, taxes, and monetary calculations.

For a restaurant ERP, I would generally use `DECIMAL` for stored monetary amounts and define explicit rounding rules for taxes, discounts, and totals. If a payment gateway uses integer minor units, I would convert to that representation at the integration boundary.

I would also store the currency explicitly, because an amount without a currency is ambiguous.

The key is to use exact numeric representations, choose appropriate precision, and apply a consistent rounding policy throughout the application.

### 7. What Makes a Database Migration Safe on a Live Table?

A migration is safe when it can be applied to a running production system without unexpected downtime, data loss, or breaking currently deployed application versions.

For large tables, I follow an **expand-and-contract migration strategy**.

Suppose I need to rename a column from `CustomerName` to `FullName`.

Renaming it immediately could break the existing application version, which still expects `CustomerName`.

Instead, I use these stages:

**Step 1: Expand the schema.**

Add the new column while retaining the old one.

```sql
ALTER TABLE Customers
ADD COLUMN FullName VARCHAR(150) NULL;
```

I first check the database engine's DDL behavior, locking requirements, and whether the operation can be performed online.

**Step 2: Deploy backward-compatible application code.**

Update the application to support the new column without requiring every running instance to switch at exactly the same moment.

Depending on the change, the application may temporarily read the old column and write both columns.

**Step 3: Backfill existing data.**

Populate `FullName` from `CustomerName` in controlled batches. For a large table, I avoid one enormous update that could create excessive locking, transaction logs, replication lag, or resource contention.

I make the backfill restartable and verify that the migrated values are correct.

**Step 4: Switch application reads and writes.**

After confirming that the new column is populated and synchronized, deploy application code that uses `FullName`.

I monitor errors and data consistency during the transition.

**Step 5: Contract the schema.**

Only after all application instances and dependent processes no longer use `CustomerName` do I remove the old column in a later migration.

Other precautions I take include:

- Backing up important data and testing recovery procedures.
- Testing migrations against production-like data volumes.
- Checking index creation and constraint-validation costs.
- Using online DDL capabilities where supported.
- Monitoring locks, replication lag, and application latency.
- Making migrations repeatable or safely resumable where practical.
- Avoiding destructive schema changes in the same deployment that first introduces their replacements.

I do not assume every DDL statement is non-blocking. The exact migration strategy depends on the database engine, version, table size, and operational requirements.

### Conclusion

I generally normalize transactional data to 3NF to maintain consistency, denormalize selectively when a measured workload justifies it, and model many-to-many relationships using junction tables with appropriate constraints.

For multi-tenancy, I choose the isolation model based on security and operational requirements. I use soft deletes and audit trails when business history matters, store money using exact numeric representations, and apply backward-compatible expand-and-contract migrations for live systems.

My overall principle is to enforce data integrity at the database level wherever possible, keep the schema aligned with real application access patterns, and make production changes incrementally with a clear validation and rollback strategy.

*Q8. Caching strategies. Compare cache-aside, read-through, write-through and write-behind. How do you handle invalidation, TTLs, cache stampede (locking, request coalescing, jitter) and stale data? What belongs in Redis vs in-process memory vs HTTP/CDN caching?*

Caching is a technique used to store frequently accessed data in a faster storage layer so that the application can reduce database queries, improve response times, and handle more requests.

For example, in a restaurant ERP, menu items, restaurant settings, and product categories may be requested frequently but updated relatively infrequently. Instead of querying the database for every request, I can cache this data using Redis or in-process memory.

However, caching introduces challenges such as stale data, cache invalidation, memory limits, and concurrent cache misses. I choose the strategy based on the data's consistency requirements and access patterns.

### 1. Cache-Aside vs Read-Through vs Write-Through vs Write-Behind

**Cache-Aside (Lazy Loading)**

In cache-aside, the application is responsible for checking the cache and loading data from the database when the cache misses.

The flow is:

1. The application checks Redis for the requested data.
2. If the data exists, it returns the cached value.
3. If the data is missing, the application queries the database.
4. The application stores the result in Redis with a TTL and returns it.

Example:

```csharp
var cachedMenu = await redis.GetAsync<Menu>("menu:101");

if (cachedMenu != null)
    return cachedMenu;

var menu = await db.GetMenuAsync(101);

if (menu != null)
    await redis.SetAsync("menu:101", menu, TimeSpan.FromMinutes(10));

return menu;
```

The exact cache API depends on the Redis client and serialization implementation.

The main advantage is that only requested data is loaded into the cache. The disadvantage is that the first request after expiration or eviction must query the database.

I generally prefer cache-aside for business applications because it is simple, flexible, and works well with Redis.

**Read-Through Caching**

In read-through caching, the application requests data from the cache layer, and the cache layer itself loads the data from the underlying data source when a cache miss occurs.

The application does not explicitly implement the cache-miss database-loading logic for every request.

This can simplify application code, but it requires a caching layer or library that supports the necessary loading behavior.

**Write-Through Caching**

In write-through caching, a write is propagated to the cache and the underlying data store as part of the write operation's defined workflow.

The goal is to keep cached data synchronized with persistent data, reducing the window in which readers observe outdated values.

However, Redis and a relational database do not automatically share one atomic transaction. If the database update succeeds but the cache update fails, the system needs a recovery strategy, such as invalidation, retries, or reconciliation.

For many business applications, I prefer updating the database first and then invalidating the affected cache entry rather than relying on a supposedly atomic write-through operation across two separate systems.

**Write-Behind (Write-Back) Caching**

In write-behind caching, the application writes to the cache first and persists the change to the database asynchronously.

This can reduce write latency and batch multiple database writes, but it introduces a risk: if the cache or asynchronous processing system fails before the data is persisted, changes may be lost.

I would use this approach only when the system can tolerate delayed persistence and has reliable mechanisms such as durable queues, retry processing, and monitoring.

I would not use a simple Redis cache as the only durable record of payments, inventory deductions, or financial transactions.

**Comparison**

| Strategy | Read behavior | Write behavior | Main concern |
|---|---|---|---|
| Cache-aside | Application loads on miss | Application updates DB and manages cache | Invalidation |
| Read-through | Cache layer loads on miss | Depends on implementation | Cache-layer complexity |
| Write-through | Reads from cache | Writes propagate to DB | Write latency and partial failures |
| Write-behind | Reads from cache | DB persistence is deferred | Data-loss risk and consistency |

### 2. Cache Invalidation and TTLs

Cache invalidation means ensuring that cached data does not remain incorrect after the underlying data changes.

Suppose an administrator updates a restaurant's menu price. If the old menu remains in Redis, customers may continue seeing the previous price until the cache expires.

One common approach is **TTL-based expiration**.

For example, I might cache menu data for 10 minutes:

```text
Key: menu:restaurant:101
TTL: 600 seconds
```

After 10 minutes, Redis expires the entry, and the next request reloads it from the database.

TTL is useful because it limits how long a stale value can remain cached, but it does not guarantee immediate consistency.

For data that changes frequently, I combine TTL with explicit invalidation.

A common cache-aside write flow is:

1. Update the database successfully.
2. Commit the transaction.
3. Delete the affected cache key.
4. Let the next read reload the latest committed data.

For example:

```text
UPDATE MenuItems in database
COMMIT
DELETE menu:restaurant:101 from Redis
```

I generally prefer invalidation over manually updating a complex cached object because it reduces the chance of inconsistent cache contents.

However, a failure can occur after the database commits but before Redis is invalidated. For important workflows, I would use a retry mechanism or transactional outbox to publish a durable cache-invalidation event.

For data that must be strongly consistent, such as available stock during checkout, I do not rely on a cached value as the final authority. I validate and modify the authoritative database state within an appropriate transaction.

### 3. What Is a Cache Stampede?

A cache stampede occurs when a popular cache entry expires or becomes unavailable and many requests simultaneously query the database to rebuild it.

For example, if 500 requests arrive for the same expired menu cache, all 500 might execute the same database query. This increases database load and can make the application slower precisely when traffic is high.

I use several techniques to prevent this.

**A. Request Coalescing**

Request coalescing ensures that concurrent requests for the same cache key share one in-flight data-loading operation.

For example, if 100 requests need `menu:101`, one request loads the menu from the database while the others await the same result.

This is particularly useful within a single application process.

However, a local in-flight request map does not coordinate separate application instances. In a distributed deployment, each instance may still issue its own database query.

**B. Distributed Locking**

For highly contended cache keys across multiple instances, I can use a Redis lock.

The general flow is:

1. A request detects a cache miss.
2. It attempts to acquire a short-lived lock for the cache key.
3. The lock holder loads the data from the database and populates Redis.
4. Other requests wait briefly, retry the cache lookup, or use an acceptable stale value.

The lock must have a bounded lifetime, and ownership must be checked when releasing it. If a lock expires while the original loader is still running, duplicate loads can still happen. For critical operations, I also consider fencing or version checks to prevent an outdated loader from overwriting newer data.

I do not introduce distributed locking for every cache miss because it adds complexity and network overhead. For many applications, request coalescing and TTL jitter are sufficient.

**C. TTL Jitter**

If thousands of cache entries receive exactly the same TTL, they may expire together and cause a burst of database queries.

I avoid synchronized expiration by adding a random variation to the TTL.

For example:

```text
Base TTL: 600 seconds
Actual TTL: 600 + random(0, 60) seconds
```

This spreads expirations over time and reduces synchronized cache misses.

**D. Stale-While-Revalidate**

For data where a slightly outdated value is acceptable, I can continue serving a previously cached value while refreshing it in the background.

For example, a restaurant dashboard might display a sales summary that is a few seconds old while the system refreshes the latest aggregate.

I would not apply this blindly to payments, stock availability, or other operations where stale data could cause incorrect business decisions.

### 4. How Do You Handle Stale Data?

There is no single caching strategy that guarantees both maximum performance and immediate consistency across independent systems. I choose an approach based on how incorrect or outdated the data can be.

For relatively stable data, such as categories or restaurant settings, a TTL with explicit invalidation may be sufficient.

For frequently changing data, I may use shorter TTLs, event-driven invalidation, or versioned cache keys.

For critical operations, I use the database as the source of truth and perform the necessary validation inside a transaction.

I also consider race conditions during cache invalidation. For example:

1. Request A reads an old value from the database.
2. Request B updates the database and deletes the cache key.
3. Request A writes its previously read value into Redis.

The cache now contains stale data even though invalidation occurred.

To address this, I can use versioned values, conditional cache writes, coordinated refreshes, or generation-based cache keys. For high-value data, I prefer an approach that prevents an older loader from overwriting a newer version.

### 5. What Belongs in Redis vs In-Process Memory vs HTTP/CDN Cache?

I choose the cache layer based on the data's scope, size, lifetime, and access pattern.

**In-process memory cache**

An in-process cache, such as `IMemoryCache` in ASP.NET Core, stores data inside the application's memory.

I use it for small, frequently accessed data that is safe to duplicate across application instances, such as static configuration or small reference datasets.

Advantages:
- Very low access latency.
- No network round trip.
- Simple implementation.

Disadvantages:
- Each application instance has its own copy.
- Entries disappear when the process restarts.
- Invalidation across multiple instances is difficult without additional coordination.

**Redis**

Redis is an external, shared in-memory data store that can be accessed by multiple application instances.

I use it for:
- Shared cache-aside data.
- Frequently accessed database results.
- Distributed rate-limiting counters.
- Short-lived coordination state and carefully designed distributed locks.
- Session data when using distributed session management.

Redis can also support durable or persistent data structures depending on its configuration, but I would not assume that a cache deployment provides the durability guarantees required for financial records.

The main trade-off is the additional network call and the operational responsibility of maintaining the Redis service.

**HTTP and CDN caching**

HTTP caching operates at the HTTP layer and can allow browsers, proxies, or CDNs to reuse responses without contacting the origin application.

I use it for:
- Static JavaScript, CSS, fonts, and images.
- Public product images and other cacheable assets.
- Public API responses when their freshness and privacy requirements permit caching.

For example:

```http
Cache-Control: public, max-age=31536000, immutable
```

This is appropriate for versioned static assets whose URLs change when their contents change.

For user-specific responses, I choose cache directives carefully. For sensitive payment or account responses, `Cache-Control: no-store` may be appropriate.

A CDN can significantly reduce origin traffic, but it requires correct cache keys and invalidation rules. I avoid accidentally caching personalized responses in shared caches.

**Comparison**

| Cache layer | Scope | Best use case |
|---|---|---|
| In-process memory | One application instance | Small, local, frequently read data |
| Redis | Shared across instances | Shared application data and coordination |
| Browser cache | Individual client | Static assets and suitable HTTP responses |
| CDN | Distributed edge locations | Public static assets and cacheable content |

### 6. My Practical Caching Strategy

For a restaurant ERP built with React and ASP.NET Core, I would use the following approach:

1. **Menu and category reads:** Redis cache-aside with a moderate TTL and explicit invalidation after updates.
2. **Small, stable configuration:** In-process memory cache where duplication across instances is acceptable.
3. **Static frontend assets:** Browser and CDN caching with content-hashed filenames and long-lived cache headers.
4. **Dashboard summaries:** Redis caching with controlled refreshes or stale-while-revalidate where a small delay is acceptable.
5. **Payments and inventory:** Database transactions and atomic updates remain authoritative; cache values are not used to guarantee correctness.
6. **High-traffic cache misses:** Request coalescing, TTL jitter, and distributed coordination only when justified by measured contention.
7. **Monitoring:** Track cache hit ratio, miss rate, latency, eviction rate, memory usage, database load, and stale-data incidents.

### Conclusion

I choose cache-aside for most application-level caching because it is simple and flexible, use TTLs and explicit invalidation to control freshness, and prevent cache stampedes through request coalescing, TTL jitter, or distributed locking when necessary.

I use in-process memory for local data, Redis for shared application caching, and HTTP/CDN caching for appropriate client-facing content. Most importantly, I never treat a cache as the source of truth for a business operation that requires strict consistency.

*Q9. Concurrency, threading and async I/O. How do thread-per-request, event-loop and async/await server models differ, and which workloads suit each? What are race conditions, deadlocks and thread pool starvation? How do connection pools work, and why can a bad pool size or a leaked connection take a whole service down?*

Concurrency means handling multiple operations during overlapping periods, while parallelism means executing multiple operations at the same time. They are related, but they are not the same.

In a backend application, concurrency is important because many requests may arrive simultaneously, and each request may involve database queries, external API calls, file operations, or CPU-intensive processing.

### 1. Thread-per-Request vs Event Loop vs Async/Await

**Thread-per-request model**

In this model, the server assigns a dedicated thread to handle each incoming request. The thread executes the request handler and may block while waiting for a database query or an external API response.

For example, if 100 requests arrive concurrently, the server may use many threads to handle them.

The advantage is that the programming model is straightforward, especially when using synchronous code. The disadvantage is that threads consume memory and scheduling resources. If many threads spend time blocked on I/O, the application can become inefficient or run out of available threads.

This model can work well for moderate workloads and applications where synchronous programming is acceptable, but the number of concurrent requests is constrained by available resources and server configuration.

**Event-loop model**

An event loop uses a small number of threads to coordinate many concurrent operations. When an operation is waiting for I/O, the event loop can process other ready work instead of blocking the thread.

For example, in Node.js, a request handler can initiate a database query and allow the event loop to process other callbacks while waiting for the result.

This model is particularly effective for I/O-heavy workloads, such as APIs that handle many concurrent network requests.

However, CPU-intensive synchronous work can block the event loop and delay unrelated requests. CPU-heavy tasks should be moved to worker threads, a separate process, or a background worker when appropriate.

**Async/await server model**

`async/await` is a programming model for expressing asynchronous operations in a readable, sequential style. It is not a separate concurrency architecture by itself.

In ASP.NET Core, asynchronous request handlers typically use the thread pool to execute application code. When a genuinely asynchronous I/O operation is pending, the request does not need to keep a worker thread blocked while waiting. A worker can execute other work, and the continuation resumes when the operation completes.

For example:

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _db.Orders.FindAsync(id);

    if (order == null)
        return NotFound();

    return Ok(order);
}
```

The asynchronous database operation allows the request to wait without synchronously blocking a thread, assuming the database provider implements asynchronous I/O properly.

**Comparison**

| Model | Main characteristic | Best suited for |
|---|---|---|
| Thread-per-request | A thread handles each request | Simpler synchronous workloads |
| Event loop | Coordinates many operations on a small number of threads | High-concurrency, I/O-heavy workloads |
| Async/await | Makes asynchronous operations easier to express | I/O-heavy APIs and scalable backend services |

In practice, I would use asynchronous I/O for database calls, HTTP requests, and file operations in ASP.NET Core or Node.js applications, while keeping CPU-intensive processing separate from request-handling threads when necessary.

### 2. What Is a Race Condition?

A race condition occurs when the result of an operation depends on the timing or ordering of concurrent operations, and that ordering can produce an incorrect result.

For example, suppose the inventory quantity is 1 and two requests try to purchase the last item.

Both requests read the quantity as 1. Both conclude that stock is available, and both attempt to deduct one item.

Without proper concurrency control, the application may oversell the product.

I prevent this by enforcing correctness at the database level, using an atomic conditional update or a transaction with appropriate row locking.

For example:

```sql
UPDATE Inventory
SET Quantity = Quantity - 1
WHERE ProductId = 101
  AND Quantity >= 1;
```

I then check the number of affected rows. Only one competing request can successfully deduct the final unit.

Race conditions can also occur in application memory when multiple threads modify shared state. Depending on the situation, I use locks, thread-safe data structures, atomic operations, or message-based coordination.

The important principle is that checking a condition and updating the state must be coordinated when concurrent requests could invalidate that condition.

### 3. What Is a Deadlock?

A deadlock occurs when two or more operations wait indefinitely for resources held by one another.

For example:

- Thread A acquires Lock 1 and waits for Lock 2.
- Thread B acquires Lock 2 and waits for Lock 1.
- Neither thread can proceed because each is waiting for the other.

I reduce deadlocks by acquiring locks in a consistent order, keeping critical sections short, avoiding unnecessary locks, and not holding locks while waiting for external API calls.

Database deadlocks can occur for similar reasons when transactions acquire row or index locks in conflicting orders.

When a database detects a deadlock, it typically aborts one transaction. The application should handle the error and, where appropriate, retry the entire transaction a limited number of times with backoff.

Retries must be designed carefully so that they do not duplicate external side effects, such as payments.

### 4. What Is Thread Pool Starvation?

Thread pool starvation occurs when the available worker threads are occupied or blocked for so long that other work cannot obtain a thread promptly.

For example, suppose an ASP.NET Core application handles many requests using synchronous database calls. Each request blocks a worker thread while waiting for the database. As concurrent requests increase, more workers become occupied, and new requests wait for available threads.

The result can be increased latency, request timeouts, and declining throughput even when CPU utilization is relatively low.

I prevent this by:

- Using asynchronous I/O instead of blocking on I/O operations.
- Avoiding `.Result` and `.Wait()` on asynchronous tasks in request-handling code.
- Avoiding synchronous locks around long-running operations.
- Moving CPU-intensive work to appropriate background workers.
- Setting timeouts and limiting concurrent operations against downstream services.
- Monitoring thread pool queues, request latency, and downstream dependency timings.

Increasing the thread pool size alone is not always the correct solution. If the real problem is a slow database or blocked downstream service, additional threads can increase resource contention rather than solve it.

### 5. How Do Connection Pools Work?

Creating a database connection involves work such as establishing a network connection, negotiating protocol details, and authenticating with the database.

Creating a new physical connection for every query would add unnecessary overhead.

A connection pool maintains reusable database connections. When an application needs a connection, it borrows one from the pool, executes its database operations, and returns it when finished.

For example, in an ASP.NET Core application using MySQL:

```csharp
await using var connection =
    new MySqlConnection(connectionString);

await connection.OpenAsync();

await using var command = new MySqlCommand(
    "SELECT Id, Name FROM Products WHERE Id = @id",
    connection);

command.Parameters.AddWithValue("@id", productId);

await using var reader = await command.ExecuteReaderAsync();
```

With a properly configured pooling-enabled provider, opening the logical connection generally borrows a physical connection from the pool when one is available.

Disposing the connection returns it to the pool rather than necessarily closing the underlying network connection.

A connection pool usually has a maximum size. If all connections are busy, new requests wait until a connection becomes available or a timeout occurs.

### 6. Why Can a Bad Pool Size Take Down a Service?

Connection pool sizing is a balance between application concurrency and database capacity.

**If the pool is too small:**

- Requests spend time waiting for a connection.
- Request latency increases.
- Timeouts become more frequent.
- Threads or asynchronous request operations remain occupied waiting for database access.

For example, if a pool allows only 10 concurrent connections but the application has 200 requests that simultaneously need database access, many requests may queue.

**If the pool is too large:**

- The application may open too many concurrent database connections.
- Database memory and CPU usage can increase.
- Lock contention and query contention can worsen.
- The database may reach its connection limit, affecting other applications or services.

Pool sizes must be considered across all application instances, not just one instance.

For example, if 10 application instances each permit 100 database connections, the deployment may allow up to 1,000 pooled connections, potentially exceeding the database's capacity.

I would size pools according to measured query latency, database capacity, expected concurrency, and the number of deployed instances. I would also set command timeouts, connection-acquisition timeouts, and appropriate concurrency limits.

### 7. Why Can a Leaked Connection Take Down a Service?

A connection leak occurs when application code obtains a database connection but fails to release it properly.

For example, if an exception occurs before the connection is disposed, that connection may remain unavailable to other requests.

As more connections leak, the pool gradually becomes exhausted. New requests can no longer acquire connections, and endpoints that depend on the database begin timing out.

I prevent this by using deterministic resource disposal, such as `using` or `await using` in C#, and ensuring that connections, commands, and readers are disposed even when exceptions occur.

I also avoid holding connections open while performing unrelated work, waiting for user input, or calling external APIs.

For long-running transactions, I keep the transaction scope as small as possible so that connections and database locks are not held unnecessarily.

### 8. How Do You Diagnose a Concurrency or Connection Pooling Problem?

When an API becomes slow under load, I investigate the entire request path rather than assuming the issue is CPU or threading.

I would check:

1. **Request metrics:** Request rate, p95/p99 latency, timeouts, and error rates.
2. **Thread pool metrics:** Worker thread availability, queue length, and signs of blocking.
3. **Database metrics:** Active connections, connection acquisition waits, query latency, locks, and deadlocks.
4. **Application code:** Synchronous I/O, connection leaks, long-running transactions, and shared mutable state.
5. **Load testing:** Whether latency increases gradually or collapses after a concurrency threshold.

For example, if CPU utilization is low but request latency and database connection waits are high, I would investigate blocked I/O, pool exhaustion, slow queries, and leaked connections before adding more server resources.

### Conclusion

I use asynchronous I/O to handle high-concurrency workloads efficiently, while recognizing that `async/await` does not automatically make CPU-intensive work faster or eliminate race conditions.

I prevent race conditions through proper synchronization and database constraints, reduce deadlocks through consistent locking strategies, and avoid thread pool starvation by eliminating unnecessary blocking.

Finally, I treat database connection pools as a shared, finite resource. Correct pool sizing, deterministic connection disposal, short transactions, and monitoring are essential to keeping a backend service responsive under load.

*Q10. Background jobs and messaging. When do you move work to a queue (emails, report generation, webhooks)? Compare RabbitMQ, Kafka and SQS in terms of delivery guarantees. Explain at-least-once vs exactly-once, idempotent consumers, retries with backoff, dead-letter queues, and the outbox pattern for keeping the database and queue consistent.*

## 1. When do you move work to a background queue?

I move work to a background queue when a task does not need to be completed within the user's immediate HTTP request, takes significant time, or needs reliable retries.

For example, when a user places an order, the API should validate the request, save the order, and return a response quickly. Sending confirmation emails, generating invoices, updating analytics, and notifying external services can happen asynchronously through background workers.

**The typical flow is:**

1. The API receives the request and validates it.
2. The application saves the required data to the database.
3. A job or message is published to a queue.
4. A background worker consumes the message and performs the task.
5. The worker acknowledges successful processing or retries the message if processing fails.

Common use cases include:
- **Emails and SMS:** Send notifications without making the user wait for an external email or SMS provider.
- **Report generation:** Generate large PDF, Excel, or CSV reports in the background and notify the user when they are ready.
- **Webhooks:** Deliver events to third-party systems with retries if their endpoints are temporarily unavailable.
- **Image processing:** Resize, compress, or generate thumbnails after an upload.
- **Scheduled tasks:** Process subscriptions, reminders, or periodic billing.

I generally keep the synchronous path limited to operations that must succeed before returning the response. However, if the task is essential to the transaction, such as reserving inventory or authorizing payment, I do not blindly move it to a background job. I first define the business consistency requirements.

## 2. RabbitMQ vs Kafka vs Amazon SQS

All three support asynchronous processing, but they solve somewhat different problems.

| Feature | RabbitMQ | Apache Kafka | Amazon SQS |
|---|---|---|---|
| Main purpose | Message broker and task distribution | Durable event streaming and replay | Managed message queue |
| Message model | Exchanges, queues, bindings | Topics, partitions, consumer groups | Standard and FIFO queues |
| Consumption | Messages are typically acknowledged and removed from the queue | Events remain for a configured retention period | Messages are hidden during processing and deleted after successful handling |
| Ordering | Can preserve queue order under suitable conditions; concurrency can affect processing order | Ordered within a partition | Standard queues do not guarantee strict ordering; FIFO queues preserve order within message groups |
| Delivery | Commonly at-least-once with acknowledgements and suitable configuration | Commonly at-least-once; idempotent producers and transactions provide additional guarantees | Standard is at-least-once; FIFO offers deduplication and ordering features |
| Replay | Requires application-level retention or another mechanism | Built-in replay while events remain retained | Not a general-purpose retained event log; messages are deleted after successful processing |
| Operations | Self-managed or hosted options | Usually more involved to operate, though managed services exist | Fully managed by AWS |

**When I would choose each:**

- **RabbitMQ:** For background jobs such as email delivery, invoice generation, and routing messages to different worker queues. Its routing, acknowledgements, and retry/dead-letter mechanisms are useful for task processing.
- **Kafka:** For high-volume event streams, event-driven architectures, activity tracking, analytics pipelines, and multiple independent consumers that need to replay historical events.
- **Amazon SQS:** For a straightforward, scalable background-job system on AWS where I want AWS to manage most of the broker infrastructure.

The choice depends on throughput, ordering, replay requirements, operational complexity, and the cloud environment. I would not choose Kafka simply because an application has background jobs; a conventional queue is often simpler.

## 3. At-least-once vs exactly-once delivery

**At-least-once delivery** means a message will be delivered one or more times, assuming the system continues retrying and eventually recovers. The same message can be processed more than once.

For example, a worker processes an order notification successfully but crashes before acknowledging the message. The broker may deliver that message again, causing the worker to repeat the operation.

**Exactly-once processing** means the business effect happens only once, even if messages are delivered repeatedly or failures occur. This is harder than it sounds because a broker cannot automatically guarantee that an external database update, payment API call, and message acknowledgement all happen atomically.

Some platforms provide exactly-once guarantees within specific boundaries. For example, Kafka transactions can atomically commit consumed offsets and produced Kafka records. However, that does not automatically make an external database update or payment-provider call exactly once.

In most business applications, I design for **at-least-once delivery with idempotent processing**. It is practical, reliable, and protects against duplicate side effects.

## 4. How do you implement an idempotent consumer?

An idempotent consumer produces the same business result if it processes the same message multiple times.

For example, suppose a message asks a worker to mark order `ORD-101` as paid. If the worker receives the message twice, it should not create two payments or deduct the amount twice.

A common approach is to assign each message a unique `messageId` and maintain a processed-message table.

```sql
CREATE TABLE ProcessedMessages (
    MessageId VARCHAR(100) PRIMARY KEY,
    ProcessedAt TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

The worker then performs the deduplication check and business operation in the same database transaction:

```text
BEGIN TRANSACTION

1. Insert messageId into ProcessedMessages.
2. If the unique key already exists, treat the message
   as a duplicate and skip the business operation.
3. Otherwise, apply the business update.
4. Commit the transaction.

Acknowledge the message only after the commit succeeds.
```

The unique constraint is essential because two workers might receive the same message concurrently. An application-level check alone can allow both workers to pass the check before either records the message.

If the transaction fails, both the business update and the deduplication record should roll back. The message can then be retried safely.

For external side effects, such as charging a payment provider or sending a webhook, a database transaction cannot make the external call atomic. I would use the provider's idempotency key if supported, or maintain a durable delivery state and reconcile uncertain outcomes.

## 5. Retries with exponential backoff

A temporary failure should not necessarily cause a job to fail permanently. For example, an email provider might return HTTP `503`, or a webhook endpoint might time out.

Instead of retrying immediately in a tight loop, I use **exponential backoff**, increasing the delay after each failed attempt.

A typical policy is:

```text
Attempt 1: immediate
Attempt 2: wait 1 second
Attempt 3: wait 2 seconds
Attempt 4: wait 4 seconds
Attempt 5: wait 8 seconds
```

In production, I also add random jitter and a maximum delay so many workers do not retry simultaneously. I set a maximum attempt count and distinguish transient failures, such as timeouts and HTTP `503`, from permanent failures, such as invalid input.

For example, an invalid email address generally will not become valid just because the worker retries it. That job should be rejected or sent to a dead-letter queue for investigation.

I also ensure retries do not bypass idempotency protections, because a timeout may mean the operation succeeded even though the worker never received the response.

## 6. What is a dead-letter queue (DLQ)?

A dead-letter queue stores messages that cannot be processed successfully after the configured retry policy, or that are explicitly rejected according to the broker's rules.

For example, if a report-generation job repeatedly fails because its input references a missing record, I do not want it retrying indefinitely and consuming worker capacity.

Instead, the message moves to a DLQ, where I can inspect:
- The original message and message ID.
- The number of delivery attempts.
- The failure reason and error details.
- The time of failure and relevant correlation IDs.

I monitor DLQ depth and alert when it increases unexpectedly. After fixing the underlying issue, I can replay or redrive messages, but only after checking that replaying them will not duplicate business effects.

A DLQ is not a substitute for error handling. It is a mechanism for isolating persistent failures and making them observable.

## 7. How do you keep the database and queue consistent? — The outbox pattern

One of the most important problems in asynchronous systems is the **dual-write problem**.

Suppose an API creates an order in the database and then publishes an `OrderCreated` message.

There are two failure cases:

1. The database transaction commits, but publishing the message fails. The order exists, but downstream workers never hear about it.
2. The message is published, but the database transaction fails. Workers receive an event for an order that does not exist.

Publishing the message before committing the database transaction does not solve the problem; it only reverses the possible failure.

The **transactional outbox pattern** solves this by storing the business change and the message-to-be-published in the same database transaction.

For example:

```sql
BEGIN TRANSACTION;

INSERT INTO Orders (OrderId, Status)
VALUES ('ORD-101', 'Pending');

INSERT INTO OutboxMessages
    (MessageId, EventType, Payload, CreatedAt)
VALUES
    ('MSG-101', 'OrderCreated',
     '{"orderId":"ORD-101"}', CURRENT_TIMESTAMP);

COMMIT;
```

Both records commit together. If the transaction rolls back, neither record is saved.

A separate outbox publisher continuously reads unpublished records and sends them to RabbitMQ, Kafka, or SQS. After a successful publish, it marks the record as published. If publishing fails, it retries later.

There is still a possible failure where the broker accepts the message but the publisher crashes before marking the outbox record as published. The publisher may send that message again after restarting. Therefore, the consumer must still be idempotent.

The outbox pattern provides reliable coordination between the database transaction and eventual message publication without requiring a distributed transaction across the database and broker.

## 8. How I would design a reliable background-job system

For a real application, I would combine these mechanisms:

1. **Queue:** Use RabbitMQ or SQS for ordinary background jobs, or Kafka for retained event streams and replay.
2. **Durable messages:** Configure persistence and acknowledgements appropriately for the chosen broker.
3. **Idempotent consumers:** Use unique message IDs and database constraints to prevent duplicate business effects.
4. **Retries:** Retry transient failures using exponential backoff, jitter, and a bounded attempt count.
5. **Dead-letter queue:** Isolate permanently failing messages and alert on them.
6. **Transactional outbox:** Save the business change and its outgoing event in one database transaction.
7. **Monitoring:** Track queue depth, oldest-message age, processing latency, failure rate, retry count, and DLQ depth.
8. **Graceful shutdown:** Stop accepting new work and allow in-flight jobs to finish or become eligible for redelivery.

**My key principle is that a reliable queue system does not assume messages will be delivered exactly once. It assumes failures and duplicate deliveries can happen, then uses durable messages, idempotent processing, controlled retries, and transactional coordination to protect business correctness.**

*Q11. Error handling, logging and observability. How do you design a consistent API error format, global exception handling and validation layers? What makes logs useful (structured logs, correlation/request IDs, no secrets or PII)? Explain the three pillars (logs, metrics, traces), health checks, and how you’d debug a latency spike in production.*

## 1. How do you design a consistent API error format?

I design API errors so that every endpoint returns a predictable structure, regardless of whether the error comes from validation, authentication, business rules, or an unexpected server exception.

For example, instead of returning different formats from different controllers, I use a consistent error response containing a machine-readable error code, a human-readable message, optional validation details, and a request ID for troubleshooting.

A typical response could look like this:

```json
{
  "status": 422,
  "code": "VALIDATION_FAILED",
  "message": "The request contains invalid fields.",
  "errors": [
    {
      "field": "email",
      "code": "INVALID_FORMAT",
      "message": "Enter a valid email address."
    }
  ],
  "requestId": "req-a81f23"
}
```

Here, `code` is intended for application logic, while `message` is intended for users or developers. The frontend can use the error code to display the appropriate message without depending on the exact wording of the API response.

I use appropriate HTTP status codes:
- **400 Bad Request:** The request is malformed or cannot be parsed.
- **401 Unauthorized:** Authentication is missing or invalid.
- **403 Forbidden:** The user is authenticated but lacks permission.
- **404 Not Found:** The requested resource does not exist or is intentionally concealed.
- **409 Conflict:** The request conflicts with the current state, such as creating a duplicate resource.
- **422 Unprocessable Content:** The request is syntactically valid but fails semantic validation or business input rules.
- **429 Too Many Requests:** The client has exceeded a rate limit.
- **500 Internal Server Error:** An unexpected server-side failure occurred.
- **503 Service Unavailable:** The service is temporarily unable to handle the request.

I keep the format consistent across endpoints and avoid returning stack traces, database errors, internal paths, or implementation details to clients.

## 2. How do you implement global exception handling?

I avoid writing repetitive `try-catch` blocks in every controller. Instead, I use centralized exception-handling middleware or a global exception handler.

The general request flow is:

1. The request enters the middleware pipeline.
2. Authentication, authorization, and validation run at their appropriate stages.
3. The controller calls the application or service layer.
4. If an unexpected exception occurs, the global handler catches it, logs the exception, and converts it into a safe HTTP error response.
5. The response includes the appropriate status code and request ID.

For example, in ASP.NET Core, I can register centralized exception handling using `UseExceptionHandler` or implement an `IExceptionHandler`.

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
```

A custom handler can map known application exceptions to appropriate status codes and return a standardized response. Unexpected exceptions should be logged with their stack traces on the server, while clients receive a generic `500` response.

I distinguish expected failures from unexpected failures. For example, a resource-not-found exception should normally become a `404`, while a database connection failure should generally become a `500` or `503`, depending on the failure and service behavior.

I also avoid swallowing exceptions or returning `200 OK` with an error message in the response body. The HTTP status must accurately represent the outcome.

## 3. How do you separate validation layers?

I separate validation according to what is being validated and where that rule belongs.

**Request or DTO validation:** Checks whether the incoming data has the correct structure and format. Examples include required fields, valid email format, string length, and numeric ranges.

**Business validation:** Checks domain rules that require business context. Examples include whether an order can be cancelled, whether an account has sufficient balance, or whether a discount is applicable.

**Database constraints:** Protect data integrity even when multiple requests execute concurrently. Examples include unique constraints, foreign keys, and `NOT NULL` constraints.

For example, checking whether an email field is present belongs in request validation. Checking whether that email is already registered may belong in the application or service layer, but a database unique constraint is still necessary to prevent concurrent requests from creating duplicates.

In ASP.NET Core, I can use Data Annotations, FluentValidation, or endpoint-level validation for request validation. Business rules belong in the application or domain layer, while database constraints enforce integrity at the persistence layer.

I do not rely only on frontend validation because clients can bypass it. Server-side validation is always required.

## 4. What makes logs useful?

Logs are useful when they explain what happened, when it happened, which operation was involved, and how to investigate the failure.

I prefer **structured logging** over unstructured text logs.

For example, instead of:

```text
Order failed for user 123
```

I would log structured fields such as:

```json
{
  "timestamp": "2026-10-14T10:30:00Z",
  "level": "Error",
  "message": "Order creation failed",
  "requestId": "req-a81f23",
  "orderId": "ORD-101",
  "operation": "CreateOrder",
  "errorCode": "PAYMENT_PROVIDER_TIMEOUT"
}
```

Structured logs allow log platforms to filter and aggregate fields without parsing arbitrary message text.

I use appropriate log levels:

- **Trace/Debug:** Detailed diagnostic information, generally disabled or restricted in production.
- **Information:** Important application events, such as a job completing successfully.
- **Warning:** Recoverable or unusual conditions, such as a retry or degraded dependency.
- **Error:** An operation failed and requires investigation.
- **Critical:** A severe failure that threatens the application's ability to operate.

I include correlation or request IDs so that logs from the API, background workers, and downstream services can be related to the same operation. In distributed systems, I propagate trace context as well, rather than relying exclusively on a custom request ID.

I never log passwords, access tokens, refresh tokens, API keys, payment-card details, or unnecessary personally identifiable information. I also sanitize untrusted input to prevent log injection and restrict access to production logs.

## 5. Explain the three pillars of observability

The three pillars are **logs, metrics, and traces**. They complement each other, and none is a complete replacement for the others.

### Logs — What happened?

Logs record individual events and their context.

For example, a payment request failed because the payment provider timed out. The error log may contain the operation name, error category, request ID, and exception details.

Logs are useful for investigating a specific failure.

### Metrics — How often, how much, and how long?

Metrics are numerical measurements collected over time.

Common application metrics include:
- Request rate, usually measured in requests per second.
- Error rate, such as the percentage of HTTP `5xx` responses.
- Latency, including p50, p95, and p99 response times.
- CPU and memory utilization.
- Database connection-pool utilization.
- Queue depth and oldest-message age.
- Dependency failure rates.

For example, average latency might remain low while a small percentage of requests become extremely slow. Monitoring p95 and p99 latency helps identify that problem.

I avoid using high-cardinality labels such as user IDs or request IDs in metric dimensions because they can create excessive time-series data and increase monitoring costs.

### Traces — Where was the time spent?

Distributed traces follow a request across multiple services and dependencies. Each operation is represented by a span, and related spans belong to the same trace.

For example:

```text
HTTP Request                  2,000 ms
  ├── Application logic         50 ms
  ├── Database query          1,700 ms
  └── External API call         200 ms
```

This trace suggests that the database operation is responsible for most of the latency.

Distributed tracing is particularly useful when a request passes through an API, a database, a message broker, and multiple downstream services.

In practice, I would use OpenTelemetry instrumentation and an appropriate backend to collect and analyze traces, metrics, and logs.

## 6. What are health checks?

Health checks indicate whether an application instance and its dependencies are functioning as expected.

I distinguish between two important types.

**Liveness check:** Answers whether the application process is alive and should continue running. It should not fail merely because a temporary dependency is unavailable, or an orchestrator might repeatedly restart otherwise healthy processes.

**Readiness check:** Answers whether the application is ready to receive traffic. It can account for whether essential dependencies, initialization, or required resources are available.

For example, an ASP.NET Core application can expose health endpoints:

```csharp
builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready");
```

This is a minimal example; both endpoints use the default health-check configuration unless additional checks and separate predicates are configured. In production, I configure their behavior deliberately.

A database connectivity check may belong in readiness when the application cannot serve requests without the database. An optional analytics service should not necessarily make the entire application unready.

Health endpoints should expose only necessary operational information and should not reveal secrets or internal infrastructure details to unauthorized callers.

## 7. How would you debug a latency spike in production?

I would follow a systematic process rather than immediately changing code or restarting the application.

**Step 1: Confirm the impact.**

I check dashboards for request rate, error rate, p50/p95/p99 latency, CPU, memory, garbage collection, thread-pool utilization, and dependency health.

I determine when the spike started, which endpoints are affected, whether it affects all users or only a subset, and whether error rates increased at the same time.

**Step 2: Identify the affected component.**

I use distributed traces to identify where requests spend their time. If most latency is inside the application, I investigate application execution and resource contention. If the delay is in a database span, I focus on queries, locks, connection pools, and database load. If an external API is slow, I inspect its response times, timeouts, and failure rates.

**Step 3: Compare against recent changes.**

I check recent deployments, configuration changes, database migrations, traffic increases, and changes in background-job volume. If the spike started immediately after a deployment, I investigate the release and consider rolling it back if the evidence supports that decision.

**Step 4: Investigate the likely bottleneck.**

For a database bottleneck, I examine slow-query logs, execution plans, missing indexes, lock waits, long transactions, and connection-pool exhaustion.

For an application bottleneck, I inspect CPU profiles, memory allocation, garbage collection, thread-pool starvation, synchronous blocking in asynchronous code, and expensive serialization.

For a downstream dependency, I inspect latency, timeouts, retries, rate limits, and circuit-breaker behavior. Excessive retries can amplify a dependency outage and make the latency spike worse.

For a background-processing bottleneck, I check queue depth, oldest-message age, worker concurrency, and processing duration.

**Step 5: Mitigate safely.**

Depending on the evidence, I might roll back a problematic release, scale a saturated component, optimize a slow query, reduce unnecessary work, or apply bounded timeouts and controlled retries.

I avoid blindly increasing timeouts or retries because doing so can increase resource consumption and worsen an overload situation.

**Step 6: Verify and prevent recurrence.**

I confirm that latency, error rates, and resource utilization return to acceptable levels. Then I document the root cause, add missing dashboards or alerts, and implement a regression test or other corrective action.

For example, if p95 latency rises from 200 ms to 2 seconds and traces show that a database query consumes 1.7 seconds, I would inspect the query plan, index usage, lock waits, and database load before deciding whether the problem is query inefficiency or contention.

## 8. How I bring everything together

For a production API, I would implement:

1. A consistent error response format and centralized exception handling.
2. Request validation, business-rule validation, and database integrity constraints at their appropriate layers.
3. Structured logs with request IDs, trace context, and safe diagnostic details.
4. Metrics and alerts for latency, errors, traffic, resource utilization, and dependency failures.
5. Distributed tracing to identify slow operations across service boundaries.
6. Liveness and readiness checks with clearly defined behavior.
7. A production debugging process that starts with measurable evidence and ends with verified remediation.

**My main principle is that error handling tells clients how an operation failed, logging records the details of individual events, and observability helps us understand the behavior of the entire system. A good production system needs all three to diagnose problems quickly without exposing sensitive information.**

*Q12. API security hardening. How do you prevent SQL injection (parameterized queries), mass assignment, IDOR/BOLA and broken access control? Explain rate limiting algorithms (token bucket, sliding window), CORS on the server side, password hashing (bcrypt/argon2, salting), secrets management, and input validation vs output encoding.*

## 1. How do you prevent SQL injection?

SQL injection occurs when an application builds SQL queries by concatenating untrusted user input into a query string. An attacker may manipulate the query to access, modify, or delete data without authorization.

For example, this is unsafe:

```javascript
const query =
  `SELECT * FROM Users WHERE email = '${email}'`;
```

If `email` contains malicious SQL, the resulting query may execute unintended logic.

I prevent SQL injection by using **parameterized queries or prepared statements**. The SQL structure remains separate from the user-supplied values, so those values are treated as data rather than executable SQL.

```javascript
const [users] = await db.execute(
  "SELECT * FROM Users WHERE email = ?",
  [email]
);
```

In .NET, I use parameterized commands through ADO.NET, Dapper, or Entity Framework Core. For example, Dapper supports named parameters:

```csharp
var user = await connection.QuerySingleOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Email = @Email",
    new { Email = email }
);
```

I do not rely on manually escaping strings because it is easy to get wrong. I also remember that parameterization protects values, not arbitrary SQL identifiers. If a user can select a sort column, I validate it against an allowlist instead of concatenating arbitrary column names into the query.

Finally, I apply least-privilege database permissions and avoid exposing detailed SQL errors to clients. Parameterization is the primary defense; restricted database permissions reduce the impact if another vulnerability exists.

## 2. What is mass assignment, and how do you prevent it?

Mass assignment happens when an application automatically binds incoming request fields to a database entity or domain object without controlling which fields the client is allowed to modify.

Suppose an application has this user entity:

```csharp
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Role { get; set; }
    public bool IsAdmin { get; set; }
}
```

If the API directly binds a request to this entity, an attacker might submit:

```json
{
  "name": "Dev",
  "email": "dev@example.com",
  "isAdmin": true,
  "role": "Admin"
}
```

If the application accepts and persists these fields without authorization, the attacker may escalate privileges.

I prevent this by using **dedicated request DTOs or allowlisted fields**. For example:

```csharp
public class UpdateProfileRequest
{
    public string Name { get; set; }
    public string Email { get; set; }
}
```

The application explicitly maps the permitted fields to the entity instead of binding the entire entity from the request.

Sensitive fields such as `Role`, `IsAdmin`, `OwnerId`, and account status should be changed only through dedicated operations with appropriate authorization checks.

The key principle is that a field existing in a model does not mean the client should be allowed to modify it.

## 3. What are IDOR and BOLA?

**IDOR (Insecure Direct Object Reference)** occurs when an application exposes a resource identifier but fails to verify whether the current user is authorized to access the referenced resource.

**BOLA (Broken Object Level Authorization)** is the API security term commonly used for this class of vulnerability.

For example, consider an endpoint:

```http
GET /api/orders/1002
```

A user changes the ID to `/api/orders/1003`. If the server returns another customer's order simply because the record exists, the endpoint has an object-level authorization vulnerability.

Authentication alone does not solve this. The user may have a valid token but still lack permission to access that specific order.

I prevent BOLA by checking authorization against the actual resource and its ownership or access policy.

For example, the database query can be scoped to the authenticated user's identity:

```sql
SELECT *
FROM Orders
WHERE OrderId = @OrderId
  AND CustomerId = @CurrentCustomerId;
```

Here, `@CurrentCustomerId` must come from the authenticated server-side identity, not from a customer ID supplied by the client.

For more complex systems, I check tenant membership, resource ownership, delegated permissions, and role-based rules. The same authorization must apply to reads, updates, deletes, downloads, and other operations.

I also test with two different users and tenants to verify that one user cannot access another user's resources by changing IDs.

## 4. How do you prevent broken access control?

Broken access control is broader than BOLA. It occurs when an application fails to enforce who can perform an action or access a resource.

Examples include:
- A normal user accessing an admin endpoint.
- A user updating another tenant's records.
- An authenticated user invoking a privileged operation through a different HTTP method.
- A user changing a role or account status they are not authorized to modify.

I use **defense in depth**:

1. Authenticate the user using a trusted identity mechanism.
2. Enforce role or permission checks at the endpoint or policy layer.
3. Perform object-level authorization in the application or service layer.
4. Scope database operations to the appropriate tenant or resource where possible.
5. Deny access by default and explicitly grant permitted actions.
6. Test both allowed and denied cases.

For example, in ASP.NET Core, I can protect an administrative endpoint with a policy:

```csharp
[Authorize(Policy = "ManageUsers")]
[HttpDelete("{id}")]
public async Task<IActionResult> DeleteUser(int id)
{
    // Perform resource-level authorization as needed.
    // Execute the authorized operation.
    return NoContent();
}
```

The policy can verify a role or specific permission. However, the endpoint policy alone is not enough if access also depends on ownership or tenant membership. Those checks must be performed separately.

I do not rely on hiding buttons in the frontend. The server must enforce authorization for every protected operation.

## 5. Explain rate limiting and its algorithms

Rate limiting controls how frequently a client can call an API. It helps protect against brute-force attempts, abusive traffic, resource exhaustion, and accidental overload.

I can apply limits by user, API key, IP address, tenant, or endpoint, depending on the use case. For authenticated APIs, a user or tenant limit is often more meaningful than an IP-only limit because many legitimate users may share an IP address.

### Token bucket

The token bucket algorithm maintains a bucket containing a limited number of tokens. Tokens are replenished at a configured rate, up to a maximum capacity. Each request consumes a token; if no token is available, the request is delayed or rejected.

For example, a bucket might hold 10 tokens and refill at 5 tokens per second. A client can make a short burst of up to 10 requests, but sustained traffic is constrained by the refill rate.

This is useful when an API should permit small bursts while controlling its long-term request rate.

### Sliding window

A sliding-window algorithm tracks requests over a moving time interval. For example, it might allow 100 requests during any rolling 60-second period.

Unlike a fixed-window counter, it reduces the boundary problem where a client sends many requests at the end of one window and many more immediately at the start of the next.

An exact sliding window can require more storage or computation. A sliding-window counter approximates the rolling limit with lower overhead.

### Comparison

| Algorithm | Main advantage | Trade-off |
|---|---|---|
| Token bucket | Allows controlled bursts | Bursts can consume the available capacity quickly |
| Fixed window | Simple and inexpensive | Can allow boundary bursts |
| Sliding window | Better control over rolling request volume | Exact tracking may cost more resources |
| Leaky bucket | Smooths outgoing traffic | May delay or reject bursts |

When a client exceeds a limit, I generally return HTTP `429 Too Many Requests`, optionally including a `Retry-After` header.

For a multi-instance API, the limiter must work across instances if the limit is intended to be global. An in-memory counter on each server can otherwise allow clients to exceed the intended aggregate limit. A shared store such as Redis or a suitable API gateway can help enforce distributed limits.

I also apply stricter limits to sensitive endpoints such as login, password reset, OTP verification, and expensive report generation.

## 6. What is CORS, and how do you configure it securely?

CORS stands for Cross-Origin Resource Sharing. It is a browser-enforced mechanism that controls whether frontend JavaScript from one origin can access responses from another origin.

An origin consists of the scheme, hostname, and port. For example, `https://app.example.com` and `https://api.example.com` are different origins.

The API server returns headers such as:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

For requests that require preflight, the browser first sends an `OPTIONS` request to check whether the actual request is permitted.

I configure CORS on the server to allow only the required frontend origins, methods, and headers. In production, I avoid allowing every origin unnecessarily.

If cookie-based authentication is used, the server must explicitly allow the frontend origin and return:

```http
Access-Control-Allow-Credentials: true
```

The wildcard origin `*` cannot be used with credentialed CORS requests. Cookie settings such as `Secure` and `SameSite` must also be configured appropriately.

**An important distinction:** CORS is not authentication or authorization. Non-browser clients can call an API without obeying browser CORS restrictions. Therefore, CORS does not prevent direct API access or replace server-side access control. For cookie-authenticated applications, CSRF protection may also be necessary.

## 7. How do you store passwords securely?

I never store passwords as plaintext or encrypt them using a reversible encryption scheme just for authentication.

Instead, I use a dedicated password-hashing algorithm such as **Argon2id or bcrypt**. These algorithms are deliberately expensive to compute, making large-scale password guessing more difficult if the password database is compromised.

Password hashing differs from general-purpose hashing. A fast algorithm such as SHA-256 is not appropriate by itself for password storage because attackers can test guesses extremely quickly.

A password-hashing process works as follows:

1. The user creates a password.
2. The server generates a unique random salt.
3. The password-hashing algorithm derives a hash using the password, salt, and configured work factor.
4. The database stores the encoded hash and the parameters needed for verification.
5. During login, the application verifies the submitted password against the stored hash using the library's verification function.

A **salt** ensures that identical passwords do not produce identical stored hashes and prevents attackers from efficiently reusing precomputed hash tables across accounts.

I prefer Argon2id when supported by the platform and operational requirements; bcrypt is also widely used. I configure the work factor based on current guidance and production performance, and I upgrade parameters over time as needed.

I also enforce reasonable password policies, rate-limit login attempts, and use secure password-reset flows. A pepper can provide an additional defense if it is stored separately from the password database, such as in a secrets manager, but it does not replace salting or a strong password hash.

## 8. How do you manage secrets securely?

Secrets include database credentials, API keys, signing keys, cloud credentials, and third-party service tokens.

I follow these practices:

- **Do not hardcode secrets:** Never commit production credentials into source code or configuration files tracked in Git.
- **Use a secrets manager:** Store production secrets in a dedicated system such as AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault.
- **Use environment-specific configuration:** Development, staging, and production should have separate credentials and permissions.
- **Apply least privilege:** A service should receive only the permissions it needs.
- **Rotate credentials:** Establish a rotation process, especially after suspected exposure or personnel changes.
- **Restrict access:** Limit which applications and engineers can read or modify secrets.
- **Prevent accidental exposure:** Redact secrets from logs, error responses, CI/CD output, and monitoring events.

For local .NET development, User Secrets can keep sensitive values out of the project source tree. It is intended for development, not as a production secrets vault.

For production, I prefer managed identity or workload identity where supported, so an application can access cloud resources without embedding long-lived credentials in its configuration.

If a secret is accidentally committed to Git, removing it from the latest commit is not enough. I revoke or rotate it because the credential may remain in repository history or other copies.

## 9. Input validation vs output encoding

These two controls solve different problems.

**Input validation** checks whether incoming data meets the application's expected format and rules. For example, an API may require a valid email address, a positive quantity, a known enum value, or a maximum string length.

Validation reduces invalid data and constrains what the application accepts, but it is not a universal defense against injection attacks. SQL parameterization is still required.

**Output encoding** makes data safe for the context in which it will be rendered or interpreted. For example, a user's name may contain characters that have special meaning in HTML. If the application inserts that value into an HTML page without appropriate encoding, an attacker may exploit cross-site scripting (XSS).

I use context-appropriate encoding and safe rendering APIs. In a web frontend, I prefer `textContent` or framework-provided text interpolation over injecting untrusted strings through `innerHTML`. If HTML is genuinely required, I use an appropriate HTML sanitizer.

Encoding rules differ between HTML text, HTML attributes, JavaScript, CSS, and URLs. Encoding for one context does not automatically make a value safe in another.

I also distinguish output encoding from SQL parameterization: output encoding helps prevent browser-side injection, while parameterized queries prevent SQL injection.

## 10. How I combine these controls in a production API

For a typical REST API, I would apply security at several layers:

1. **Transport:** Use HTTPS and secure configuration.
2. **Authentication:** Verify credentials or tokens correctly.
3. **Authorization:** Enforce endpoint permissions, ownership, and tenant boundaries.
4. **Data access:** Use parameterized queries and database constraints.
5. **Request handling:** Use DTOs, allowlisted fields, and server-side validation.
6. **Abuse prevention:** Apply distributed rate limiting, especially to sensitive endpoints.
7. **Credential security:** Use Argon2id or bcrypt for passwords and a secrets manager for application credentials.
8. **Browser security:** Configure CORS narrowly and use appropriate CSRF protection for cookie-based authentication.
9. **Output safety:** Encode or sanitize untrusted data according to the rendering context.
10. **Monitoring:** Log security-relevant events without recording passwords, tokens, or unnecessary personal data.

**My main principle is that API security cannot depend on a single middleware or library. I combine input controls, secure database access, authentication, resource-level authorization, rate limiting, credential protection, and safe output handling so that failure in one layer does not automatically compromise the entire application.**

*Q13. Networking under the hood. What happens from typing a URL to getting a response (DNS, TCP handshake, TLS handshake, HTTP/1.1 vs HTTP/2 vs HTTP/3)? What do keep-alive, connection reuse and head-of-line blocking mean, and how does a reverse proxy like Nginx sit in that path?*

When I type a URL into a browser, several steps happen before I receive a response from the server.

**1. DNS Resolution**

First, the browser checks its cache to see whether it already knows the IP address for the domain. If not, the operating system and DNS resolver help resolve the domain name, such as `example.com`, into an IP address.

DNS allows us to use human-readable domain names instead of remembering IP addresses.

**2. Establishing a TCP Connection**

Once the IP address is resolved, the browser establishes a TCP connection with the server using a three-way handshake:

- **SYN:** The client requests a connection.
- **SYN-ACK:** The server acknowledges the request.
- **ACK:** The client confirms the connection.

TCP provides reliable, ordered delivery of data.

**3. TLS Handshake**

If the website uses HTTPS, the browser and server perform a TLS handshake to establish secure communication.

During this process, they negotiate encryption parameters, validate the server's certificate, and establish session keys. Afterward, HTTP data can be transmitted securely.

With TLS 1.3, the handshake typically requires one round trip after TCP connection establishment, although session resumption can reduce latency.

**4. Sending the HTTP Request**

Once the connection is ready, the browser sends an HTTP request containing information such as the method, URL path, headers, and possibly a request body.

For example:

```http
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json
```

The server processes the request and returns an HTTP response containing a status code, headers, and response body.

**5. HTTP/1.1 vs HTTP/2 vs HTTP/3**

- **HTTP/1.1:** Supports persistent connections, but requests on a single connection are generally processed in order. This can cause application-level head-of-line blocking when an earlier response delays later responses.
- **HTTP/2:** Uses binary framing and multiplexing, allowing multiple request and response streams over a single TCP connection. However, TCP packet loss can still delay all streams waiting for the missing data.
- **HTTP/3:** Uses QUIC over UDP instead of TCP. QUIC provides encrypted transport and independent streams, so packet loss affecting one stream generally does not block delivery of data on other streams.

**6. Keep-Alive and Connection Reuse**

Keep-alive means keeping a connection open after a request so it can be reused for subsequent requests.

For example, if a frontend makes multiple API calls to the same server, connection reuse can avoid repeatedly performing TCP and TLS handshakes. This reduces latency and connection overhead.

**7. Head-of-Line Blocking**

Head-of-line blocking occurs when one delayed item prevents other items from progressing.

In HTTP/1.1, an earlier response can delay later responses on the same connection. In HTTP/2, TCP packet loss can delay data across multiple streams. HTTP/3 reduces this problem through independent QUIC streams.

**8. Role of Nginx as a Reverse Proxy**

Nginx sits between the client and the backend application. Instead of connecting directly to the application server, the client sends its request to Nginx, which forwards it to the appropriate backend.

For example:

```text
Browser / Frontend
        |
      HTTPS
        |
      Nginx
        |
   Reverse Proxy
        |
   Backend API
   (.NET / Node.js)
        |
      Database
```

Nginx can handle TLS termination, route requests to different services, load-balance traffic across multiple backend instances, serve static files, and apply request limits.

For example, Nginx might route `/api` requests to a .NET API while serving frontend assets directly.

**In my development work,** understanding this flow helps me debug DNS issues, HTTPS certificate errors, CORS problems, connection failures, slow API responses, and reverse-proxy misconfigurations. It also helps me understand why connection reuse and HTTP version selection can affect application performance.

**In short:** DNS resolves the domain, TCP or QUIC establishes transport connectivity, TLS secures communication where applicable, HTTP carries the request and response, and a reverse proxy such as Nginx can manage and forward traffic to backend services.

*Q14. API design: pagination, filtering and bulk operations. Compare offset, keyset (cursor) and seek pagination, and explain why offset degrades on large tables and breaks under concurrent inserts. How do you design stable sorting, partial responses, bulk create/update endpoints with partial failures, and long-running operations (202 Accepted plus a status endpoint)?*

In API design, I focus on making endpoints efficient, predictable, and scalable, especially when dealing with large datasets or operations that process multiple records.

### 1. Offset vs. Keyset vs. Seek Pagination

**Offset pagination** uses `limit` and `offset` to retrieve records.

For example:

```http
GET /api/users?limit=20&offset=40
```

SQL:

```sql
SELECT id, name, email
FROM users
ORDER BY id
LIMIT 20 OFFSET 40;
```

This returns 20 records after skipping the first 40 records.

The main advantage is simplicity, and it supports page-number navigation. However, on large tables, the database may need to scan or traverse many rows before returning the requested page. Performance can degrade as the offset increases.

It can also produce inconsistent results when records are inserted or deleted between requests. For example, if a new record is inserted at the beginning, the next offset-based request may return a duplicate record.

**Keyset pagination** uses the values of the last record from the previous page to fetch the next page.

```http
GET /api/users?limit=20&afterId=120
```

SQL:

```sql
SELECT id, name, email
FROM users
WHERE id > 120
ORDER BY id
LIMIT 20;
```

Instead of skipping records, the database continues from the last-seen key. With an appropriate index, this is generally more efficient for deep pagination and less affected by inserts before the cursor.

**Seek pagination** is commonly used as another name for keyset pagination. More generally, it describes locating the next page by seeking to a position in an ordered dataset rather than skipping a large number of rows.

For production APIs, I prefer keyset or seek pagination for large datasets and infinite scrolling. I use offset pagination when users need simple page-number navigation and the dataset is relatively small or stable.

### 2. Stable Sorting and Cursor Design

Pagination requires deterministic ordering. If multiple records have the same sort value, I include a unique tie-breaker such as the primary key.

For example:

```sql
SELECT id, name, created_at
FROM users
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The cursor must contain both `created_at` and `id`. The next-page query becomes:

```sql
SELECT id, name, created_at
FROM users
WHERE created_at < @lastCreatedAt
   OR (
       created_at = @lastCreatedAt
       AND id < @lastId
   )
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

This prevents records with identical timestamps from being skipped or repeated simply because their sort values are equal.

I also ensure that appropriate indexes support the filter and sort columns. For public APIs, I would usually encode the cursor and may sign it to prevent tampering.

One important point is that cursor pagination does not automatically provide a consistent snapshot. If records are updated or deleted during pagination, results can still change. If a perfectly consistent export is required, I would use a snapshot or a fixed dataset boundary.

### 3. Filtering and Sorting

I design filtering through explicit query parameters rather than allowing clients to construct arbitrary SQL.

For example:

```http
GET /api/users?status=active&role=admin&sortBy=createdAt&sortOrder=desc
```

The backend validates the filter values, applies authorization rules, and uses parameterized SQL queries.

For sorting, I use an allowlist of supported columns because SQL parameters cannot safely substitute arbitrary column names.

I also consider:

- Combining filters with pagination.
- Adding indexes for frequently used filter and sort combinations.
- Defining sensible defaults and maximum page sizes.
- Returning a consistent response format.
- Preventing users from accessing records outside their permissions.

### 4. Partial Responses

Sometimes clients do not need every field of a resource. I can support field selection through a query parameter.

For example:

```http
GET /api/users?fields=id,name,email
```

A response could look like this:

```json
{
  "data": [
    {
      "id": 101,
      "name": "Dev",
      "email": "dev@example.com"
    }
  ]
}
```

This reduces response size and unnecessary data transfer, particularly for mobile applications.

However, I validate requested fields against an allowlist and enforce authorization. Sensitive fields must never become accessible simply because a client requests them.

### 5. Bulk Create and Update Operations

When a client needs to create or update hundreds of records, sending one HTTP request per record introduces unnecessary network overhead. I would consider a bulk endpoint.

For example:

```http
POST /api/users/bulk
Content-Type: application/json
```

```json
{
  "items": [
    {
      "externalId": "U101",
      "name": "Aman",
      "email": "aman@example.com"
    },
    {
      "externalId": "U102",
      "name": "Riya",
      "email": "riya@example.com"
    }
  ]
}
```

For bulk operations, I define the transaction and failure semantics clearly.

- **All-or-nothing:** If one item fails, the entire transaction rolls back. This is useful when all records must remain consistent.
- **Partial success:** Valid items are processed, while invalid items return individual errors. This is useful for imports and large batch updates.

For partial success, I would return a result for each item, preferably correlated using its `externalId` or another client-provided identifier.

Example:

```json
{
  "total": 3,
  "succeeded": 2,
  "failed": 1,
  "results": [
    {
      "externalId": "U101",
      "status": "succeeded",
      "id": 101
    },
    {
      "externalId": "U102",
      "status": "failed",
      "error": {
        "code": "DUPLICATE_EMAIL",
        "message": "Email already exists."
      }
    },
    {
      "externalId": "U103",
      "status": "succeeded",
      "id": 103
    }
  ]
}
```

I would validate individual records, enforce database constraints, limit the maximum batch size, and use transactions where appropriate.

I would also consider idempotency for retryable operations, so a client retry does not accidentally create duplicate records. For bulk updates, I would define whether a missing field means “leave unchanged” and how conflicting concurrent updates are handled.

### 6. Long-Running Operations: 202 Accepted

Some operations, such as importing a large CSV file, generating a report, or processing thousands of records, may take too long to complete within a normal HTTP request.

In such cases, I use asynchronous processing.

The client sends a request:

```http
POST /api/imports
```

The server validates the request, creates a background job, and immediately returns:

```http
HTTP/1.1 202 Accepted
Location: /api/operations/op-789
Retry-After: 3
```

```json
{
  "operationId": "op-789",
  "status": "pending",
  "statusUrl": "/api/operations/op-789"
}
```

The client then polls the status endpoint:

```http
GET /api/operations/op-789
```

While processing, the response might be:

```json
{
  "operationId": "op-789",
  "status": "running",
  "progress": {
    "completed": 400,
    "total": 1000
  }
}
```

Once processing finishes, the status endpoint returns a terminal state, such as `succeeded`, `failed`, or `partially_succeeded`, along with a result or error summary.

On the backend, I would store the job status durably and use a background worker or queue for processing. I would also handle retries, idempotency, authorization on the status endpoint, and cleanup of old job records.

**Important:** `202 Accepted` means the request has been accepted for processing, not that the operation has completed successfully.

### 7. How I Choose the Approach

- **Small datasets and page-number navigation:** Offset pagination.
- **Large datasets and infinite scrolling:** Keyset or seek pagination.
- **Mobile applications and bandwidth-sensitive clients:** Partial responses where useful.
- **Imports with independent record failures:** Bulk endpoints with per-item results.
- **Operations taking several seconds or longer:** `202 Accepted` with a status endpoint and background processing.

Overall, my goal is to design APIs that remain performant as data grows, behave predictably under concurrent changes, and communicate failures clearly to clients.

*Q15. Replication, sharding and scaling the database. How do primary-replica replication, replication lag and read-your-own-writes issues work? When do you scale vertically, add read replicas, partition, or shard, and how do you choose a shard key? What breaks when you shard (joins, transactions, rebalancing)?*

When a database starts handling more traffic and data, I consider different scaling strategies based on the bottleneck. The main approaches are vertical scaling, replication, partitioning, and sharding.

### 1. Primary-Replica Replication

In primary-replica replication, one database server acts as the **primary**, handling writes, while one or more **replicas** maintain copies of the data and can serve read queries.

For example:

```text
             Application
                  |
           Database Layer
             /         \
       Write Query    Read Query
           |              |
        Primary        Replica 1
           |              |
           +---- Replication
```

When the application inserts or updates a record, the primary records the change and replicates it to the replicas.

Replication can be synchronous or asynchronous.

- **Synchronous replication:** The primary waits for the required replica acknowledgements before confirming the write, depending on the configured consistency guarantees. This can improve durability but increase write latency.
- **Asynchronous replication:** The primary confirms the write before replicas necessarily receive it. This usually improves write latency, but replicas may temporarily contain stale data.

Read replicas are useful when an application has many more reads than writes, such as dashboards, reports, and product listings.

### 2. Replication Lag and Read-Your-Own-Writes

Replication lag is the delay between a change being committed on the primary and becoming visible on a replica.

For example:

1. A user updates their profile on the primary database.
2. The API returns a successful response.
3. The frontend immediately fetches the profile from a replica.
4. The replica has not received the update yet, so the API returns the old profile.

This is called a **read-your-own-writes consistency problem**.

To handle it, I can:

- Route reads to the primary immediately after a write.
- Keep reads for a particular user or request on the primary for a short period.
- Use replication-position or consistency-token mechanisms when supported.
- Wait until a replica has applied the relevant change before reading from it.

I would not blindly send every read to a replica, especially for payment status, account balances, or other operations that require immediately consistent data.

Also, replication lag is not always a fixed duration. Under heavy load, it can increase significantly.

### 3. Vertical Scaling

Vertical scaling means increasing the resources of a single database server, such as CPU, RAM, storage capacity, or disk I/O performance.

For example, upgrading a database server from 8 GB RAM to 32 GB RAM may improve query performance and allow more data to remain in memory.

I would consider vertical scaling when:

- The application is growing but still fits comfortably on one database server.
- The main bottleneck is CPU, memory, or disk I/O.
- I want to avoid the complexity of distributed data management.

Its limitation is that hardware has practical limits, and the database can remain a single point of failure unless additional high-availability measures are implemented.

### 4. Adding Read Replicas

Read replicas allow read traffic to be distributed across multiple database servers.

For example, a reporting dashboard can query replicas while write operations continue on the primary.

I would consider replicas when read traffic is the bottleneck and query optimization, proper indexing, and caching are no longer sufficient.

However, replicas do not automatically increase write capacity because writes still go through the primary. They also introduce replication lag and consistency considerations.

### 5. Partitioning vs. Sharding

**Partitioning** divides a large table into smaller logical or physical sections. In many relational databases, partitions remain part of the same database system.

For example, an orders table can be partitioned by month:

```text
Orders
  ├── Orders_2026_01
  ├── Orders_2026_02
  ├── Orders_2026_03
  └── Orders_2026_04
```

When a query filters by order date, the database may scan only the relevant partitions. This is called partition pruning.

Partitioning is useful for large tables, data retention, and queries that naturally filter by the partition key. It does not necessarily distribute workload across multiple independent servers.

**Sharding** distributes data across multiple database instances. Each shard stores only a subset of the overall dataset.

For example:

```text
             Application
                  |
             Shard Router
             /          \
         Shard A       Shard B
       Users 1–500K  Users 500K+
```

Sharding can increase storage capacity and distribute read and write workloads across servers. However, it introduces more application and operational complexity.

### 6. Choosing a Shard Key

A shard key determines which shard stores a particular record. I choose it based on the application's access patterns, data distribution, and scaling requirements.

For example, in a multi-tenant SaaS application, `tenant_id` can be a good shard key because most queries are scoped to a tenant.

Potential benefits include:

- Keeping most tenant-specific queries on one shard.
- Reducing cross-shard queries.
- Distributing tenants across multiple database instances.

However, I must avoid a shard key that creates a hotspot. If one tenant generates most of the traffic, its shard may become overloaded even when other shards are mostly idle.

A good shard key should provide reasonably balanced data and workload distribution, support common query patterns, and remain stable over time.

Possible strategies include:

- **Range-based sharding:** Records are assigned based on key ranges. It supports range queries but can create hotspots.
- **Hash-based sharding:** A hash of the shard key determines placement. It often distributes records more evenly but makes range queries harder.
- **Tenant-based sharding:** Records are grouped by tenant. It is useful for multi-tenant systems but may need special handling for very large tenants.

### 7. What Breaks When You Shard?

Sharding introduces several challenges.

**Cross-shard joins:** If related records live on different shards, a normal database join may no longer be possible as a single local operation. The application may need to fetch and combine data, duplicate selected data, or use a different data model.

**Distributed transactions:** A transaction spanning multiple shards is more complicated than a transaction within one database. It may require distributed transaction coordination, such as two-phase commit, or an application-level consistency strategy such as a saga.

**Unique constraints:** Enforcing a globally unique value, such as an email address, becomes more difficult if the database can only enforce uniqueness within individual shards. A centralized uniqueness mechanism or carefully designed key strategy may be required.

**Rebalancing:** As data grows, some shards may become overloaded. Moving records to new shards requires data migration, routing updates, monitoring, and careful handling of concurrent writes.

**Operational complexity:** Backups, monitoring, schema migrations, failover, and troubleshooting become more complicated because multiple database instances must work together.

For these reasons, I would not introduce sharding until simpler approaches such as indexing, query optimization, caching, vertical scaling, and read replicas are insufficient.

### 8. My Practical Scaling Strategy

For a typical application, I would approach database scaling in this order:

1. **Optimize first:** Inspect slow queries, add appropriate indexes, avoid unnecessary data retrieval, and examine execution plans.
2. **Scale vertically:** Increase database resources when a single server is reaching its capacity.
3. **Add read replicas:** Distribute read-heavy workloads, while accounting for replication lag.
4. **Partition large tables:** Use a suitable partition key when it improves query performance and data lifecycle management.
5. **Shard when necessary:** Distribute data and workload across database instances when a single primary can no longer meet the application's requirements.

I would make these decisions using metrics such as query latency, CPU and memory utilization, disk I/O, connection saturation, replication lag, and write throughput.

**In short:** Replication improves read scalability and availability when correctly configured, partitioning helps manage large tables, and sharding distributes data across database servers. The right strategy depends on the actual bottleneck, consistency requirements, and operational complexity.

*Q16. CAP, consistency models and SQL vs NoSQL. Explain CAP and PACELC in practical terms, plus strong vs eventual consistency. How do you choose between PostgreSQL/MySQL, MongoDB, Redis and a wide-column store for a given workload, and what do you give up with each?*

When choosing a database, I consider the data structure, query patterns, consistency requirements, transaction needs, scalability, and operational complexity. There is no single database that is best for every workload.

### 1. CAP Theorem

CAP stands for **Consistency, Availability, and Partition Tolerance**. It describes the trade-offs in a distributed data system when a network partition occurs.

- **Consistency (C):** Every successful read returns the latest committed value, or an error if the system cannot guarantee that result.
- **Availability (A):** Every request to a non-failing node receives a response, although the response might not contain the latest data.
- **Partition Tolerance (P):** The system continues operating despite communication failures between nodes.

The important point is that when a network partition occurs, a distributed system cannot guarantee both strong consistency and availability for every request.

For example, suppose two database nodes lose communication with each other. If both accept writes independently, they may return conflicting values. If the system prevents one side from accepting writes, some requests may fail or be delayed, but it can preserve consistency.

In practical terms:

- A **CP-oriented approach** prioritizes consistency during a partition and may reject or delay some requests.
- An **AP-oriented approach** continues serving requests on reachable nodes but may return stale data or require conflict resolution later.

CAP does not mean that a database permanently chooses only two properties under all conditions. The trade-off becomes relevant when a partition occurs, and real systems make decisions at different levels.

### 2. PACELC Theorem

PACELC extends CAP by considering normal operating conditions as well as network partitions.

It stands for:

- **P:** If there is a network partition,
- **A/C:** The system trades off availability and consistency.
- **E:** Else, when there is no partition,
- **L/C:** The system trades off latency and consistency.

For example, a distributed database might replicate data across regions. Waiting for remote replicas to acknowledge a write can improve consistency guarantees, but it increases latency. A system that responds after a local write may be faster but could expose stale data elsewhere.

Therefore, even when the network is healthy, distributed systems must balance latency and consistency.

PACELC is useful when designing multi-region applications because it highlights that the trade-off does not disappear when network failures are absent.

### 3. Strong Consistency vs. Eventual Consistency

**Strong consistency** means that after a successful write, subsequent reads under the specified consistency guarantees observe that write, subject to the system's consistency model and transaction boundaries.

For example, after updating a customer's account balance, the next read should reflect the committed update.

I prefer strong consistency for:

- Financial transactions.
- Inventory reservations where overselling must be prevented.
- Permission changes that must take effect immediately.
- Operations requiring strict coordination between related records.

**Eventual consistency** means that replicas may temporarily return different values, but if updates stop and replication proceeds successfully, the replicas eventually converge.

For example, a user's profile picture might take a short time to appear consistently across all regions.

I would consider eventual consistency for:

- Activity feeds.
- Analytics dashboards.
- Some recommendation systems.
- Replicated content where a short delay is acceptable.

The trade-off is that eventual consistency can improve availability and reduce latency, but the application must tolerate stale reads and sometimes handle conflicting updates.

Strong and eventual consistency are not the only options. Systems can also offer session guarantees, read-your-own-writes, causal consistency, or configurable consistency levels.

### 4. PostgreSQL vs. MySQL

Both PostgreSQL and MySQL are relational databases that support SQL, indexes, constraints, and transactions.

**PostgreSQL** is a strong choice when I need sophisticated SQL queries, complex joins, advanced indexing, robust data integrity, or features such as JSONB and specialized indexes.

**MySQL** is a strong choice for conventional transactional applications, web backends, and systems where its ecosystem, existing infrastructure, and operational familiarity fit the requirements.

For example, in a restaurant ERP system, I might store orders, payments, users, inventory, and menu items in PostgreSQL or MySQL. Foreign keys, unique constraints, and transactions help preserve relationships and business rules.

What I give up with either database is the simplicity of a schema-free document model. I also need to manage schema migrations, indexes, query performance, and the capacity of the database infrastructure as the application grows.

Neither database is automatically more scalable or more reliable in every situation. Schema design, workload, configuration, and operational practices matter significantly.

### 5. MongoDB

MongoDB is a document-oriented NoSQL database. It stores data as BSON documents, which can contain nested objects and arrays.

For example, a product document could contain its details, attributes, and embedded variants.

I would consider MongoDB when:

- The data is naturally document-oriented.
- Records have varying attributes.
- The application frequently reads or updates related information within a document.
- The development model benefits from flexible document structures.

MongoDB supports indexes, replication, aggregation, and multi-document transactions, so it is not limited to simple key-value operations.

However, flexible schemas do not eliminate the need for data validation. Complex relationships and reporting across many entities can require more involved aggregation or application-side processing than in a relational database. Distributed transactions and cross-document operations can also have additional costs.

I would choose MongoDB based on the access patterns and document model, not simply because the data structure may change.

### 6. Redis

Redis is an in-memory data store that supports data structures such as strings, hashes, lists, sets, sorted sets, and streams.

I would primarily use Redis for:

- Caching frequently accessed data.
- Session storage.
- Rate limiting.
- Short-lived tokens and counters.
- Leaderboards and selected real-time workloads.

For example, a backend might cache a frequently requested menu response in Redis to reduce database load and API latency.

The main trade-off is that memory is more expensive than disk-based storage, and the cache introduces invalidation and expiration problems. Redis also supports persistence and replication, but its durability and consistency characteristics depend on the deployment and configuration.

I would not automatically use Redis as the primary source of truth for complex transactional data when a relational database is a better fit.

### 7. Wide-Column Stores

Wide-column databases, such as Apache Cassandra, are designed for distributed workloads involving large amounts of data and high write throughput.

They organize data around partition keys and clustering columns, with data modeling strongly influenced by the queries the application needs to execute.

I would consider a wide-column store when the system needs to handle large volumes of distributed writes, predictable key-based access, and horizontal scaling across many nodes.

Examples include time-series event ingestion, telemetry, and high-volume activity records.

However, I must design tables around known query patterns. Arbitrary joins, flexible relational queries, and conventional relational constraints are generally not the main strengths of these systems. Data duplication is often intentional, and consistency guarantees depend on the database and its configuration.

For example, Cassandra can provide tunable consistency levels, but a distributed relational database and a wide-column store do not have identical transaction or query capabilities.

### 8. How I Choose the Database

| Database | Best fit | Main trade-off |
|---|---|---|
| PostgreSQL | Complex SQL, relational data, advanced queries | Requires relational schema design and operational tuning |
| MySQL | Transactional web applications and conventional SQL workloads | Some advanced query and data-modeling capabilities differ from PostgreSQL |
| MongoDB | Document-oriented data and flexible record structures | Joins, cross-document relationships, and complex reporting may be less convenient |
| Redis | Caching, sessions, counters, and low-latency access | Memory cost, cache invalidation, and deployment-dependent durability |
| Cassandra | High-volume distributed writes and query-specific access patterns | Limited ad hoc querying, joins, and relational constraints |

### 9. My Practical Approach as a Developer

I would first identify the application's source of truth and its most important queries.

For a typical business application, I would start with PostgreSQL or MySQL because relational transactions and constraints are valuable for users, orders, payments, and inventory.

If read traffic becomes high, I would evaluate caching and read replicas. If the application needs flexible document storage, I would consider MongoDB for that particular workload. If low-latency caching is needed, I would add Redis. If the system needs massive distributed write throughput with predictable access patterns, I would evaluate a wide-column database.

I would avoid introducing multiple databases without a clear reason because every additional technology increases monitoring, deployment, backup, security, and troubleshooting complexity.

**In short:** CAP helps explain the consistency-versus-availability trade-off during network partitions, while PACELC also considers latency versus consistency during normal operation. Database selection should follow the workload, consistency requirements, query patterns, and operational constraints rather than treating SQL or NoSQL as universally better.

*Q17. Distributed transactions and the saga pattern. Why can’t you rely on a single ACID transaction across services? Compare two-phase commit with choreographed and orchestrated sagas, compensating actions, and how you handle partial failure, timeouts and duplicate events.*

In a monolithic application, multiple database operations can often be handled within a single ACID transaction. However, in a microservices architecture, different services usually own different databases. This makes it difficult to maintain one atomic transaction across the entire business workflow.

### 1. Why Can't We Rely on a Single ACID Transaction Across Services?

ACID stands for Atomicity, Consistency, Isolation, and Durability.

Suppose an e-commerce application has three services:

- Order Service — manages orders.
- Payment Service — processes payments.
- Inventory Service — manages stock.

When a customer places an order, all three services must coordinate. The order must be created, payment must be processed, and inventory must be reserved.

If each service owns its own database, a normal database transaction cannot atomically commit changes across all three databases.

For example, payment might succeed, but the Inventory Service might fail before reserving the product. The system is now in a partially completed state.

Distributed transactions are possible using coordination protocols, but they introduce complexity, latency, and availability trade-offs. External systems such as payment gateways also cannot generally participate in our database transaction.

For this reason, distributed applications often use the **Saga pattern** to coordinate a business workflow through multiple local transactions.

### 2. What Is the Saga Pattern?

A saga is a sequence of local transactions executed across multiple services. Each service commits its own transaction, and the overall workflow progresses through messages or commands.

If a later step fails, the system can execute compensating actions to undo or counteract the business effects of earlier steps.

For example:

```text
Customer places order
        |
        v
Create pending order
        |
        v
Reserve inventory
        |
        v
Process payment
        |
        v
Confirm order
```

If payment fails after inventory has been reserved, the system can release the reserved inventory and mark the order as failed.

The key point is that a saga does not provide one global ACID transaction. Instead, it manages consistency across services through local transactions, coordination, retries, and compensation.

### 3. Two-Phase Commit (2PC) vs. Saga

**Two-Phase Commit (2PC)** is a distributed transaction protocol that coordinates participating databases through a transaction coordinator.

It has two phases:

**Phase 1 — Prepare**

The coordinator asks all participants whether they can commit. Each participant prepares its transaction and promises to commit if instructed, while retaining the necessary transactional resources.

**Phase 2 — Commit or Abort**

If all participants successfully prepare, the coordinator instructs them to commit. If a participant fails to prepare, the coordinator instructs them to abort.

Advantages of 2PC:
- Provides atomic commit across participating resource managers when the protocol operates correctly.
- Prevents a transaction from committing at only some participants under normal protocol execution.

Disadvantages:
- Participants may hold locks and other resources while waiting for a decision.
- The coordinator can become a bottleneck.
- Failures can leave participants waiting for a decision, reducing availability.
- All participants must support the necessary transaction protocol.

**The Saga pattern**, in contrast, commits local transactions independently and uses forward actions and compensations to manage failures.

Advantages of sagas:
- Services remain loosely coupled.
- Each service controls its own database transaction.
- Workflows can support long-running business processes.
- Failures can be handled through retries and compensations.

Disadvantages:
- Intermediate states may be visible.
- Compensation is not always a perfect reversal.
- The application must handle retries, duplicate messages, and inconsistent intermediate states.

For most independently deployed business microservices, I would consider a saga before introducing distributed 2PC, especially when workflows include external APIs or long-running operations.

### 4. Choreographed vs. Orchestrated Sagas

There are two common ways to coordinate a saga.

**Choreographed saga**

Each service publishes an event after completing its local transaction. Other services listen for those events and execute their own steps.

Example:

```text
Order Service
     |
     | OrderCreated
     v
Inventory Service
     |
     | InventoryReserved
     v
Payment Service
     |
     | PaymentSucceeded
     v
Order Service
     |
     v
Order Confirmed
```

If payment fails, the Payment Service publishes a failure event. The relevant services react by releasing inventory and updating the order status.

I would use choreography for relatively simple workflows with a small number of services and clear event-driven responsibilities.

The drawback is that business logic becomes distributed across event handlers. As the workflow grows, understanding the overall process, handling cycles, and debugging failures become harder.

**Orchestrated saga**

A dedicated orchestrator controls the workflow. It sends commands to services, receives their results, and decides the next step.

Example:

```text
       Order Orchestrator
          /    |     \
         v     v      v
      Order  Inventory Payment
      Service Service  Service
         \      |      /
          \     |     /
          Workflow Status
```

The orchestrator might execute these steps:

1. Create a pending order.
2. Reserve inventory.
3. Charge the customer.
4. Confirm the order.
5. If payment fails, release inventory and mark the order as failed.

I prefer orchestration when the workflow has several steps, conditional paths, timeouts, retries, or compensating actions because the business process is easier to observe and manage in one place.

The trade-off is that the orchestrator introduces another component that must be deployed, monitored, and maintained.

### 5. What Are Compensating Actions?

A compensating action is a business operation that counteracts a previously completed operation.

For example:

| Forward action | Compensating action |
|---|---|
| Reserve inventory | Release inventory |
| Authorize payment | Void authorization |
| Capture payment | Refund payment, where supported |
| Create pending order | Cancel the order |

Compensation is not the same as rolling back a database transaction. The original transaction has already committed, and its effects may already be visible to other systems.

For example, a captured payment cannot simply be erased. The system must issue a refund, which is a new business transaction and may itself fail or take time.

Therefore, I design compensation as an explicit, retryable operation with its own status and error handling.

### 6. How Do You Handle Partial Failures and Timeouts?

In a distributed system, a timeout does not necessarily mean an operation failed. The service might have completed the operation, but its response was lost.

For example, a payment request times out. If I immediately retry it with a new request identifier, the customer might be charged twice.

I would handle this as follows:

**Step 1: Use explicit workflow states.**

For example:

```text
PENDING
  |
  v
INVENTORY_RESERVED
  |
  v
PAYMENT_PROCESSING
  |
  +----> CONFIRMED
  |
  +----> COMPENSATING
             |
             v
          CANCELLED
```

I would persist these states so the workflow can resume after a process restart.

**Step 2: Apply timeouts and bounded retries.**

I would set reasonable timeouts, retry transient failures with exponential backoff and jitter, and avoid retrying permanent business errors such as insufficient funds.

**Step 3: Distinguish unknown outcomes from confirmed failures.**

If a payment request times out, I would query the payment provider or retry using the same idempotency key rather than assuming the payment did not happen.

**Step 4: Compensate when necessary.**

If the payment is definitively unsuccessful, the orchestrator can release inventory and cancel the order. If the payment outcome is unknown, it should reconcile the outcome before taking an action that could cause an incorrect refund or duplicate charge.

**Step 5: Make recovery observable.**

I would track the workflow ID, current step, retry count, last error, and compensation status. Failed workflows should be visible to monitoring and support teams.

### 7. How Do You Handle Duplicate Events?

Message brokers and distributed consumers may deliver the same event more than once. A consumer may process an event successfully and crash before acknowledging it, causing the event to be delivered again.

I design consumers to be **idempotent**, meaning that processing the same logical operation repeatedly does not produce duplicate business effects.

For example, I can maintain a table of processed event IDs:

```sql
CREATE TABLE ProcessedEvents (
    EventId VARCHAR(100) PRIMARY KEY,
    ProcessedAt TIMESTAMP NOT NULL
);
```

The consumer records the event ID and applies the business update within the same local database transaction. A duplicate event then becomes a no-op.

For payments and other external side effects, I also use provider-supported idempotency keys. Database deduplication alone cannot guarantee that an external API call will happen only once.

### 8. The Transactional Outbox Pattern

One common problem is updating a database and publishing an event as two separate operations.

For example, the Order Service commits an order but crashes before publishing `OrderCreated`. The order exists, but downstream services never receive the event.

I would use the **Transactional Outbox pattern**:

1. Save the order and an outbox event in the same database transaction.
2. A background publisher reads unpublished outbox records.
3. The publisher sends the events to the message broker.
4. Published records are marked as processed.

If the publisher crashes after sending an event but before marking it as published, the event may be sent again. Therefore, consumers must still handle duplicates.

The outbox pattern prevents the database update and event publication from silently diverging, provided the outbox publisher and recovery process are implemented correctly.

### 9. My Practical Approach

For a typical order-processing workflow, I would use an orchestrated saga if multiple services need conditional steps and compensation.

I would combine it with:

- Local ACID transactions within each service.
- Durable workflow state.
- Idempotency keys for external side effects.
- Transactional outbox and idempotent consumers.
- Bounded retries, timeouts, and exponential backoff.
- Compensating actions for completed steps.
- Monitoring, reconciliation, and manual recovery for exceptional cases.

**In short:** Two-phase commit coordinates atomic commits across participating resource managers, but can reduce availability and hold resources during failures. Sagas coordinate independent local transactions and recover through compensating business actions. Reliable implementations require explicit workflow states, idempotency, durable messaging, and a plan for uncertain outcomes.

*Q18. Monolith vs microservices. What problems do microservices solve, and what new ones do they create (network failure, versioning, data ownership, debugging)? What is a modular monolith, when would you split a service, and how do circuit breakers, timeouts, bulkheads and retries with jitter prevent cascading failures?*

When designing a backend architecture, I choose between a monolith and microservices based on the application's complexity, team size, deployment requirements, and scalability needs. Microservices are not automatically better; they solve some architectural problems while introducing distributed-system complexity.

### 1. Monolith vs. Microservices

**Monolithic architecture** means the application's functionality is deployed as one application, usually with a single deployment unit.

For example, an e-commerce backend might contain authentication, orders, payments, and inventory in one .NET application.

```text
           Client / Frontend
                   |
             Backend API
                   |
      +------------+------------+
      |            |            |
    Users        Orders       Payments
      |            |            |
      +------------+------------+
                   |
                Database
```

The modules can still be logically separated, but they share the application's runtime and deployment process.

Advantages:
- Simpler development and deployment.
- Easier local debugging and testing.
- In-process function calls are faster than network calls.
- Database transactions across modules are easier when they share a database.

Disadvantages:
- The entire application may need to be redeployed for a small change.
- Different modules cannot be independently scaled as easily.
- Poorly structured code can create tight coupling.
- As the codebase grows, build times and coordination between teams can increase.

**Microservices architecture** divides the application into independently deployable services, each responsible for a specific business capability.

```text
              API Gateway
                   |
        +----------+----------+
        |          |          |
     User API   Order API  Payment API
        |          |          |
      User DB    Order DB   Payment DB
```

Advantages:
- Services can be deployed and scaled independently.
- Teams can work on separate services with clearer ownership.
- A failure in one service can potentially be isolated from others.
- Different services can use technologies suited to their workloads.

However, microservices introduce network latency, partial failures, distributed tracing, service discovery, API versioning, and data consistency challenges.

**My approach:** I would generally start with a well-structured monolith unless independent deployment, scaling, team ownership, or fault isolation provides a clear business benefit.

### 2. What Is a Modular Monolith?

A modular monolith is a single deployable application divided into well-defined modules with explicit responsibilities and controlled dependencies.

For example, a restaurant ERP backend could contain:

- Identity and access management.
- Menu and inventory.
- Orders and billing.
- Payments.
- Reporting.

These modules can share the same application process and database infrastructure, but each module owns its business logic and exposes a controlled interface.

```text
             Single .NET Application
     +----------------------------------+
     | Identity | Menu | Orders | Billing|
     |    Well-defined module boundaries |
     +----------------------------------+
                       |
                    Database
```

I would enforce boundaries so the Orders module cannot arbitrarily modify the internal data structures of the Inventory module.

A modular monolith provides many benefits of good service boundaries without requiring network communication between modules.

It is a good choice when the team is small, deployment requirements are straightforward, and the business domain is still evolving.

### 3. When Would I Split a Service?

I would split a module into a separate service when there is a measurable reason, not simply because the codebase has become large.

Typical reasons include:

1. **Independent scaling:** A reporting workload consumes significant resources and affects transaction processing.
2. **Independent deployment:** A module changes frequently and needs its own release cycle.
3. **Team ownership:** A dedicated team needs autonomy over a well-defined business capability.
4. **Fault isolation:** A failure in a noncritical workload must not take down critical operations.
5. **Different operational requirements:** A component needs a different runtime, security boundary, or availability target.

For example, in a restaurant ERP, order processing and billing might initially live in one application. If reporting queries become resource-intensive, I might first optimize the queries, use a reporting replica, or move reporting into an asynchronous pipeline. Only if the operational benefits justify it would I extract a separate reporting service.

Before splitting, I would check whether the module has clear ownership, limited dependencies, and a manageable data boundary.

### 4. Network Failures, Versioning, and Data Ownership

Microservices communicate over a network, and network calls can fail in ways that ordinary in-process calls do not.

**Network failures:** A service may be unavailable, slow, or unreachable. A timeout can also occur after the remote service has completed its operation but before the response reaches the caller.

I handle this using timeouts, idempotency, appropriate retries, and explicit error handling.

**API versioning:** Services may be deployed at different times, so a consumer might temporarily communicate with an older or newer provider version.

I prefer backward-compatible changes, such as adding optional fields rather than renaming or removing existing fields. Breaking changes may require a new API version and a controlled migration period.

**Data ownership:** Each service should own its data and expose operations through its API or events instead of allowing other services to modify its tables directly.

For example, the Order Service should request an inventory reservation rather than directly updating the Inventory Service's database.

This reduces coupling, but it means cross-service workflows need explicit coordination and may require eventual consistency.

**Debugging:** A single request can pass through several services. I use correlation IDs, structured logs, metrics, and distributed tracing to identify where latency or failure occurs.

### 5. Circuit Breaker Pattern

A circuit breaker prevents an application from repeatedly calling a dependency that is failing or responding too slowly.

It has three main states:

- **Closed:** Requests are allowed through, and failures are monitored.
- **Open:** Requests are rejected immediately or handled through a fallback, avoiding calls to the unhealthy dependency.
- **Half-open:** A limited number of test requests are allowed to determine whether the dependency has recovered.

For example, if a payment provider is consistently timing out, the circuit breaker can temporarily stop new calls instead of allowing every request to wait for the same failing dependency.

After a recovery interval, the breaker allows test calls. If they succeed, it closes; if they fail, it opens again.

The circuit breaker does not fix the dependency. It protects the caller and gives the failing service time to recover.

### 6. Timeouts

Every remote call should have an appropriate timeout.

Without a timeout, a request might wait indefinitely for a slow service, tying up threads, connections, or other resources.

For example, an Order Service calling an Inventory Service might have a short deadline appropriate to the user-facing request.

I would also propagate an overall request deadline through downstream calls so each service does not consume the entire available time independently.

Timeouts should be based on measured latency and the application's latency budget. A timeout is not proof that the remote operation failed, so writes must be designed to handle uncertain outcomes.

### 7. Bulkheads

Bulkheads isolate resources so that one failing dependency cannot consume all the resources required by unrelated operations.

The concept comes from ship compartments: damage in one section should not sink the entire vessel.

For example, I might maintain separate concurrency limits for calls to a payment provider and a reporting service. If reporting becomes slow, it should not consume every available connection or worker needed for payment processing.

Bulkheads can be implemented using bounded worker pools, semaphores, connection pools, or separate execution resources.

The trade-off is that limits must be configured carefully. Too few resources reduce throughput, while too many can overwhelm downstream systems.

### 8. Retries with Exponential Backoff and Jitter

Retries are useful for transient failures, such as temporary network errors or rate limits. However, immediate retries can make an overloaded service even less healthy.

With exponential backoff, the waiting time increases after each failed attempt.

For example:

```text
Attempt 1: Immediate
Attempt 2: Wait approximately 200 ms
Attempt 3: Wait approximately 400 ms
Attempt 4: Wait approximately 800 ms
```

These are illustrative delays, not fixed production settings.

**Jitter** adds randomness to the delay so many clients do not retry at exactly the same time. This reduces synchronized retry spikes, often called a thundering herd.

I also:
- Set a maximum retry count and overall deadline.
- Retry only appropriate transient failures.
- Respect `Retry-After` when applicable.
- Use idempotency keys for retryable write operations.
- Avoid retrying at multiple layers without coordination, which can multiply the number of downstream requests.

Retries should not be used blindly for validation errors, authorization failures, or other permanent failures.

### 9. How These Resilience Patterns Work Together

Consider an Order Service that depends on a Payment Service.

```text
          Order Service
                |
          Request Timeout
                |
          Circuit Breaker
                |
        Bounded Concurrency
                |
          Payment Service
```

The timeout limits how long the caller waits. The circuit breaker stops repeated calls when the dependency is unhealthy. The bulkhead limits how many requests can occupy resources. Controlled retries with backoff and jitter handle suitable transient failures.

Together, these patterns reduce the risk that one slow service causes a cascading failure across the system.

However, they must be configured together. For example, excessive retries can exhaust a bulkhead, and a circuit breaker that opens too aggressively can reject healthy requests.

### 10. My Practical Decision

For a typical business application, I would start with a modular monolith, enforce clear module boundaries, and measure performance and deployment bottlenecks.

If a capability later needs independent scaling, deployment, team ownership, or stronger fault isolation, I would extract it into a service with a clear API and explicit data ownership.

For remote calls, I would establish sensible timeouts, use bounded retries with exponential backoff and jitter, apply circuit breakers where appropriate, and isolate critical resources with bulkheads. I would also instrument the system with metrics, structured logs, and distributed tracing.

**In short:** A monolith minimizes distributed-system complexity, while microservices enable independent deployment, scaling, and ownership. A modular monolith is often a practical starting point, and resilience patterns help contain failures once communication crosses service boundaries.

*Q19. Load balancing, statelessness and horizontal scaling. What does a stateless service mean, and where does session or upload state go instead? Compare round-robin, least-connections and consistent hashing, sticky sessions, health checks, graceful shutdown, and zero-downtime deployments (rolling, blue-green, canary).*

When an application receives more traffic, I need to ensure that requests are distributed across multiple application instances without losing user state or interrupting service during deployments.

### 1. What Is a Stateless Service?

A stateless service does not depend on data stored exclusively in its own local memory to process the next request. Each request contains the necessary information, or the service retrieves it from a shared or external system.

For example, suppose I deploy a Node.js API on three servers behind a load balancer.

```text
             Client
                |
          Load Balancer
          /      |      \
       API 1   API 2   API 3
          \      |      /
           Shared Database
```

If a user logs in through API 1 and the next request reaches API 3, the request should still work.

This is easier when authentication uses a validated token or when session information is stored in a shared session store such as Redis.

A stateful implementation that stores the only copy of a user's session in API 1's memory may fail when the next request reaches API 3.

**Benefits of stateless services:**
- Instances can be added or removed easily.
- Requests can be distributed across servers.
- Instance failures are easier to recover from.
- Deployments and autoscaling become simpler.

### 2. Where Does Session and Upload State Go?

Statelessness does not mean that the application has no state. It means that important state is not tied exclusively to an individual application instance.

**Session state:** I can store sessions in Redis or a shared database. The client carries a session identifier in a secure cookie, and any API instance can use that identifier to retrieve the session.

Alternatively, I can use signed access tokens, such as JWTs, for authentication. A JWT can carry claims that the server validates without querying a session store on every request, although revocation and authorization changes still need appropriate handling.

**File uploads:** I would generally store uploaded files in object storage, such as Amazon S3 or another compatible storage service, rather than on an individual API server's local disk.

For example:

```text
Client
   |
   v
API Server -----> Database
   |
   v
Object Storage
(Uploaded Files)
```

For large files, the backend can issue a time-limited presigned upload URL so the client uploads directly to object storage. The database stores the file metadata and object key.

For temporary upload workflows, I would store upload metadata, progress, and ownership information in a shared database or cache.

This approach allows any API instance to handle the next request without depending on files or session data available only on another server.

### 3. What Is a Load Balancer?

A load balancer distributes incoming requests across multiple application instances.

For example, Nginx can act as a reverse proxy and load balancer in front of several .NET or Node.js API servers.

```text
               Users
                 |
               HTTPS
                 |
              Nginx
             /  |  \
            /   |   \
        API 1 API 2 API 3
            \   |   /
             Database
```

The load balancer improves traffic distribution and can stop routing requests to instances that fail health checks.

However, it does not automatically make an application highly available. The application instances, load balancer, database, network, and other dependencies must also be designed to avoid single points of failure.

### 4. Round-Robin vs. Least-Connections vs. Consistent Hashing

**Round-robin**

The load balancer sends requests to each server in sequence.

For example:

```text
Request 1 → API 1
Request 2 → API 2
Request 3 → API 3
Request 4 → API 1
```

It is simple and works well when servers have similar capacity and requests have similar processing costs.

The limitation is that it does not account for how busy a server is. A server handling several long-running requests may receive another request even when another instance is idle.

**Least-connections**

The load balancer routes new requests to the server with the fewest active connections.

This can work better when requests have different durations or connections remain open for a long time.

However, connection count is only an approximation of workload. One expensive request can consume more CPU than many lightweight requests, and long-lived connections do not always represent active processing.

**Consistent hashing**

Consistent hashing maps a key, such as a user ID or cache key, to a particular server or node.

For example, requests with the same key are routed to the same node while the cluster remains stable.

It is useful for cache locality and distributing keyed workloads. When nodes are added or removed, consistent hashing can limit how many keys need to be reassigned.

However, it can still create hotspots if some keys receive much more traffic than others. It is not a general replacement for ordinary load balancing, and it does not guarantee that a particular user's request will always succeed on one server.

### 5. What Are Sticky Sessions?

Sticky sessions, also called session affinity, attempt to route a particular client back to the same application instance.

For example, a user who initially reaches API 1 continues to be routed to API 1 for subsequent requests.

This can be useful when an older application stores session state in local memory, but it introduces several problems:

- Traffic can become unevenly distributed.
- A server failure can disrupt the user's session.
- Scaling or replacing instances becomes more complicated.
- Session state remains coupled to the instance unless separately persisted.

I prefer shared session storage or properly designed token-based authentication over sticky sessions whenever practical.

Sticky sessions can still be useful for specific protocols or legacy applications, but they should not be the primary solution to state management.

### 6. Health Checks

Health checks allow the load balancer or orchestration platform to determine whether an instance should receive traffic.

I distinguish between two common checks:

- **Liveness check:** Determines whether the process is functioning or needs to be restarted.
- **Readiness check:** Determines whether the instance is ready to serve requests.

For example, a newly started API may be alive but not ready because it has not loaded required configuration or initialized its dependencies.

I would avoid making a liveness check depend on every downstream service, because a temporary database outage could otherwise trigger unnecessary restarts across all API instances.

Readiness checks can account for dependencies that are genuinely required to serve traffic. Checks should also use suitable timeouts, thresholds, and intervals to avoid reacting to temporary fluctuations.

### 7. Graceful Shutdown

Graceful shutdown means allowing an application instance to stop receiving new requests while giving existing requests a reasonable opportunity to finish.

A typical sequence is:

1. Mark the instance as unready.
2. Stop routing new requests to it.
3. Allow in-flight requests to complete within a configured grace period.
4. Close connections and background workers cleanly.
5. Exit the process.

For example, during a .NET API deployment, graceful shutdown helps prevent requests from being interrupted simply because an old instance is being replaced.

If a request cannot finish before the deadline, the application still needs appropriate cancellation and retry behavior. Graceful shutdown reduces disruption but cannot guarantee that every request will finish successfully.

### 8. Zero-Downtime Deployment Strategies

The goal of zero-downtime deployment is to release a new application version without making the service unavailable to users. In practice, it requires compatible application behavior, healthy capacity, and careful handling of in-flight requests.

**Rolling deployment**

Instances are updated gradually.

```text
Before:  V1  V1  V1
Step 1:  V2  V1  V1
Step 2:  V2  V2  V1
After:   V2  V2  V2
```

Advantages:
- Uses existing infrastructure efficiently.
- Does not require a complete duplicate environment.
- Allows gradual replacement of instances.

Disadvantages:
- Multiple application versions run simultaneously.
- Database changes must remain compatible with both versions.
- Capacity can temporarily decrease while instances are being replaced.

**Blue-green deployment**

Two environments are maintained: blue is the current production environment, and green is the new version.

```text
             Load Balancer
                  |
             Traffic Switch
               /       \
          Blue (V1)   Green (V2)
```

The new version is deployed and tested in green. Once it is ready, traffic is switched from blue to green.

Advantages:
- Fast traffic switching.
- Straightforward application rollback if the old environment remains available.
- The new environment can be validated before receiving production traffic.

Disadvantages:
- Requires additional infrastructure during the deployment.
- Database changes and external side effects may not be easy to roll back.
- Switching traffic does not automatically migrate existing sessions or long-lived connections.

**Canary deployment**

A small percentage of production traffic is sent to the new version first. The team monitors errors, latency, and other metrics before gradually increasing traffic.

For example:

```text
Initial: 95% V1 | 5% V2
Next:    75% V1 | 25% V2
Next:    50% V1 | 50% V2
Final:    0% V1 | 100% V2
```

Advantages:
- Limits exposure if the new version has a problem.
- Supports validation under real production traffic.
- Allows automated rollback based on metrics.

Disadvantages:
- Requires reliable traffic routing and monitoring.
- Users may experience different versions during rollout.
- Database and API compatibility must be maintained across versions.

### 9. How Do You Handle Database Changes During Deployment?

A common mistake is deploying a new application version that expects a database column before the schema change has been safely applied.

I prefer an **expand-and-contract migration strategy**:

1. Add the new schema elements in a backward-compatible way.
2. Deploy application code that can work with the old and new schema.
3. Migrate existing data if needed.
4. Switch application reads and writes to the new structure.
5. Remove obsolete schema elements only after the old application version is no longer in use.

This is particularly important during rolling, blue-green, and canary deployments because multiple application versions may access the same database concurrently.

### 10. My Practical Approach

For a production API, I would:

1. Keep API instances stateless where practical.
2. Store sessions in shared storage or use an appropriate token-based authentication design.
3. Store uploads in shared object storage.
4. Use a load balancer with round-robin or least-connections according to the workload.
5. Configure readiness and liveness checks separately.
6. Implement graceful shutdown and bounded request timeouts.
7. Use rolling deployments for routine updates, blue-green for straightforward environment switching, or canary releases when gradual production validation is valuable.
8. Make database migrations backward-compatible and monitor error rates, latency, and instance health during releases.

**In short:** Statelessness makes horizontal scaling easier, load balancing distributes traffic, health checks and graceful shutdown protect request handling, and deployment strategies reduce release risk. The critical requirement is to externalize necessary state and keep application versions compatible while instances are being added, removed, or replaced.

*Q20. Real-time communication. Compare polling, long polling, Server-Sent Events and WebSockets. How do you scale WebSockets across multiple instances (pub/sub backplane such as Redis), handle authentication on connect, heartbeats, reconnection and message ordering/delivery guarantees?*

Real-time communication allows a server to deliver updates to clients with minimal delay, without requiring the user to refresh the page manually.

For example, in a restaurant management system, when a waiter places an order, the kitchen display system should receive the order immediately. Similarly, order status changes should appear on the waiter’s screen without repeatedly fetching the entire order list.

I choose the communication mechanism based on how frequently data changes, whether communication needs to be bidirectional, and how many concurrent clients the application must support.

### 1. Polling

Polling means the client sends HTTP requests to the server at regular intervals to check whether new data is available.

For example:

```javascript
setInterval(async () => {
  const response = await fetch("/api/orders");
  const orders = await response.json();
  updateOrders(orders);
}, 5000);
```

Here, the client checks for updates every five seconds.

**Advantages:**
- Simple to implement using ordinary HTTP APIs.
- Easy to debug and operate.
- Works with standard HTTP infrastructure.

**Disadvantages:**
- Generates repeated requests even when nothing changes.
- Updates may be delayed until the next polling interval.
- Frequent polling increases server and network overhead.

I would use polling for dashboards where updates every few seconds are sufficient and implementation simplicity is important.

### 2. Long Polling

Long polling is similar to polling, but the server holds an HTTP request open until new data becomes available or a timeout occurs.

The client receives the response and immediately opens another request.

```text
Client                 Server
  |                       |
  |---- Request --------->|
  |                       | Wait for event
  |                       |
  |<--- New data ---------|
  |                       |
  |---- Next request ---->|
```

**Advantages:**
- Delivers updates sooner than ordinary interval-based polling.
- Works over conventional HTTP.
- Can be useful when persistent bidirectional connections are unavailable.

**Disadvantages:**
- Repeated request creation and connection management add overhead.
- Large numbers of waiting requests consume server resources.
- It is more complex than ordinary polling under heavy concurrency.

I would consider long polling for applications that need near-real-time updates but cannot use a persistent streaming connection.

### 3. Server-Sent Events (SSE)

SSE allows a server to push events to a client over a long-lived HTTP connection. Communication is primarily **one-way, from server to client**.

The browser uses the `EventSource` API:

```javascript
const source = new EventSource("/api/notifications");

source.onmessage = (event) => {
  const notification = JSON.parse(event.data);
  console.log(notification);
};

source.onerror = () => {
  console.log("Connection interrupted");
};
```

The server sends events using the `text/event-stream` content type.

**Advantages:**
- Supports server-to-client streaming over HTTP.
- Has built-in browser reconnection behavior.
- Supports event IDs that can help resume a stream.
- Is simpler than WebSockets when clients only need to receive updates.

**Disadvantages:**
- The standard browser API is designed for one-way communication.
- Authentication with custom headers is awkward with native `EventSource`.
- Connection limits and proxy buffering or timeout settings need consideration.

I would use SSE for notifications, live dashboards, progress updates, and streaming status messages.

### 4. WebSockets

WebSockets provide persistent, full-duplex communication between a client and server over a single connection.

After the initial HTTP handshake, the connection is upgraded to the WebSocket protocol, allowing both sides to send messages independently.

```text
Client                    Server
  |                          |
  |---- HTTP Upgrade ------->|
  |<--- Upgrade Accepted ----|
  |                          |
  |<====== Message =========>|
  |<====== Message =========>|
  |<====== Message =========>|
```

For example, in a restaurant ERP system, a waiter can send an order update while the kitchen display receives new orders immediately. The server can also broadcast status changes to authorized clients.

**Advantages:**
- Supports bidirectional communication.
- Avoids repeatedly establishing HTTP requests.
- Works well for interactive, low-latency applications.

**Disadvantages:**
- Requires connection lifecycle management.
- Persistent connections consume resources.
- Scaling across multiple instances requires routing and message distribution strategies.
- Reconnection and delivery guarantees must be designed at the application level.

I would choose WebSockets for chat applications, collaborative interfaces, live order management, and other applications requiring frequent bidirectional updates.

### 5. Comparing the Four Approaches

| Feature | Polling | Long polling | SSE | WebSockets |
|---|---|---|---|---|
| Communication | Request/response | Request/response held open | Server to client | Bidirectional |
| Connection | Repeated requests | Repeated long requests | Persistent HTTP stream | Persistent WebSocket |
| Update latency | Depends on interval | Usually low | Low | Low |
| Browser reconnection | Application-managed | Application-managed | Built in | Application-managed |
| Best use case | Periodic refresh | Near-real-time updates | Notifications and streams | Interactive real-time systems |

### 6. How Do You Scale WebSockets Across Multiple Instances?

A WebSocket connection remains attached to the server instance that accepted it. Suppose a user is connected to API 1, but an event is generated by API 2. API 2 cannot directly send that event through API 1's in-memory connection registry.

I would use a **pub/sub backplane**, such as Redis Pub/Sub, to distribute events across application instances.

```text
                Clients
              /    |    \
             /     |     \
          API 1   API 2   API 3
             \      |      /
              \     |     /
             Redis Pub/Sub
```

The flow works as follows:

1. A client establishes a WebSocket connection to API 1.
2. API 1 tracks the connection and its authorized user or groups.
3. An event is generated by API 2.
4. API 2 publishes the event to the shared backplane.
5. The relevant API instances receive the event.
6. Each instance delivers it to its locally connected, authorized clients.

For example, if a kitchen order is created through API 2, a waiter connected to API 1 can still receive the notification through the backplane.

In ASP.NET Core SignalR, a Redis backplane can distribute messages between server instances. For a multi-server deployment, I would also verify the deployment's connection affinity and transport requirements.

**Important:** Redis Pub/Sub distributes live messages but does not durably retain them for disconnected subscribers. If an event must not be lost, I would persist it in a database or use a durable messaging system and provide a replay or synchronization mechanism.

### 7. How Do You Authenticate a WebSocket Connection?

Authentication must happen when the connection is established, and authorization must also be enforced for subsequent messages and subscriptions.

My approach would be:

1. The client authenticates using the application's normal login flow.
2. It establishes the WebSocket connection over `wss://`.
3. The server validates the session or access token during the handshake.
4. The server associates the connection with the authenticated user.
5. The server checks permissions before allowing access to particular groups or events.

For browser clients, cookies can be useful when configured with appropriate `Secure`, `HttpOnly`, and `SameSite` settings. The server must also validate the `Origin` header against an allowlist to mitigate cross-site WebSocket hijacking.

Browser WebSocket APIs do not allow arbitrary custom headers in the same way as ordinary `fetch` requests. If token-based authentication is needed, I would consider a secure cookie or a short-lived connection ticket. I would avoid putting long-lived access tokens in query strings because URLs may be logged.

In ASP.NET Core SignalR, browser clients sometimes pass an access token through the `access_token` query parameter for WebSocket or Server-Sent Events connections. If this approach is necessary, I would restrict it to the appropriate endpoint and prevent token values from appearing in logs.

I would also handle token expiry, permission changes, connection limits, and authorization when joining user-specific or tenant-specific groups.

### 8. Heartbeats and Connection Health

A connection can appear open even when the network has failed or a device has gone offline.

I use heartbeat mechanisms to detect stale connections.

WebSockets support protocol-level Ping and Pong control frames. A server or client can send a Ping and expect a Pong within a defined timeout. If the peer does not respond, the connection can be closed and re-established.

I would configure:
- Appropriate heartbeat intervals.
- A timeout for missed heartbeats.
- Proxy and load-balancer idle timeouts that accommodate the connection.
- Cleanup of disconnected clients and subscriptions.

Heartbeat settings should be coordinated across the client, server, and network infrastructure. Sending heartbeats too frequently wastes resources, while excessively long intervals delay detection of dead connections.

### 9. Reconnection and Message Delivery

WebSocket connections can break because of network changes, server restarts, deployments, or mobile devices entering sleep mode. The client should reconnect with exponential backoff and jitter.

For example:

```javascript
let retryDelay = 1000;

function getRetryDelay() {
  const jitter = Math.random() * retryDelay;
  const delay = retryDelay + jitter;

  retryDelay = Math.min(retryDelay * 2, 30000);
  return delay;
}
```

This is a simplified backoff example; production code must also schedule the reconnection attempt, handle cancellation, and reset the delay after a stable connection is restored.

A successful reconnection does not guarantee that the client received every event sent while it was disconnected.

For important events, I would design an explicit delivery strategy:

- Assign each event a unique ID.
- Include an aggregate or stream sequence number where ordering matters.
- Persist important events.
- Allow clients to resume from their last acknowledged event or sequence number.
- Deduplicate repeated events.
- Reconcile with the server's current state after reconnecting.

For example, after reconnecting, a kitchen display can fetch the current active orders and then resume receiving live updates.

### 10. Message Ordering and Delivery Guarantees

WebSocket preserves the order of messages sent over a single connection. However, this does not guarantee global ordering across multiple application instances, concurrent publishers, or independent asynchronous handlers.

If ordering matters, I would partition events by a key such as `orderId` and assign sequence numbers to the events for that order.

For example:

```json
{
  "eventId": "evt-104",
  "orderId": "ORD-501",
  "sequence": 4,
  "type": "OrderStatusChanged",
  "status": "Preparing"
}
```

The client can detect missing or out-of-order sequence numbers and request synchronization when necessary.

I also distinguish between delivery guarantees:

- **At-most-once:** A message may be lost, but it is not intentionally redelivered.
- **At-least-once:** A message is retried until acknowledged or otherwise resolved, so duplicates are possible.
- **Effectively-once processing:** Duplicate deliveries are made harmless through idempotency and deduplication.

End-to-end exactly-once delivery is not something I would assume WebSockets provide. It requires carefully coordinated persistence, acknowledgement, retry, and side-effect handling.

For business-critical events, I would persist the event before publishing it and use a durable message mechanism when necessary. For transient UI updates, such as cursor positions or typing indicators, occasional loss may be acceptable.

### 11. My Practical Choice

For a restaurant ERP or similar business application, I would use:

- **Polling:** For low-frequency refreshes and simple status checks.
- **SSE:** For one-way notifications and long-running operation progress.
- **WebSockets or SignalR:** For real-time order updates, kitchen displays, and bidirectional interactions.
- **Redis backplane:** To distribute live messages between multiple SignalR or WebSocket server instances.
- **Durable storage or messaging:** For business events that must survive disconnections and server restarts.

I would combine this with authentication, group-level authorization, heartbeats, reconnection with jitter, event IDs, and state synchronization.

**In short:** Polling repeatedly asks for updates, long polling holds requests open, SSE streams events from server to client, and WebSockets enable bidirectional communication. Scaling WebSockets requires a shared event-distribution mechanism, while reliable delivery requires additional application-level guarantees beyond the connection itself.

*Q21. File handling and large payloads. How do you handle uploads safely and at scale (multipart vs presigned URLs to object storage, size and type validation, virus scanning)? How do streaming responses, backpressure and chunked transfer avoid loading whole files into memory, and how do you serve private files securely (signed URLs, CDN)?*

In my approach, I handle file uploads by validating them securely, avoiding unnecessary memory usage, and storing large files in object storage instead of directly in the application server. The exact approach depends on the file size, security requirements, and expected traffic.

## 1. Multipart Uploads vs. Presigned URLs

**Multipart uploads** are useful when the client uploads a file to the backend using `multipart/form-data`. The backend validates the request and either streams the file to storage or processes it in chunks.

For example, in a React application, I can use `FormData` to upload a profile image or document to a .NET API.

For larger files or high traffic, I prefer **presigned URLs**. The backend authenticates the user, checks their upload permissions, and generates a short-lived URL for object storage, such as Amazon S3. The client uploads the file directly to storage, so the file does not pass through my application server.

This reduces backend bandwidth, memory usage, and load.

- **Multipart upload through the backend:** Suitable for smaller files or when the backend must inspect or transform the content during upload.
- **Presigned URL:** Suitable for large files, videos, documents, and high-volume applications.
- **Multipart object-storage upload:** For very large files, storage services can support uploading separate parts, allowing retries without restarting the entire upload.

## 2. Upload Security and Validation

I never trust the filename, extension, or MIME type supplied by the client. I apply multiple validation layers:

1. **Authentication and authorization:** Verify that the user is logged in and allowed to upload to the requested resource.
2. **File-size limits:** Enforce maximum sizes at the reverse proxy, API, and storage layers where applicable.
3. **File-type validation:** Check the extension and declared MIME type, then inspect the actual file signature or magic bytes.
4. **Filename safety:** Generate a unique server-side object key instead of using the user's filename directly. This helps prevent path traversal and filename collisions.
5. **Virus scanning:** Scan files using an antivirus engine such as ClamAV or a managed malware-scanning service, especially for documents and executable-capable formats.
6. **Storage isolation:** Store uploads outside the public web root, with private access by default.
7. **Rate limiting and quotas:** Prevent abuse by limiting upload frequency, total storage, and concurrent uploads.

For presigned uploads, I also restrict the URL's permissions, object key, expiration, and allowed content conditions where supported. I do not treat the URL itself as proof that the uploaded content is safe. I can upload to a quarantine location, scan the file, and move or mark it as available only after validation succeeds.

## 3. Streaming Responses and Memory Efficiency

When handling large files, I avoid reading the entire file into memory. Instead, I process it as a stream of smaller chunks.

For example, loading a 2 GB file into a byte array can consume a large amount of memory per request. If several users do this concurrently, the application can run out of memory.

With streaming, the application reads a chunk, processes or forwards it, and then continues with the next chunk. This keeps memory usage more predictable.

In .NET, I can use `Stream`, `FileStream`, and asynchronous methods such as `CopyToAsync()` to transfer data without buffering the entire file in application memory.

```csharp
await using var input = file.OpenReadStream();
await using var output = new FileStream(
    destinationPath,
    FileMode.CreateNew,
    FileAccess.Write,
    FileShare.None,
    bufferSize: 81920,
    useAsync: true
);

await input.CopyToAsync(output, cancellationToken);
```

In production, I would also enforce upload limits, handle cancellation and exceptions, and ensure partial files are cleaned up if the upload fails. For object storage, I would generally use its SDK or presigned-upload workflow rather than writing to local disk.

## 4. Backpressure and Chunked Transfer

**Backpressure** means a producer should not continue generating or sending data faster than the consumer can handle it.

For example, if a client downloads a large file over a slow connection, the server should not keep reading the entire file into memory while waiting for the client. Streaming APIs, bounded buffers, and proper flow control allow the transfer to slow down as needed.

**Chunked transfer encoding** is an HTTP/1.1 mechanism that allows a response to be sent in chunks without specifying the complete content length in advance. It can be useful for dynamically generated responses.

However, chunked transfer encoding is not the same as application-level file chunking, and it is not required for all streaming responses. If the content length is known, the server can send a `Content-Length` header instead. HTTP/2 and HTTP/3 use their own framing mechanisms rather than HTTP/1.1 chunked transfer encoding.

The important principle is to stream data incrementally, use bounded buffers, propagate cancellation, and avoid unnecessary buffering by the application or reverse proxy.

## 5. Serving Private Files Securely

I keep private files in non-public object storage and never expose permanent public URLs for sensitive documents.

There are two common approaches:

- **Backend-mediated download:** The client requests a file from the API. The backend authenticates the user, checks authorization, and streams the file or redirects to a temporary storage URL.
- **Presigned download URL:** The backend checks the user's permissions and generates a short-lived URL for downloading a specific object. The client downloads directly from object storage.

For frequently accessed private files, I can also use a CDN with signed URLs or signed cookies. The CDN must validate the signature and expiry, and its cache configuration must prevent one user's private content from being served to another user.

I also consider `Content-Disposition` to control whether a file is downloaded or displayed, and `X-Content-Type-Options: nosniff` to reduce content-type sniffing risks. Authorization must happen before issuing a download URL, and the URL should grant only the minimum required access.

## 6. Handling Failures and Scaling

At scale, I would design uploads to be resumable where possible, retry failed parts, and clean up incomplete multipart uploads. I would store file metadata—such as owner, object key, size, content type, scan status, and creation time—in the database, while keeping the actual file in object storage.

For long-running tasks such as video processing or malware scanning, I would put a job on a queue and process it asynchronously. The API could return `202 Accepted` with an upload or processing status that the client can poll.

**In summary:** For small uploads, multipart requests are often sufficient. For large files, direct uploads using presigned URLs are usually more scalable. I validate uploads at multiple layers, stream large payloads instead of buffering them entirely, apply backpressure, and keep private files behind authorization or short-lived signed access.

*Q22. Configuration, deployment and 12-factor practices. How do you manage config and secrets across environments (env vars, vault, never in git)? What do Docker images, multi-stage builds and container health checks give you? Explain CI/CD pipeline stages, database migration strategies for zero-downtime releases (expand/contract), and rollback plans.*

In my approach, I keep application configuration separate from code, manage secrets securely, and make deployments repeatable through Docker and CI/CD pipelines. My goal is to deploy the same application reliably across development, staging, and production without exposing credentials or causing unnecessary downtime.

## 1. Configuration and Secrets Management

I separate configuration from application code so the same build can run in different environments.

For example, a React frontend, .NET API, and MySQL database might use different database connections, API URLs, logging levels, and feature flags in development and production.

I use environment variables or environment-specific configuration files for non-sensitive settings. For sensitive values, such as database passwords, JWT signing keys, API tokens, and cloud credentials, I use a secrets manager such as HashiCorp Vault or a cloud provider's secret-management service.

My main practices are:

- Never commit passwords, API keys, private certificates, or production connection strings to Git.
- Keep local development settings in ignored files, such as `.env`, and provide safe example files without real credentials.
- Inject production secrets at runtime through the deployment platform or secrets manager.
- Apply least-privilege access so each service can access only the secrets it needs.
- Rotate credentials periodically and immediately when exposure is suspected.
- Avoid printing secrets in logs, error responses, or CI/CD output.

In .NET, I can use `appsettings.json` for safe defaults and environment-specific configuration, environment variables for deployment overrides, and a secret manager for sensitive production values. ASP.NET Core User Secrets is useful for local development, but it is not a production secrets vault.

The key principle is that **configuration changes between environments, but the application artifact should remain the same**.

## 2. Docker Images and Multi-Stage Builds

Docker packages an application and its runtime dependencies into an image, making deployments more consistent across development, testing, and production.

For example, a .NET API can run inside a container with a known .NET runtime instead of depending on the server's manually configured environment.

I use multi-stage builds to separate compilation from runtime execution. The first stage uses an SDK image to restore dependencies, build, and publish the application. The final stage uses a smaller runtime image and copies only the published output.

Benefits include:

- Smaller production images and reduced attack surface.
- Faster and more predictable deployments.
- Separation of build-time tools from runtime dependencies.
- Easier image versioning and promotion between environments.
- More consistent local development and production behavior.

I also use `.dockerignore` to avoid copying unnecessary files, pin base-image versions, scan images for known vulnerabilities, and avoid embedding secrets in image layers.

## 3. Container Health Checks

A container being running does not necessarily mean the application is working correctly. Health checks help detect unhealthy instances.

I distinguish between:

- **Liveness:** Is the application process functioning, or is it stuck and potentially needs restarting?
- **Readiness:** Is the application ready to receive traffic?
- **Startup checks:** Does the application need additional time to initialize before regular health checks begin?

For example, a .NET API can expose a health endpoint such as `/health`. A container orchestrator or deployment platform can use the result to decide whether the instance should receive traffic or be restarted.

I avoid making liveness checks depend on every external service, because a temporary database outage could otherwise cause repeated application restarts. Readiness checks can include critical dependencies when appropriate.

## 4. CI/CD Pipeline Stages

CI/CD automates the process of validating, packaging, and deploying code. A typical pipeline looks like this:

1. **Source control:** A developer pushes code or opens a pull request.
2. **Build:** Restore dependencies and compile the application.
3. **Automated tests:** Run unit tests, integration tests, and relevant API or frontend tests.
4. **Code and security checks:** Run linting, static analysis, dependency vulnerability scans, and secret detection.
5. **Artifact creation:** Build a versioned Docker image and publish it to a container registry.
6. **Deployment to staging:** Deploy the same artifact to a production-like environment.
7. **Validation:** Run smoke tests, health checks, and critical end-to-end tests.
8. **Production deployment:** Release through a controlled strategy, such as rolling deployment or blue-green deployment.
9. **Post-deployment monitoring:** Check error rates, latency, logs, and business-critical operations.

For example, I could use GitHub Actions to build and test a .NET API, create a Docker image, push it to a registry, and deploy it to a server or container platform.

I prefer immutable, versioned artifacts. If version `v1.4.2` passes staging tests, I promote that same image to production rather than rebuilding it separately.

## 5. Database Migrations and Zero-Downtime Releases

Database changes are one of the most challenging parts of deployments because the old and new application versions may run simultaneously during a rolling release.

I use backward-compatible migrations and the **expand-and-contract** strategy to reduce deployment risk.

Suppose I need to rename a database column from `full_name` to `display_name`.

A risky approach is to rename the column and deploy the new application at the same time. The old application version may still expect `full_name`, causing errors.

Instead, I use these steps:

**Phase 1 — Expand**

- Add the new `display_name` column without immediately removing `full_name`.
- Deploy application code that can work with the old and new schema.
- If necessary, temporarily write to both columns and backfill existing records.

**Phase 2 — Migrate**

- Move reads to `display_name`.
- Backfill and verify existing data.
- Monitor for unexpected null values, mismatches, or application errors.

**Phase 3 — Contract**

- Confirm that no running application version depends on `full_name`.
- Stop writing to and reading from the old column.
- Remove the old column in a later deployment.

This approach allows old and new application versions to coexist while the schema evolves.

For database migrations, I also:

- Review potentially destructive operations and lock behavior.
- Test migrations against realistic data volumes.
- Run migrations through a controlled deployment step rather than allowing every application instance to execute them concurrently.
- Ensure migration scripts are versioned, repeatable where appropriate, and observable.
- Plan for long-running backfills separately from schema changes.

Zero-downtime deployment is not guaranteed by expand-and-contract alone; it also depends on backward-compatible application code, database locking, traffic routing, and the nature of the change.

## 6. Deployment Strategies and Rollbacks

I choose the deployment strategy based on system complexity, traffic, and infrastructure.

- **Rolling deployment:** Replace application instances gradually. It reduces disruption but requires compatibility between versions.
- **Blue-green deployment:** Run the new version alongside the old one, validate it, and switch traffic when ready. The old version can remain available for application rollback.
- **Canary deployment:** Release to a small portion of traffic first, monitor results, and gradually increase exposure.

I define rollback criteria before deployment, such as a sustained increase in HTTP 5xx errors, a significant latency increase, failed health checks, or broken critical business flows.

If the new version causes problems, I can route traffic back to the previous application version or redeploy the previous known-good image.

However, **rolling back application code does not automatically roll back the database safely**. Database changes should generally remain backward-compatible during the rollback window. If data has already been transformed or discarded, recovery may require a forward fix, a carefully designed compensating migration, or restoration from a verified backup.

For critical systems, I also verify backup and recovery procedures rather than assuming that a backup exists and is usable.

## 7. The 12-Factor App Principles

The 12-Factor App methodology provides guidelines for building maintainable, deployable cloud applications. The principles I apply most directly are:

- **Codebase:** Keep the application in version control with a clearly managed deployment history.
- **Dependencies:** Declare and isolate dependencies rather than relying on manually installed packages.
- **Config:** Keep environment-specific configuration outside the codebase.
- **Backing services:** Treat databases, caches, and message queues as attached resources accessed through configuration.
- **Build, release, run:** Separate building the artifact, combining it with configuration, and running it.
- **Processes:** Keep application processes stateless where possible and store persistent data in external services.
- **Logs:** Treat logs as event streams and send them to a centralized logging system.
- **Disposability:** Start quickly and shut down gracefully, allowing deployments to replace instances safely.
- **Dev/prod parity:** Keep development, staging, and production environments as similar as practical.

These principles make applications easier to test, scale, observe, and deploy consistently.

## Final Summary

My deployment approach is to externalize configuration, protect secrets, build secure and minimal Docker images, and automate testing and deployment through CI/CD. For database changes, I use backward-compatible expand-and-contract migrations so application versions can coexist during releases. I also define monitoring thresholds, rollback procedures, and database recovery plans before production deployment.

The main objective is to make releases **repeatable, secure, observable, and recoverable without unnecessary downtime**.

*Q23. REST vs GraphQL vs gRPC. What problems does each solve (over-fetching, N+1 resolvers, type-safe contracts, streaming)? What are the trade-offs in caching, versioning, error handling, tooling and browser support, and where would you use each in one system (public API, mobile BFF, internal service calls)?*

In my understanding, REST, GraphQL, and gRPC solve different API communication problems. I choose between them based on the client requirements, data structure, performance, caching, and whether the communication happens between a browser, mobile app, or backend services.

## 1. REST API

REST is a resource-oriented API style that uses standard HTTP methods such as `GET`, `POST`, `PUT`, `PATCH`, and `DELETE`. It commonly exchanges JSON data.

For example, in a restaurant management application, I might have endpoints such as:

- `GET /api/orders` — retrieve orders.
- `GET /api/orders/101` — retrieve a specific order.
- `POST /api/orders` — create an order.
- `PATCH /api/orders/101` — update an order.
- `DELETE /api/orders/101` — delete an order, if permitted.

**Advantages:**
- Simple to implement, understand, test, and maintain.
- Works natively with browsers and virtually all HTTP clients.
- Supports HTTP caching through `Cache-Control`, `ETag`, and conditional requests.
- Uses standard HTTP status codes, making error handling straightforward.
- OpenAPI and Swagger can document endpoints and generate client code.

**Limitations:**

REST can cause over-fetching when an endpoint returns more fields than the client needs. It can also cause under-fetching when a screen needs data from several endpoints, requiring multiple network requests.

For example, a mobile order screen might need the order status, customer name, and total amount, while the API returns the complete order object with many additional fields.

I can address this by designing focused endpoints, supporting field selection where appropriate, or introducing a Backend for Frontend (BFF).

## 2. GraphQL

GraphQL allows clients to specify exactly which fields they need through a query. Instead of exposing multiple resource endpoints, a typical GraphQL API exposes a single endpoint, such as `/graphql`.

For example, a mobile application might request:

```graphql
query {
  order(id: "101") {
    id
    status
    total
    customer {
      name
    }
  }
}
```

The server returns the requested fields rather than a fixed, potentially larger response.

**Advantages:**
- Reduces over-fetching by allowing clients to select fields.
- Can reduce under-fetching by retrieving related data in one request.
- Provides a strongly typed schema and introspection.
- Supports client-code generation and tools such as GraphiQL.
- Works well when web and mobile clients require different response shapes.

**Limitations:**

GraphQL introduces complexity on the server side. Each field may invoke a resolver, and poorly designed resolvers can cause the **N+1 query problem**.

For example, if I retrieve 100 orders and each order separately queries the database for its customer, the server might execute one query for the orders and another 100 queries for the customers.

I can solve this using batching and caching tools such as DataLoader, efficient database joins, and carefully designed resolver logic.

I would also apply query depth and complexity limits, pagination, authorization checks at the appropriate data-access boundaries, and rate limiting to prevent expensive queries from exhausting server resources.

GraphQL caching is also more involved than standard REST caching because different queries can request different data through the same endpoint. Client-side normalized caches and persisted queries can help.

## 3. gRPC

gRPC is a remote procedure call framework that commonly uses Protocol Buffers (Protobuf) for serialization and HTTP/2 for transport.

Instead of requesting a resource through a URL, the client calls a method defined in a service contract.

For example, a restaurant application might define:

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest)
      returns (OrderResponse);
}

message GetOrderRequest {
  string order_id = 1;
}

message OrderResponse {
  string order_id = 1;
  string status = 2;
  double total = 3;
}
```

The `.proto` file defines the contract, and tools generate typed client and server code for supported languages.

**Advantages:**
- Compact binary messages can reduce payload size and serialization overhead.
- Strong contracts and generated clients reduce integration mistakes.
- HTTP/2 supports multiplexing multiple streams over a connection.
- Supports unary calls, server streaming, client streaming, and bidirectional streaming.
- Well suited to frequent communication between backend services.

For example, an order service could call an inventory service to reserve stock, using a generated client with deadlines and cancellation.

**Limitations:**
- Browser clients cannot generally call standard gRPC directly; gRPC-Web or another compatible gateway is needed.
- Binary payloads are less convenient to inspect manually than JSON.
- HTTP-level caching and public API integration are less straightforward.
- Teams need to manage `.proto` contracts and maintain backward compatibility.

For streaming, gRPC is particularly useful for backend-to-backend communication, telemetry, and systems that exchange continuous streams of data.

## 4. Key Differences

| Feature | REST | GraphQL | gRPC |
|---|---|---|---|
| Communication model | Resources and HTTP methods | Client-defined queries and mutations | Remote procedure calls |
| Data format | Usually JSON | Usually JSON responses | Usually binary Protobuf |
| Contract | OpenAPI is optional but common | Typed schema required | `.proto` contract required |
| Over-fetching | Possible | Reduced through field selection | Fixed message structure |
| N+1 risk | Possible in backend data access | Common resolver-design risk | Possible in service/database calls |
| Caching | Strong HTTP and CDN support | Usually needs client or custom caching | Usually custom caching |
| Error handling | HTTP status codes | `data` plus `errors` may be returned | gRPC status codes |
| Versioning | URL, headers, or compatible evolution | Evolve schema and deprecate fields | Evolve Protobuf messages compatibly |
| Streaming | SSE or WebSockets can be added | Subscriptions, typically over SSE or WebSockets | Native streaming RPCs |
| Browser support | Native | Native | Requires a compatible bridge |
| Best fit | Public APIs and CRUD | Flexible frontend data requirements | Internal service communication |

## 5. Caching, Versioning, and Error Handling

### Caching

With REST, I can cache a public `GET` response at a browser, reverse proxy, or CDN using HTTP caching headers. Private or user-specific responses need careful cache controls.

With GraphQL, I would commonly use a normalized client cache and, where beneficial, persisted queries or a server-side cache. I would not assume every GraphQL request is automatically cacheable.

With gRPC, I would generally implement caching at the application or service layer when it makes sense. Standard gRPC does not provide the same straightforward HTTP CDN caching model as REST.

### Versioning

For REST, I can introduce a new API version, such as `/api/v2/orders`, when a breaking change is necessary. For compatible changes, I prefer adding optional fields and preserving existing behavior.

For GraphQL, I generally evolve the schema incrementally, add new fields, deprecate old fields, and remove them only after clients have migrated.

For gRPC, I preserve compatibility when modifying Protobuf messages. I avoid reusing field numbers and carefully manage field removal, renaming, and type changes.

### Error handling

In REST, I use appropriate HTTP status codes such as `400`, `401`, `403`, `404`, and `500`, together with a consistent error response.

In GraphQL, an HTTP response may contain both partial `data` and an `errors` array. Therefore, the client must inspect GraphQL errors rather than assuming an HTTP `200` means every field resolved successfully.

In gRPC, I use standard status codes such as `INVALID_ARGUMENT`, `NOT_FOUND`, `UNAUTHENTICATED`, and `UNAVAILABLE`. I also configure deadlines and cancellation so failed downstream calls do not consume resources indefinitely.

## 6. Where I Would Use Each in One System

Suppose I am building a restaurant ERP with a web dashboard, React Native mobile application, and several backend services.

**Public API — REST**

I would expose REST endpoints for restaurant management, orders, menu items, authentication, and third-party integrations. It is easy for external developers to consume and works well with API gateways, standard HTTP tooling, and caching.

**Mobile BFF — GraphQL or REST**

If the mobile application needs data aggregated from orders, customers, menus, and inventory, and different screens require very different fields, I would consider GraphQL for the mobile-facing BFF.

If the mobile application has relatively simple workflows, predictable payloads, and straightforward endpoint requirements, I would keep REST and create a few purpose-built BFF endpoints instead. GraphQL is not automatically better just because the client is mobile.

**Internal service calls — gRPC**

If I split the system into separate order, inventory, and notification services, I would consider gRPC for internal calls where strict contracts, low serialization overhead, and streaming are valuable.

For example, the order service could call the inventory service through a typed RPC to reserve stock. I would still handle timeouts, retries carefully, authorization, and failure scenarios. gRPC does not eliminate distributed-system problems.

For asynchronous business events, such as `OrderCreated` or `PaymentCompleted`, I would also consider a message broker rather than forcing every interaction through synchronous API calls.

## Final Summary

My default decision would be REST for public APIs and conventional CRUD operations, GraphQL when multiple frontend clients need flexible data selection, and gRPC for internal services that benefit from strongly typed contracts and efficient streaming.

I would not choose based on performance claims alone. I would consider actual payload sizes, network latency, query patterns, caching requirements, browser compatibility, operational complexity, and the team's experience. A single system can use all three at different boundaries when each choice solves a real problem.

*Q24. Database internals: MVCC, WAL and locking. How do MVCC and the write-ahead log let readers and writers avoid blocking each other and survive a crash? What causes table bloat and vacuum issues in Postgres, how do deadlocks form and get resolved, and what do lock types (row, table, gap/next-key in InnoDB) mean in practice?*

In my understanding, MVCC, the Write-Ahead Log, and database locking work together to provide concurrency, consistency, and crash recovery. They allow a database to handle multiple transactions safely without forcing every read and write to block each other.

I'll use PostgreSQL and MySQL with InnoDB as examples because their locking and storage implementations differ in important ways.

## 1. What Is MVCC?

MVCC stands for **Multi-Version Concurrency Control**. Instead of maintaining only one visible version of a row, the database keeps enough version information to determine which version each transaction is allowed to see.

For example, suppose an order has a total of ₹1,000. One transaction updates it to ₹1,500 while another transaction reads it.

With MVCC, the reader can continue seeing the appropriate committed version while the writer creates a newer version. Under common isolation levels, a normal read does not need to wait for the row update to finish.

In PostgreSQL, an updated row generally creates a new tuple version, while the old version remains until it is safe to remove. In InnoDB, older row versions are reconstructed using undo information.

**Benefits of MVCC:**
- Readers and writers can often proceed concurrently.
- Transactions see data according to their isolation level.
- It reduces unnecessary read-write blocking.
- It helps provide consistent snapshots for queries and transactions.

However, MVCC does not eliminate all locking. Conflicting writes can still block each other, and stronger isolation levels may introduce additional conflicts.

## 2. How MVCC Works in PostgreSQL

PostgreSQL associates row versions with transaction visibility information, including transaction IDs that indicate when versions were created or invalidated.

Imagine the following sequence:

1. Transaction A reads an account balance of ₹5,000.
2. Transaction B updates the balance to ₹4,000 and commits.
3. Transaction A performs another read.

Under PostgreSQL's default `READ COMMITTED` isolation level, each statement gets a new snapshot. Therefore, Transaction A's second `SELECT` can see ₹4,000.

Under `REPEATABLE READ`, Transaction A generally continues seeing the version visible in its transaction snapshot, so the second read sees ₹5,000.

This is why I distinguish between **statement-level snapshots** and **transaction-level snapshots**.

An important detail is that PostgreSQL does not immediately overwrite the old tuple version in place. The old version can remain physically present even though new transactions no longer see it.

## 3. What Is the Write-Ahead Log (WAL)?

The Write-Ahead Log records changes that may need to be recovered after a crash.

The central rule is:

**The relevant WAL records must be flushed to durable storage before the corresponding modified data pages are considered safely written to their data files.**

For example, suppose a transaction updates an account balance from ₹5,000 to ₹4,000.

A simplified sequence is:

1. The database generates WAL records describing the change.
2. The relevant data page is modified in memory.
3. At commit, PostgreSQL ensures the required WAL records are flushed to durable storage when synchronous durability is enabled.
4. The transaction is acknowledged as committed.
5. The modified data page can be written to the main data files later.

The data page does not need to be written to its final location before every commit. This reduces the amount of synchronous disk I/O needed for transactions.

### How WAL helps with crash recovery

Suppose the server crashes after the transaction commits but before the modified data page is written to disk.

When PostgreSQL restarts, it replays the necessary WAL records to bring the database files to a consistent state. This is commonly called **redo recovery**.

WAL also supports point-in-time recovery when combined with an appropriate base backup and archived WAL.

WAL is not a replacement for backups. It supports recovery, but long-term protection against accidental deletion, corruption, or other failures requires an appropriate backup and recovery strategy.

Durability also depends on configuration and the storage system. For example, changing synchronous commit behavior can change when a transaction is acknowledged relative to WAL flushing.

## 4. How MVCC and WAL Work Together

MVCC and WAL solve different problems.

| Mechanism | Main purpose |
|---|---|
| MVCC | Determines which row versions a transaction can see |
| WAL | Makes changes recoverable after a crash |
| Locks | Coordinate conflicting operations and protect shared resources |
| Isolation levels | Define which concurrency anomalies transactions may observe |

MVCC lets readers access an appropriate row version without necessarily waiting for an active writer. WAL ensures committed changes can be recovered after a crash.

For example, a reporting query can read a consistent version of the orders table while another transaction updates orders. Meanwhile, WAL records the changes needed for recovery.

These mechanisms complement each other, but MVCC alone does not guarantee crash durability, and WAL alone does not determine which version a transaction should see.

## 5. What Causes Table Bloat and VACUUM Issues in PostgreSQL?

PostgreSQL uses MVCC and generally does not immediately remove obsolete row versions after updates and deletes.

For example:

```sql
UPDATE orders
SET status = 'completed'
WHERE id = 101;
```

This normally creates a new tuple version. The previous version eventually becomes dead when no relevant transaction can still need it.

If dead tuples accumulate faster than PostgreSQL can reclaim or reuse their space, **table bloat** can develop.

Bloat increases storage usage and can make queries slower because more pages may need to be read.

### What does VACUUM do?

`VACUUM` identifies dead tuples that are no longer needed by active transactions and makes their space available for reuse. It also performs other maintenance, including helping prevent transaction ID wraparound.

PostgreSQL's autovacuum processes tables automatically based on configured thresholds and activity.

A normal `VACUUM` usually makes space reusable inside the relation but does not generally shrink the file on disk. `VACUUM FULL` rewrites the table and can return space to the operating system, but it requires an exclusive lock and can cause significant downtime for that table.

### Why can VACUUM fall behind?

Common causes include:

- Heavy update and delete workloads.
- Long-running transactions holding old snapshots.
- Sessions left idle inside open transactions.
- Replication slots retaining WAL or, in relevant configurations, preventing removal of old row versions.
- Autovacuum settings that are poorly matched to table size and write volume.
- Insufficient maintenance resources or I/O capacity.

Long-running snapshots are particularly important because PostgreSQL cannot remove row versions that might still be visible to an active transaction.

### How I would investigate bloat

I would check table sizes, dead-tuple estimates, autovacuum activity, long-running transactions, replication-slot retention, and database logs.

Then I would consider:
- Ending or fixing unnecessarily long transactions.
- Tuning autovacuum thresholds and cost settings.
- Improving indexes and query patterns.
- Scheduling targeted maintenance where appropriate.
- Monitoring transaction ID age and WAL retention.

I would not immediately run `VACUUM FULL` on a busy production table because the locking and rewrite costs could be disruptive.

## 6. How Do Deadlocks Form?

A deadlock occurs when two or more transactions wait for locks held by each other, creating a cycle in the wait-for graph.

Consider two bank-account rows, A and B.

Transaction 1:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;
```

Transaction 2 updates the same accounts in the opposite order:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE id = 2;

UPDATE accounts
SET balance = balance + 50
WHERE id = 1;
```

A possible execution is:

1. Transaction 1 locks account A.
2. Transaction 2 locks account B.
3. Transaction 1 waits for account B.
4. Transaction 2 waits for account A.

Neither transaction can proceed, so a deadlock exists.

### How does the database resolve it?

PostgreSQL detects deadlocks after a configurable wait and aborts one of the transactions to break the cycle. InnoDB also detects deadlocks and rolls back a selected victim, generally favoring a transaction that is less expensive to roll back.

The application should handle the resulting error and retry the **entire transaction** when appropriate, with bounded retries and backoff.

The best prevention strategy is to acquire locks in a consistent order. For example, every transfer operation could lock the account with the smaller ID first, then the account with the larger ID.

I would also keep transactions short and avoid unnecessary database calls while holding locks.

## 7. What Do Row and Table Locks Mean in Practice?

A lock controls which concurrent operations may access or modify a resource.

### Row-level locks

A row lock protects a particular row from conflicting operations.

For example:

```sql
SELECT *
FROM orders
WHERE id = 101
FOR UPDATE;
```

In PostgreSQL, `FOR UPDATE` locks the selected rows against conflicting row-level operations until the transaction ends.

Another transaction can often update a different order without waiting. However, if it attempts to update the same order, it may have to wait.

An ordinary PostgreSQL `SELECT` generally does not conflict with this row lock because it can read an MVCC-visible version.

### Table-level locks

Table locks control broader operations on a table. They do not necessarily block all access; the effect depends on the lock mode.

For example, PostgreSQL `ACCESS EXCLUSIVE` locks block ordinary reads as well as writes on the affected table. Many routine operations use weaker lock modes.

A schema change such as some forms of `ALTER TABLE` may require a strong lock, so I would check its locking behavior before applying it to a busy production table.

Table locks are not always bad. They are necessary for coordinating operations that affect the table as a whole, but unnecessarily strong locks can reduce concurrency.

## 8. What Are Gap Locks and Next-Key Locks in InnoDB?

InnoDB uses row-level locking, but under certain isolation levels and query conditions, it can also lock index gaps.

A **gap lock** prevents other transactions from inserting index records into a particular gap between existing index values.

A **next-key lock** combines a record lock with a gap lock, protecting an index record and the gap immediately before it.

For example, imagine an indexed column containing values:

`10, 20, 30`

A transaction performs a locking range query for values between 10 and 30. Under InnoDB's default `REPEATABLE READ` behavior, depending on the query and index access path, the engine may acquire next-key locks that prevent inserts into relevant gaps.

Why does this matter? Without appropriate range locking, another transaction might insert a row that falls into the queried range, potentially affecting repeated range queries or causing a phantom read.

A few important distinctions:

- Gap and next-key locks are primarily associated with InnoDB's locking and range-access behavior, particularly under `REPEATABLE READ`.
- The exact locks depend on the index, predicate, isolation level, and execution plan.
- A unique-index lookup for an existing unique key can often use a record lock without locking an adjacent gap.
- Under `READ COMMITTED`, InnoDB generally reduces gap locking for ordinary searches and index scans, with exceptions such as certain foreign-key and duplicate-key checks.

PostgreSQL does not use InnoDB-style gap and next-key locks for ordinary queries. PostgreSQL's `SERIALIZABLE` isolation instead uses Serializable Snapshot Isolation, including predicate-lock tracking to detect potentially unsafe conflicts. These predicate locks do not necessarily block inserts directly; PostgreSQL may abort a transaction when it detects a serialization anomaly.

## 9. Practical Example: Preventing Overselling

Suppose two customers try to purchase the final item in stock.

A naive implementation reads the stock quantity, checks that it is greater than zero, and then updates it. Two concurrent transactions might both read the same quantity and both attempt to purchase the item.

One solution is a conditional atomic update:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity > 0;
```

The application checks the number of affected rows. If one row was updated, the reservation succeeded; if zero rows were updated, no stock was available or the product did not match the predicate.

This works because the database coordinates concurrent updates and evaluates the update against the current applicable row state. It is often simpler than performing a separate read followed by a write.

For more complex operations, I would use an appropriate transaction and locking strategy, ensure constraints protect critical invariants, and test concurrent requests.

## Final Summary

MVCC allows transactions to read appropriate versions of data while reducing unnecessary reader-writer blocking. WAL provides the durable recovery information needed to restore database consistency after a crash. PostgreSQL VACUUM manages obsolete row versions, while proper transaction design and consistent lock ordering help prevent deadlocks.

For practical development, I focus on understanding isolation levels, keeping transactions short, inspecting slow queries and lock waits, and choosing the simplest concurrency-control strategy that correctly protects the business rule.

*Q25. ORMs vs query builders vs raw SQL. What do lazy and eager loading, the unit of work and change tracking cost you? When does an ORM generate slow or incorrect queries, how do you handle bulk operations and migrations safely, and when is dropping to raw SQL the right call?*

In my projects, I choose between an ORM, a query builder, and raw SQL based on the complexity of the query, maintainability, performance requirements, and how much control I need over database operations.

An ORM improves developer productivity by mapping database records to application objects. A query builder lets me construct SQL programmatically while retaining more control over the generated query. Raw SQL gives me the most direct control over database behavior.

For example, in a .NET application, I might use Entity Framework Core for standard CRUD operations, Dapper for lightweight data access, and raw SQL for complex reports or performance-critical queries.

## 1. ORM vs Query Builder vs Raw SQL

| Feature | ORM | Query builder | Raw SQL |
|---|---|---|---|
| Example | Entity Framework Core, Hibernate | Knex.js, SQL query builders | SQL commands, stored procedures |
| Abstraction | High | Medium | Low |
| Development speed | High for standard CRUD | Good for custom queries | Depends on query complexity |
| Type safety | Often supports typed entities and LINQ | Depends on the library | Depends on tooling and how results are mapped |
| Query control | Generated by the ORM | More explicit query construction | Full control over SQL |
| Change tracking | Commonly supported | Usually not built in | Usually not built in |
| Complex queries | Can become cumbersome | Good control | Often the clearest option |
| Performance predictability | Requires understanding generated SQL | Generally more explicit | High control, but still depends on query design |
| Best fit | Domain entities and business workflows | Flexible application queries | Complex analytics, bulk operations, and tuned SQL |

I would not assume that raw SQL is automatically faster. A well-written ORM query can generate efficient SQL, while poorly written raw SQL can still cause full table scans, excessive locking, or bad execution plans.

## 2. Lazy Loading vs Eager Loading

These strategies determine when related data is retrieved.

### Lazy loading

Lazy loading fetches related data only when the application accesses the relationship.

For example, I load 100 orders and then access each order's customer:

```csharp
var orders = await db.Orders.ToListAsync();

foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name);
}
```

If lazy loading is enabled and the customer navigation property is not already loaded, accessing it can trigger another database query for each order.

This creates the **N+1 query problem**: one query loads the orders, followed by potentially 100 additional queries for their customers.

The result is more database round trips and increased latency.

### Eager loading

Eager loading retrieves related data as part of the initial query or through additional queries executed as part of the loading operation.

In Entity Framework Core:

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

This makes the relationship explicit and avoids the typical per-order lazy-loading pattern.

However, eager loading is not automatically optimal. Including several collection relationships can produce large joined result sets, duplicate parent data, and substantial memory consumption.

In EF Core, `AsSplitQuery()` can help avoid cartesian explosion by retrieving related collections through separate SQL queries, though it introduces additional round trips and can have consistency implications under concurrent changes.

**My approach:** I load the relationships the current use case actually needs. For list screens, I often prefer projection into DTOs instead of loading complete entity graphs.

For example:

```csharp
var orders = await db.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Status = o.Status,
        CustomerName = o.Customer.Name,
        Total = o.Total
    })
    .ToListAsync();
```

This limits the selected columns and returns data shaped for the API response.

## 3. Unit of Work and Change Tracking

Many ORMs implement the Unit of Work pattern through a context or session. In Entity Framework Core, `DbContext` tracks entity changes and coordinates persistence when I call `SaveChangesAsync()`.

For example:

```csharp
var order = await db.Orders.FindAsync(101);

if (order is not null)
{
    order.Status = "Completed";
    await db.SaveChangesAsync();
}
```

The ORM tracks the entity and detects which properties changed. It then generates the necessary SQL statements.

### What does change tracking cost?

Change tracking is convenient, but it consumes CPU and memory. The ORM must maintain entity state, relationship information, and other tracking metadata. Large tracked object graphs can become expensive.

For read-only queries, I can disable tracking:

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.Status == "Pending")
    .ToListAsync();
```

This reduces tracking overhead when I do not need to modify the returned entities through the same context.

For large workloads, I also avoid keeping a single `DbContext` alive for too long. A long-lived context can accumulate tracked entities, retain stale state, and make behavior harder to reason about.

The `DbContext` should generally be scoped to a unit of work, commonly one application operation or request, rather than shared concurrently between requests.

## 4. When Can an ORM Generate Slow or Incorrect Queries?

An ORM does not eliminate the need to understand SQL. Performance problems often come from the query pattern or from assumptions about the generated SQL.

Common problems include:

**N+1 queries:** Lazy loading triggers repeated queries for related entities.

**Over-fetching:** Loading complete entities when the application needs only two or three fields.

**Cartesian explosion:** Joining multiple collection relationships produces many duplicated rows.

**Client-side evaluation or excessive materialization:** Fetching too much data and filtering it in application memory instead of letting the database perform the filtering. EF Core restricts implicit client evaluation in many query contexts, but explicit materialization can still create this problem.

**Inefficient pagination:** Large offset-based pagination can become expensive as the offset grows. Keyset pagination can be more efficient when the query has a suitable index and stable ordering.

**Missing indexes:** The ORM may generate valid SQL that still performs a table scan because the database lacks an appropriate index.

**Unexpected transaction or tracking behavior:** Incorrect context lifetimes, concurrent use of a context, or assumptions about transaction boundaries can lead to bugs.

**Translation differences:** Not every language-level expression translates into the SQL I expect. Some expressions may be unsupported, produce inefficient SQL, or have database-specific null and comparison semantics.

My debugging process is to inspect the generated SQL, measure query duration and row counts, check the execution plan, review indexes, and determine whether the problem is excessive round trips, poor SQL, or unnecessary application-side work.

In EF Core, I can use `ToQueryString()` to inspect a translated query and enable appropriate SQL logging during development. In production, I use query metrics and carefully configured logging rather than indiscriminately logging sensitive parameter values.

## 5. Bulk Operations

Standard ORM operations often work well for small batches, but tracking thousands of entities individually can be expensive.

For example, loading 50,000 records, modifying each entity, and calling `SaveChangesAsync()` can consume significant memory and generate many individual SQL statements, depending on batching and provider behavior.

For set-based updates, EF Core supports `ExecuteUpdateAsync()` and `ExecuteDeleteAsync()`:

```csharp
await db.Orders
    .Where(o => o.Status == "Expired")
    .ExecuteUpdateAsync(setters =>
        setters.SetProperty(
            o => o.Status,
            "Archived"));
```

This performs a database-side update without loading every matching entity into memory.

However, these operations bypass the normal change tracker. Entities already tracked by the context can therefore contain stale values. I account for that by avoiding conflicting tracked state or reloading/clearing it when necessary.

For very large inserts or imports, I might use provider-specific bulk-copy APIs, database-native loading mechanisms, or staging tables.

For critical bulk operations, I also consider:
- Transaction boundaries and batch sizes.
- Lock duration and transaction-log growth.
- Unique constraints and duplicate handling.
- Retry behavior and idempotency.
- Partial failures and resumability.
- Whether the operation should run asynchronously in a background job.

The best approach depends on the database provider and the required consistency guarantees.

## 6. Database Migrations and Schema Safety

ORM migrations allow schema changes to be versioned alongside application code.

In EF Core, I can create a migration from model changes, review the generated SQL, and apply it through a controlled deployment process.

For example:

```bash
dotnet ef migrations add AddOrderStatus
dotnet ef migrations script
```

I would inspect the resulting migration script before production deployment, especially if it drops columns, rebuilds indexes, changes column types, or modifies a large table.

My main practices are:

1. Review generated migrations rather than trusting them blindly.
2. Test migrations against realistic data volumes.
3. Use explicit migration jobs or controlled release steps instead of letting multiple application instances race to apply migrations.
4. Use backward-compatible schema changes when old and new application versions may coexist.
5. Plan long-running backfills separately from schema changes.
6. Maintain verified backups and a recovery strategy.

For a zero-downtime release, I prefer an expand-and-contract approach: add the new schema first, deploy compatible application code, migrate data, and remove obsolete schema only after the old code no longer depends on it.

A database rollback is not always as simple as reverting a Git commit. If a migration destroys data or changes its meaning, the safer recovery may be a forward fix rather than a down migration.

## 7. When Is Raw SQL the Right Choice?

I use raw SQL when it provides a meaningful advantage in clarity, performance, or database-specific functionality.

Examples include:

- Complex reporting queries with multiple joins, common table expressions, or window functions.
- Large set-based updates or deletes.
- Bulk imports and staging-table workflows.
- Queries that require database-specific features not well supported by the ORM.
- Performance-critical queries where the generated SQL is inefficient or difficult to control.

For example, a restaurant dashboard might need daily revenue, order counts, and average order value grouped by date. A carefully written SQL query with aggregation can be clearer and more efficient than loading thousands of order entities and calculating everything in application code.

Even when using raw SQL, I follow parameterization:

```csharp
var orders = await db.Orders
    .FromSqlInterpolated(
        $"SELECT * FROM Orders WHERE Status = {"Pending"}")
    .ToListAsync();
```

With EF Core, `FromSqlInterpolated` parameterizes interpolated values in this example. I never concatenate untrusted input directly into SQL. Dynamic identifiers, such as table names or sort columns, need separate allowlisting because SQL parameters cannot generally represent identifiers.

I also account for the fact that raw SQL may bypass ORM change tracking, automatic relationship loading, or some provider-independent behavior. Those responsibilities need to be handled explicitly.

## 8. My Practical Strategy

In a typical .NET backend, I would use the following combination:

- **Entity Framework Core:** Standard CRUD operations, entity relationships, transactions, and migrations.
- **LINQ projections and no-tracking queries:** Read-heavy API endpoints that return DTOs.
- **Dapper or a query builder:** Explicit SQL queries where I want lightweight mapping and direct control over the query.
- **Raw SQL or bulk APIs:** Complex reports, bulk imports, and operations where set-based database execution is more appropriate.

I would measure performance before introducing extra tools. Using multiple data-access approaches is reasonable when each has a clear purpose, but unnecessary mixing can increase maintenance and testing costs.

## Final Summary

An ORM improves productivity and helps manage entity state, relationships, and schema evolution, but it introduces tracking and abstraction overhead. Query builders provide more explicit query construction, while raw SQL provides maximum control at the cost of greater responsibility.

My decision is based on the workload: use an ORM for maintainable business operations, project only the data needed for read APIs, use set-based or bulk operations for large workloads, and move to raw SQL when measurements or query complexity justify it.

*Q26. Distributed locks and coordination. Why does a naive Redis SETNX lock fail (expiry during work, clock drift, failover)? What are fencing tokens, what does Redlock claim and why is it debated, and when should you use a database advisory lock, a unique constraint or leader election instead of a lock at all?*

In distributed systems, a distributed lock ensures that multiple application instances coordinate access to a shared resource. For example, if I run three instances of an order-processing service, I may want only one instance to process a particular scheduled job at a time.

However, implementing a lock using Redis or a database is not enough by itself. I also need to consider network failures, lock expiry, process pauses, failover, and what happens when an old lock holder continues working after losing ownership.

## 1. Why Is a Naive Redis SETNX Lock Not Enough?

Redis provides commands such as `SET` with `NX` and `PX` options to acquire a lock atomically.

For example:

```redis
SET lock:order:101 unique-token NX PX 10000
```

This attempts to acquire the lock if it does not exist and gives it a 10-second expiration.

The unique token identifies the owner, and the expiry helps prevent a crashed process from holding the lock forever.

However, several problems remain.

### Problem 1: The lock expires while work is running

Suppose Instance A acquires a lock for 10 seconds and starts processing a payment-related job.

If processing takes 15 seconds, the lock expires after 10 seconds. Instance B can now acquire the same lock and begin processing the same job while Instance A is still working.

Both instances may believe they are the legitimate owner.

**Possible solutions:** Choose a suitable lease duration, renew the lease safely while working, and use fencing tokens or idempotent operations when correctness requires protection against stale owners.

Lease renewal alone is not sufficient because the process can pause or lose connectivity and resume after its lease has expired.

### Problem 2: A process can continue after losing ownership

Imagine Instance A acquires a lock and then pauses for a long time because of a garbage-collection pause, operating-system scheduling, or a network issue.

Its lease expires, and Instance B acquires the lock and completes the job.

Instance A then resumes and attempts to write its result.

Redis may correctly show B as the current lock owner, but the downstream database or service may not know that A's ownership is stale.

This is why **a lock does not automatically prevent stale writes**.

### Problem 3: Failover can create conflicting owners

Suppose Redis acknowledges a lock acquisition on the primary, but the lock has not yet replicated to a replica. The primary fails, and the replica is promoted.

The promoted replica may not contain the lock. Instance B can then acquire it even though Instance A still believes it owns the original lock.

Asynchronous replication and failover behavior mean that a basic Redis lock does not necessarily provide strong mutual exclusion across all failure scenarios.

### Problem 4: Unsafe release

A client must not release a lock simply by deleting its key. The key might have expired and been acquired by another client.

The safe pattern is to release the lock only if the stored token still matches the client's unique token, using an atomic Lua script or an equivalent atomic operation.

The same ownership check is necessary when renewing the lease.

## 2. What Are Fencing Tokens?

A fencing token is a monotonically increasing number assigned to each new lock holder.

For example:

- Instance A receives token `41`.
- A's lease expires.
- Instance B acquires the lock and receives token `42`.
- B writes data using token `42`.
- A resumes and attempts to write using token `41`.

The downstream resource rejects A's write because `41` is older than the highest accepted token, `42`.

This protects the resource even when a stale lock holder continues executing.

The critical distinction is that **the downstream resource must enforce the fencing token**. Merely generating a token in Redis does not protect the data if the database or external service ignores it.

The token must come from a source that guarantees the required ordering across acquisitions. For example, a database sequence or another appropriately designed coordination mechanism can provide monotonically increasing values. A simple counter on a failover-prone Redis deployment may not be sufficient if it can roll back or produce duplicate values after failover.

Fencing tokens are especially useful for:
- Distributed workers updating shared records.
- Scheduled jobs that must not overwrite newer results.
- Services controlling external resources that can accept ordered writes.
- Long-running operations where stale processes might resume.

They are not a replacement for idempotency or transaction constraints, but they address a failure mode that a lease alone cannot solve.

## 3. What Does Redlock Do, and Why Is It Debated?

Redlock is an algorithm proposed for acquiring a distributed lock across multiple independent Redis instances.

The basic idea is:

1. Attempt to acquire the same lock with a unique token on multiple independent Redis nodes.
2. Acquire the lock only if a majority of nodes grant it within a bounded time.
3. Calculate the remaining validity period, accounting for acquisition time and clock-drift assumptions.
4. Release partially acquired locks if acquisition fails.
5. Release the lock from all nodes when finished.

Using multiple independent nodes is intended to make the algorithm more resilient than relying on one Redis primary and its asynchronously replicated replica.

### Why is Redlock debated?

The debate concerns the assumptions required to guarantee mutual exclusion in an asynchronous distributed system.

Critics, including Martin Kleppmann, argue that process pauses, network delays, and timing assumptions can cause a client to continue working after its lock is no longer valid. They also argue that fencing tokens are necessary when stale clients could corrupt a protected resource.

The Redlock proposal argues that acquiring a majority of independent Redis nodes, combined with bounded timing assumptions, provides a useful distributed-locking algorithm. The disagreement is about the strength of the guarantees under failures and timing uncertainty.

The practical point is that Redlock does not eliminate the need to consider stale owners, downstream enforcement, and the failure model of the application.

For a non-critical scheduled task, a Redis lease may be sufficient. For operations where concurrent execution could cause financial or irreversible data corruption, I would prefer a design with enforceable database constraints, transactions, or fencing tokens rather than relying on a lock alone.

## 4. When Should I Use a Database Advisory Lock?

A database advisory lock is an application-defined lock managed by the database. It is useful when multiple application instances need to coordinate a task using the same database.

For example, PostgreSQL supports advisory locks that can be acquired at session or transaction scope.

A transaction-scoped example is:

```sql
BEGIN;

SELECT pg_advisory_xact_lock(12345);

-- Perform the coordinated database operation.

COMMIT;
```

The lock is automatically released when the transaction ends.

**When I would use it:**
- Only one worker should perform a particular database maintenance task.
- Multiple application instances need to coordinate a scheduled database job.
- A workflow needs coordination that is naturally tied to a database transaction.

**Trade-offs:**
- The database becomes the coordination point.
- Long-held locks can consume connections or reduce concurrency.
- Every participant must follow the same locking convention.
- Advisory locks do not automatically protect a downstream external service.
- Transaction-scoped advisory locks are often safer than session-scoped locks when connection pooling is involved.

I would use a database advisory lock when the protected work is closely tied to PostgreSQL and keeping coordination in the same system is simpler than introducing Redis.

## 5. When Is a Unique Constraint Better Than a Lock?

In many cases, I do not need a distributed lock at all. I need the database to enforce a business invariant.

Suppose two workers try to create a payment record for the same order. Instead of acquiring a distributed lock first, I can enforce uniqueness:

```sql
CREATE UNIQUE INDEX ux_payments_order_id
ON payments(order_id);
```

Now the database prevents two payment rows for the same order, even if requests arrive concurrently through different application instances.

I can then handle the duplicate-key error or use an appropriate conflict-handling statement.

Other examples include:
- One reservation per unique booking identifier.
- One invoice number per tenant.
- One processing record per idempotency key.
- One active membership for a defined business key, where supported by an appropriate unique or partial index.

**Why I prefer constraints when possible:**

- The invariant is enforced at the database boundary.
- It protects against concurrent requests from any application instance.
- It avoids relying on every caller to acquire the same distributed lock.
- It naturally handles races between competing inserts.

However, a unique constraint only protects the invariant it expresses. It does not automatically make an entire multi-step workflow atomic or guarantee exactly-once execution of an external payment API.

For payments, I would combine an idempotency key or unique payment record with database transactions and careful handling of external payment-provider responses.

## 6. When Should I Use Leader Election Instead?

A distributed lock is typically used to coordinate ownership of a particular resource or operation. Leader election selects one instance to act as the leader for a broader responsibility.

For example, in a system with several worker instances, I might elect one leader to:
- Schedule periodic jobs.
- Coordinate background maintenance.
- Publish configuration updates.
- Manage a partition assignment.

A leader-election system generally uses leases, heartbeats, or a consensus-backed coordination mechanism. If the leader fails, another instance can take over.

However, leader election does not automatically guarantee that the previous leader has stopped working. A paused leader may resume after another leader has been elected.

Therefore, if the leader can modify important shared state, I still need an epoch or fencing token that the protected resource enforces.

For stronger coordination guarantees, systems such as etcd or ZooKeeper provide coordination primitives backed by established distributed coordination protocols. The correct choice depends on the required guarantees and operational complexity.

## 7. Which Approach Would I Choose in Practice?

| Requirement | Preferred approach | Reason |
|---|---|---|
| Prevent duplicate order records | Unique constraint | Enforces the invariant at the database level |
| Prevent duplicate API processing | Idempotency key and database constraint | Makes retries safe |
| Coordinate a PostgreSQL maintenance job | Transaction-scoped advisory lock | Coordination stays in the database |
| Reduce duplicate execution of a non-critical job | Redis lease | Lightweight and relatively simple |
| Protect shared state from stale lock holders | Fencing token enforced by the resource | Rejects older owners |
| Select one scheduler or controller | Leader election | Coordinates a system-wide role |
| Coordinate across failures requiring strong guarantees | Consensus-backed coordination plus resource-level protection | Provides a stronger foundation, but still requires careful application design |

## 8. My General Design Principle

Before introducing a distributed lock, I ask three questions:

1. **What invariant am I protecting?** For example, only one payment record per order.
2. **Can the database enforce it directly?** A unique constraint, conditional update, or transaction is often simpler and safer.
3. **If a lock expires or the owner resumes late, can stale work still damage the system?** If yes, I need resource-level protection, such as fencing tokens, version checks, or idempotent writes.

I also consider lock timeouts, safe release, bounded retries, observability, and what happens when the coordination service is unavailable.

**In summary:** Redis locks are useful for lightweight coordination, but a lease is not proof that a client is still the valid owner. Fencing tokens protect against stale owners when the resource enforces them. Redlock aims to improve Redis lock acquisition resilience but has debated timing assumptions. For business invariants, I prefer database constraints and transactions; for database-specific coordination, advisory locks; and for system-wide ownership, leader election.

*Q27. Event-driven architecture and Kafka fundamentals. Explain topics, partitions, consumer groups, offsets and ordering guarantees (per partition only). How do you choose partition keys, handle consumer lag and rebalances, and what do event sourcing and CQRS add, and cost, compared with a plain CRUD model?*

In my understanding, event-driven architecture allows services to communicate through events instead of requiring every service to call another service synchronously. Apache Kafka is a distributed event-streaming platform that stores events in topics and allows multiple consumers to process them independently.

For example, in a restaurant ERP, when an order is placed, the order service can publish an `OrderCreated` event. Inventory can consume it to update stock, notifications can send an alert, and analytics can update reporting data without the order service directly calling every system.

This reduces tight coupling, but it also introduces eventual consistency, duplicate processing, retries, and operational complexity that need to be handled properly.

## 1. Kafka Topics, Partitions, Consumer Groups, and Offsets

### Topics

A topic is a named stream of records. Examples might be:

- `orders`
- `payments`
- `inventory-events`
- `notifications`

Producers publish records to topics, and consumers read them.

Kafka retains records according to its configured retention policy, so consumers can often replay historical events by resetting their offsets, provided those records still exist.

### Partitions

A topic is divided into partitions, which allow Kafka to distribute data and processing across brokers and consumers.

For example, an `orders` topic might have three partitions:

- Partition 0
- Partition 1
- Partition 2

Each partition is an ordered, append-only log. Kafka can replicate partitions across brokers for fault tolerance.

Adding partitions can increase parallelism, but it does not automatically guarantee better performance if the database, downstream service, or consumer processing is the bottleneck.

### Consumer groups

A consumer group consists of consumers that share the work of reading a topic.

For example, if the `orders` topic has three partitions and a consumer group has three consumers, Kafka can assign one partition to each consumer.

If the group has five consumers but only three partitions, only three consumers can actively consume those partitions at a time. The other two remain idle for that topic assignment.

Different consumer groups can independently read the same events. For instance, the inventory service and notification service can each have their own group, so one group's progress does not advance the other group's offsets.

### Offsets

An offset identifies a record's position within a partition.

Consumers track offsets to know where they are in the log. Committing an offset records the consumer's progress so it can resume after a restart or rebalance.

I would generally commit offsets only after the corresponding processing has succeeded, while carefully handling the relationship between database writes and offset commits.

If I commit before processing finishes, a crash can cause an event to be skipped. If I process the event and crash before committing the offset, Kafka may deliver it again.

This is why duplicate-safe processing is important.

## 2. Kafka Ordering Guarantees

Kafka guarantees record order within a partition, not across an entire topic.

Suppose the events for order `101` are:

1. `OrderCreated`
2. `OrderPaid`
3. `OrderCompleted`

If all three records use the same partition key, Kafka can place them in the same partition and preserve their order in that partition's log.

However, if these events are spread across different partitions, a consumer cannot rely on a global ordering between them.

### Choosing a partition key

I choose a key based on the entity or workflow whose ordering matters.

For a restaurant ERP, `orderId` could be a good key because all events for a particular order can be routed to the same partition.

```text
Key: order-101  → Partition 1
Key: order-102  → Partition 2
Key: order-101  → Partition 1
```

This lets events for the same order maintain partition order while different orders can be processed concurrently.

I also consider key distribution. If most records use one key, that partition can become a hotspot. A key should balance parallelism with the ordering requirement.

Changing the number of partitions can also change the partition selected for a key under common partitioning strategies. If long-term key-to-partition stability is important, I plan partition expansion carefully.

## 3. Handling Consumer Lag

Consumer lag is the difference between how far a consumer group has progressed and how far the partition's log has advanced.

For example, if the latest offset is around `10,000` and a consumer group has processed through `9,200`, its lag is approximately 800 records, subject to the monitoring system's exact offset convention.

Lag can increase because:

- Producers are publishing faster than consumers can process.
- Consumers are blocked by slow database queries or external API calls.
- Consumers are repeatedly retrying failing records.
- There are too few partitions or consumers for the workload.
- Rebalances or infrastructure failures interrupt processing.

### How I would troubleshoot it

First, I would monitor lag by consumer group and partition, along with processing throughput, error rates, processing latency, and rebalance frequency.

Then I would identify the bottleneck:

- Optimize slow database queries and add appropriate indexes.
- Use bounded asynchronous processing where safe.
- Increase consumer instances when there are unassigned partitions.
- Increase partitions if more parallelism is needed and the key distribution supports it.
- Batch database operations when that improves throughput.
- Send repeatedly failing events to a dead-letter topic after a defined retry policy.

I would not blindly increase consumer count. If the topic has six partitions, adding twenty consumers to the same group cannot provide twenty-way partition parallelism for that topic.

## 4. Consumer Rebalances

A rebalance happens when Kafka redistributes partition ownership among consumers in a group. This can occur when consumers join or leave, fail, or when the partition assignment changes.

For example, if a group has three consumers and one crashes, Kafka may assign its partitions to the remaining consumers.

During a rebalance, consumption may pause temporarily. Poorly managed processing can also lead to repeated rebalances and increased lag.

### How I reduce rebalance problems

- Keep consumer processing times within configured limits.
- Avoid blocking the consumer thread for too long.
- Tune polling and session-related settings based on the workload.
- Use an appropriate partition assignment strategy.
- Consider cooperative rebalancing to reduce unnecessary partition movement.
- Stop consumers gracefully and release resources cleanly.
- Ensure processing is safe if an event is delivered more than once.

If I process records asynchronously, I must also preserve the relationship between completed work and committed offsets. I should not commit past records that are still being processed or have failed.

## 5. Delivery Semantics and Duplicate Processing

Kafka applications commonly discuss three delivery models:

**At-most-once:** Records may be lost from the application's processing perspective, but are not intentionally processed more than once. For example, committing before processing risks losing work if the consumer crashes.

**At-least-once:** Records are retried until processing succeeds, but duplicate processing is possible. This is a common practical choice.

**Exactly-once:** Kafka supports transactional processing for certain Kafka-to-Kafka workflows, including atomically writing output records and committing consumed offsets within a Kafka transaction.

However, exactly-once processing does not automatically extend to arbitrary external databases, email services, or payment providers.

For a consumer that updates MySQL, I would typically use a database transaction and an idempotency mechanism. For example, I could store each processed event ID in a table with a unique constraint and apply the business update in the same transaction. If the same event arrives again, the duplicate can be detected.

For external side effects, I would use the provider's idempotency support where available and design retry behavior carefully.

## 6. Reliable Event Publishing: The Dual-Write Problem

Consider an order service that performs two actions:

1. Saves an order to the database.
2. Publishes an `OrderCreated` event to Kafka.

If the database commit succeeds but publishing fails, the order exists but downstream services never receive the event.

If the event is published first and the database transaction fails, consumers may process an order that was never committed.

This is the **dual-write problem**.

A common solution is the transactional outbox pattern:

1. Save the order and an outbox event in the same database transaction.
2. A background publisher reads unpublished outbox records and publishes them to Kafka.
3. Mark records as published after successful delivery.
4. Make consumers idempotent because publishing can still be repeated after failures.

This avoids relying on a single distributed transaction spanning the database and Kafka.

## 7. What Is Event Sourcing?

In a conventional CRUD model, the database stores the current state of an entity. For example, an order row may contain its current status, total, and customer ID.

In event sourcing, the system stores a sequence of domain events as the authoritative history of changes.

An order might have events such as:

```text
OrderCreated
ItemAdded
PaymentReceived
OrderCompleted
```

The current order state can be reconstructed by applying those events in sequence. Systems may also maintain snapshots to avoid replaying the entire history every time.

### What event sourcing adds

- A detailed audit trail of business changes.
- The ability to replay events and rebuild derived state.
- Historical analysis of how an entity changed.
- The ability to build new projections from retained events.

### What it costs

- More complex event schemas and versioning.
- Event replay and snapshot management.
- More complicated debugging and operational procedures.
- Eventual consistency between projections.
- Difficulties with correcting historical events and handling sensitive data.
- Greater care required when events are duplicated or arrive out of order.

Event sourcing is not the same as using Kafka. Kafka can carry events from a conventional CRUD system without the application storing its complete business history as an event-sourced model. Kafka retention alone should not be treated as a permanent event store unless its durability and retention configuration explicitly support that requirement.

## 8. What Is CQRS?

CQRS stands for Command Query Responsibility Segregation. It separates the model used to change application state from the model used to read it.

For example, in a restaurant ERP:

- **Command model:** Handles operations such as creating an order, applying a discount, or completing a payment.
- **Query model:** Serves dashboard summaries, sales reports, and order-history screens.

The command side enforces business rules. Events or other propagation mechanisms can update a separate read model optimized for queries.

CQRS can be implemented without event sourcing, and event sourcing can be implemented without a full CQRS architecture.

### Benefits

- Read models can be optimized for specific screens and reports.
- Read traffic can scale separately from write traffic.
- Complex reporting queries need not burden the transactional model as much.
- Different query models can be built for different consumer needs.

### Costs

- Data can be temporarily stale because read models update asynchronously.
- More services, data flows, and monitoring are required.
- Rebuilding projections and handling failed updates become operational responsibilities.
- Cross-model consistency and debugging become more complicated.

## 9. Event Sourcing and CQRS vs. Plain CRUD

| Aspect | Plain CRUD | CQRS | Event sourcing |
|---|---|---|---|
| Source of truth | Current database state | Usually the command-side state | Event history |
| Complexity | Lowest | Higher | Higher |
| Audit history | Usually added separately | Depends on implementation | Built into the event model |
| Read performance | Often sufficient | Separate read models possible | Projections can optimize reads |
| Historical replay | Usually requires audit data | Depends on event storage | Core capability |
| Consistency | Often straightforward within one database | Read models may be eventually consistent | Projections may be eventually consistent |
| Best fit | Most business applications | Distinct read/write requirements | Strong historical and audit requirements |

I would start with CRUD unless the business requirements justify a more complex model. CQRS is useful when read and write workloads or models genuinely differ. Event sourcing is valuable when the history of business decisions is itself a first-class requirement, not merely because Kafka is available.

## 10. How I Would Use Kafka in One System

For a restaurant ERP, I might design the flow as follows:

1. The order API validates the request and saves the order.
2. The same database transaction saves an outbox event.
3. A publisher sends `OrderCreated` to Kafka using `orderId` as the key.
4. The inventory consumer processes the event and updates stock.
5. The notification consumer sends a confirmation.
6. The analytics consumer updates a sales projection.
7. Each consumer monitors failures, retries transient errors, and handles duplicate events safely.

The order creation response does not need to wait for email or analytics processing. However, inventory updates may be eventually consistent, so I would define explicitly whether stock must be reserved synchronously before confirming the order or whether an asynchronous reservation workflow is acceptable.

For business-critical operations, I would use database constraints and transactions to protect invariants rather than assuming Kafka ordering alone guarantees correctness.

## Final Summary

Kafka distributes event streams using topics and partitions. Consumer groups divide partition work, offsets track progress, and ordering is guaranteed within each partition rather than globally. Partition keys determine both ordering boundaries and workload distribution.

In production, I focus on consumer lag, rebalances, duplicate-safe processing, retry policies, dead-letter handling, and reliable event publishing through patterns such as the transactional outbox.

Event sourcing and CQRS provide auditability and specialized read models, but add operational and consistency costs. I would introduce them only when the system's business and scaling requirements justify those trade-offs.

*Q28. Webhooks, both directions. How do you design a reliable webhook sender (signed payloads, retries with backoff, delivery logs, disabling failing endpoints)? How do you safely receive one (signature verification, replay protection, responding fast and processing async, idempotency on event IDs, out-of-order events)?*

In my applications, I treat webhooks as an asynchronous communication mechanism between systems. I design both the sender and receiver to handle network failures, duplicate deliveries, security threats, and events arriving out of order.

### 1. Designing a reliable webhook sender

When my application needs to notify another system about an event, such as an order being completed or a payment succeeding, I follow these practices:

- **Signed payloads:** I sign the request body using HMAC-SHA256 with a shared secret. I include a signature header, such as `X-Webhook-Signature`, and a timestamp header. The receiver uses the same secret to verify that the payload has not been tampered with.
- **Asynchronous delivery:** I save the event and enqueue a delivery job instead of making the external HTTP request inside the main business transaction. A transactional outbox helps ensure that the event is not lost if the database commits but queue publishing fails.
- **Retries with exponential backoff:** If delivery fails because of a timeout, network error, or retryable HTTP status such as `429` or `5xx`, I retry with increasing delays and jitter. I respect `Retry-After` when appropriate. I avoid endlessly retrying permanent client errors such as `400` or `404`.
- **Delivery logs:** I store the event ID, endpoint, attempt count, timestamps, response status, latency, and a sanitized response summary. I avoid logging secrets or sensitive payload data.
- **Endpoint failure management:** After repeated failures, I apply a configurable retry limit, alert the endpoint owner, and potentially disable the endpoint or move deliveries to a dead-letter queue. I provide manual replay for failed deliveries.

Each delivery has a stable event ID, and every retry uses that same ID. This allows the receiver to recognize duplicate deliveries.

### 2. Safely receiving a webhook

On the receiving side, I assume that requests can be duplicated, delayed, forged, or delivered in the wrong order.

**My processing flow is:**

1. **Verify the signature:** I read the exact raw request body and calculate the expected HMAC using the endpoint's secret. I compare signatures using a constant-time comparison. Parsing and reserializing JSON before verification can change the bytes and cause incorrect verification.
2. **Validate the timestamp:** I reject requests whose signed timestamp falls outside an acceptable time window. This limits replay attacks, although it does not replace event-level deduplication.
3. **Check the event ID:** I maintain a database table with a unique constraint on the provider's event ID. This prevents the same event from being processed twice, even when the provider retries it.
4. **Respond quickly:** After authentication and durable acceptance, I store the event in a database or queue and return a success response, commonly `200` or `202`. I process the actual business logic asynchronously rather than keeping the HTTP request open.
5. **Process idempotently:** I make the business operation safe to repeat. For example, processing the same payment-success event twice must not create two orders or credit the same account twice.

An important detail is that I acknowledge the webhook only after the event has been durably accepted. Returning success before storing or enqueueing it could result in data loss if the application crashes.

### 3. Handling duplicate and out-of-order events

Webhook providers generally offer **at-least-once delivery**, not guaranteed exactly-once processing. Therefore, I design for duplicates.

For example, if I receive `payment.succeeded` twice, the unique event ID and idempotent business logic prevent the payment from being applied twice.

Events can also arrive out of order. For example, `order.shipped` might arrive before a delayed `order.confirmed` event. To handle this, I use one or more of the following approaches:

- Include an entity version or sequence number and ignore stale updates.
- Enforce valid state transitions in the database.
- Fetch the latest resource state from the provider when the event sequence is ambiguous.
- Buffer events when strict ordering is required.

I do not rely solely on timestamps for ordering because clocks can differ and timestamps may not represent the actual event sequence.

### 4. Example architecture

**Webhook sender:**

`Business transaction → Database + Outbox → Worker → Signed HTTP request → Retry and delivery log`

**Webhook receiver:**

`HTTP endpoint → Verify signature and timestamp → Deduplicate event ID → Persist or enqueue → Return 2xx → Async idempotent processing`

### 5. How I would implement this in a real application

For a React frontend with a .NET API and MySQL or PostgreSQL, I would implement the sender using a background worker and an outbox table, with retry metadata and a delivery-attempt table. On the receiving side, I would verify HMAC signatures in ASP.NET Core middleware or a dedicated service, store incoming event IDs under a unique database constraint, and process accepted events through a background worker.

I would also monitor delivery success rate, retry count, queue lag, processing failures, and dead-letter events.

**The key principle is that webhooks are not guaranteed one-time HTTP callbacks. They are distributed-system messages, so reliability depends on durable delivery, authentication, retries, idempotency, and explicit handling of event ordering.**

*Q29. Search and full-text indexing. Why do LIKE '%term%' queries not scale? How does an inverted index work, what do analyzers, tokenization and relevance scoring do, and how do you keep Elasticsearch/OpenSearch in sync with the primary database (CDC, outbox, dual writes and their risks)? When is Postgres full-text search enough?*

**Answer:**

In my applications, I choose a search strategy based on the type of search, data volume, and performance requirements. For simple filtering, I use database indexes. For complex text search, relevance ranking, typo tolerance, and faceted search, I consider PostgreSQL full-text search or Elasticsearch/OpenSearch.

### 1. Why does `LIKE '%term%'` not scale?

A query such as:

```sql
SELECT *
FROM products
WHERE name LIKE '%laptop%';
```

often requires scanning a large number of rows because the pattern starts with a wildcard. A conventional B-tree index generally cannot efficiently locate an arbitrary substring when the beginning of the value is unknown.

As the table grows, the database may need to inspect many rows and test each value against the pattern. This increases CPU and I/O costs.

The solution depends on the search requirement:

- **Exact match or prefix search:** A B-tree index can help, depending on the query and collation.
- **Arbitrary substring search:** PostgreSQL's `pg_trgm` extension with a GIN or GiST index can help.
- **Natural-language search:** PostgreSQL full-text search or a dedicated search engine is usually more appropriate.

I would not automatically introduce Elasticsearch just because a table contains many rows. I first examine the query plan, data volume, latency requirements, and search features.

### 2. How does an inverted index work?

A relational index commonly helps locate rows based on indexed column values. A search engine instead uses an **inverted index**, which maps terms to the documents containing them.

For example, suppose we have these documents:

- Document 1: `blue gaming laptop`
- Document 2: `gaming keyboard`
- Document 3: `blue laptop stand`

The inverted index might look like this:

| Term | Documents |
|---|---|
| `blue` | 1, 3 |
| `gaming` | 1, 2 |
| `laptop` | 1, 3 |
| `keyboard` | 2 |
| `stand` | 3 |

When a user searches for `blue laptop`, the search engine retrieves the posting lists for `blue` and `laptop`, combines them according to the query, and returns matching documents.

The index can also store information such as term frequency, term positions, and field information. This supports phrase searches, relevance scoring, and queries restricted to particular fields.

This is why an inverted index is effective for text search: it avoids scanning every document for every query. However, it consumes additional storage and must be updated as documents change.

### 3. What do analyzers and tokenization do?

Before indexing text, Elasticsearch/OpenSearch typically passes it through an analyzer.

An analyzer commonly consists of three stages:

1. **Character filters:** Normalize or transform the input before tokenization.
2. **Tokenizer:** Split the text into tokens.
3. **Token filters:** Transform tokens, remove stop words, or apply stemming and other normalization.

For example, the text:

`The Running Shoes — BLUE`

might be analyzed into:

`run`, `shoe`, `blue`

The exact result depends on the analyzer configuration. Lowercasing normalizes capitalization, stemming can reduce related word forms to a common stem, and stop-word removal can discard common words such as `the`.

I configure analyzers according to the language and search requirements. Stemming is useful for some natural-language searches, but it can produce undesirable matches for product names, identifiers, or domain-specific terms.

For identifiers such as `SKU-12345`, I may use a keyword field or a specialized analyzer instead of treating the identifier as ordinary prose.

**Important:** Index-time and search-time analysis should be compatible. Changing an analyzer that affects existing indexed terms may require rebuilding the index.

### 4. How does relevance scoring work?

Search engines usually rank matching documents by relevance rather than returning results in arbitrary order.

Elasticsearch/OpenSearch commonly uses **BM25**, which considers factors including:

- How frequently a term appears in a document.
- How rare the term is across the document collection.
- The document's length relative to other documents.
- Query matching across different fields.

For example, if someone searches for `wireless gaming mouse`, a document whose title contains those terms may rank higher than a long description that mentions only one term.

I can improve relevance by:

- Giving title matches more weight than description matches.
- Using exact-match boosts for product names or identifiers.
- Applying filters for category, availability, or price.
- Supporting phrase matches and synonyms where appropriate.
- Evaluating results against representative queries and user behavior.

Filters should be used for constraints such as category or status, while scoring queries determine relevance. This separation also makes the search behavior easier to reason about.

### 5. How do I keep Elasticsearch/OpenSearch synchronized with the primary database?

I generally treat the relational database as the **source of truth** and the search index as a derived, eventually consistent read model.

The challenge is that updating the database and updating the search index are two separate operations. They cannot normally be committed as one atomic transaction.

There are three common approaches.

**A. Dual writes**

The application writes to the database and then directly updates Elasticsearch/OpenSearch.

```text
Application → Primary database
           → Search index
```

This is easy to implement initially, but it has a consistency problem. The database write might succeed while the search-index update fails, leaving the two systems inconsistent. Reversing the order introduces a different failure mode.

Retries, reconciliation jobs, and idempotent updates can help, but they do not make two independent writes atomic.

I avoid relying on dual writes as the only synchronization mechanism when missing updates would be costly.

**B. Transactional outbox**

Within one database transaction, the application updates the business record and inserts an outbox event.

```text
Database transaction
  ├── Update product
  └── Insert ProductUpdated event

Outbox worker → Elasticsearch/OpenSearch
```

Because both database operations commit together, the event is not lost simply because the application crashes after committing the product update.

A background worker reads pending events, updates the search index, and marks or records successful processing. If the worker crashes after indexing but before recording success, it may deliver the event again, so indexing operations must be idempotent.

For example, using a stable document ID and replacing the document with the latest state makes repeated updates safer.

I would also monitor retry counts, processing lag, failed events, and dead-letter events.

**C. Change Data Capture (CDC)**

CDC captures committed database changes from the database's change log or transaction log and streams them to downstream consumers.

For example:

```text
PostgreSQL WAL / MySQL binlog
          ↓
       CDC tool
          ↓
   Event stream or connector
          ↓
 Elasticsearch/OpenSearch
```

CDC is useful when several downstream systems need database changes or when polling an outbox is undesirable. It can capture inserts, updates, and deletes without requiring every application code path to publish an event.

However, CDC adds operational complexity. I need to handle replication lag, connector failures, schema changes, deletes, duplicate delivery, and event ordering. A CDC pipeline also needs a reliable recovery strategy if the consumer falls behind or loses its checkpoint.

### 6. Handling stale updates and deletes

Regardless of the synchronization approach, I plan for duplicate and out-of-order events.

Suppose a product is updated twice:

- Version 10 changes the price to ₹900.
- Version 11 changes the price to ₹850.

If version 10 arrives after version 11, blindly applying both events can restore the stale price.

I can prevent this by using entity versions or sequence numbers and rejecting stale updates. For high-throughput systems, I may partition events by entity ID to preserve per-entity ordering, while still enforcing version checks in the consumer.

Deletes require explicit handling too. A deleted database record must eventually be removed from the search index, often through a delete event or a tombstone. Periodic reconciliation and full reindexing procedures help detect drift that incremental processing misses.

For large reindexing jobs, I can build a new index, populate it from a consistent database snapshot, catch up on changes that occurred during the rebuild, and then switch an alias to the new index.

### 7. When is PostgreSQL full-text search enough?

I do not automatically introduce Elasticsearch/OpenSearch. PostgreSQL full-text search is often sufficient when the application already uses PostgreSQL and needs conventional text search.

PostgreSQL provides:

- `tsvector` to represent searchable document terms.
- `tsquery` to represent search expressions.
- `to_tsvector()` and `to_tsquery()` or `websearch_to_tsquery()` for preparing search queries.
- GIN indexes for efficient full-text matching.
- Ranking functions such as `ts_rank()` and `ts_rank_cd()`.

For example:

```sql
CREATE INDEX idx_products_search
ON products
USING GIN (
    to_tsvector('english', name || ' ' || description)
);

SELECT id, name
FROM products
WHERE to_tsvector('english', name || ' ' || description)
      @@ websearch_to_tsquery('english', 'wireless gaming mouse');
```

For a production implementation, I would usually maintain a dedicated `tsvector` column, often through a generated column or controlled update mechanism, rather than recomputing the document expression inconsistently across queries and indexes.

PostgreSQL is a good fit when search is primarily linguistic full-text matching, the dataset and query load are manageable, and keeping the data in one system is valuable.

I would consider Elasticsearch/OpenSearch when I need more advanced relevance tuning, typo tolerance, sophisticated autocomplete, complex faceted navigation, specialized analyzers, or independently scalable search workloads. These capabilities depend on configuration and are not all exclusive to a dedicated search engine.

### 8. How I would choose in a real application

For a product catalog or restaurant ERP:

- I would use B-tree indexes for exact filters, identifiers, and common sorting.
- I would use PostgreSQL full-text search for straightforward keyword searches across product or menu descriptions.
- I would consider `pg_trgm` for substring matching and similarity-based search.
- I would introduce Elasticsearch/OpenSearch if advanced relevance, typo-tolerant autocomplete, facets, or search-specific scaling justified the added infrastructure.
- If I introduced a separate search index, I would prefer a transactional outbox or CDC pipeline, with idempotent consumers, version-aware updates, monitoring, and reconciliation.

**The key principle is that the search index is a derived representation of the primary data. The search technology determines how efficiently and intelligently users can find information, while the synchronization design determines whether those results remain trustworthy.**

*Q30. Time, scheduling and distributed cron. Why store timestamps in UTC and convert at the edge? How do you handle time zones, DST, clock skew and leap-second-style surprises, and run a scheduled job exactly once across multiple instances (job locking, a scheduler table, a queue with delayed messages)?*

In backend systems, I treat time handling and scheduling as reliability concerns, not just date-formatting problems. Time zones, daylight saving time, clock differences, and multiple application instances can all cause bugs if the system does not handle them explicitly.

### 1. Why store timestamps in UTC?

I store timestamps representing actual moments in time in UTC and convert them to the user's local time zone at the presentation layer.

For example, if a user in India creates an order at 8:00 PM IST, I can store the corresponding UTC timestamp, `14:30 UTC`. A user in New York can then view that same event in their own local time zone.

This provides several benefits:

- **Consistency:** All servers and services use a common time reference.
- **Simpler comparisons:** Sorting, calculating durations, and comparing timestamps do not require converting between local time zones.
- **Distributed systems:** Logs and events from servers in different regions are easier to correlate.
- **Portability:** Changing a user's time zone does not change when an existing event actually occurred.

I typically use timezone-aware UTC timestamps in application code and an appropriate database timestamp type, such as PostgreSQL `TIMESTAMPTZ`. In MySQL, I pay attention to the session time zone and the conversion behavior of `TIMESTAMP` versus `DATETIME`.

An important distinction is that **UTC timestamps are for recording actual moments, while time zones are necessary for interpreting local schedules.**

### 2. Handling time zones and daylight saving time (DST)

A time zone is not just a fixed UTC offset. It contains rules that can change over time because of daylight saving transitions or government decisions.

For example, a scheduled task intended to run at 9:00 AM in New York should generally remain at 9:00 AM local time, even when the corresponding UTC time changes during the year.

For that reason, I store:
- The scheduled local time, such as `09:00`.
- The IANA time zone, such as `America/New_York`.
- The recurrence rule, such as every weekday.
- The policy for ambiguous or nonexistent local times.

I use a time-zone-aware library, such as .NET's `TimeZoneInfo` or Noda Time, instead of manually adding or subtracting hours.

DST introduces two important edge cases:

- **Nonexistent time:** When clocks move forward, a local time such as 2:30 AM may not exist on that date.
- **Ambiguous time:** When clocks move backward, 1:30 AM may occur twice.

For recurring jobs, I define a clear policy: skip a nonexistent time, shift it forward, or apply another business-approved rule. For an ambiguous time, I choose whether to run at the earlier occurrence, the later occurrence, or both.

The correct policy depends on the business requirement. For example, a daily report might skip a nonexistent time, while a financial deadline may require an explicitly defined interpretation.

### 3. Clock skew and leap-second surprises

Even when every service uses UTC, their system clocks may differ slightly.

**Clock skew** can cause problems with token expiration, distributed locks, event ordering, and scheduled job execution. I use NTP or an equivalent time-synchronization service, monitor clock offsets, and avoid assuming that timestamps from different machines establish a perfectly reliable event order.

For ordering events, I prefer database sequence numbers, entity versions, or broker offsets where appropriate, rather than relying exclusively on wall-clock timestamps.

I also distinguish between two kinds of time:

- **Wall-clock time:** Used for calendar dates, deadlines, and scheduled events.
- **Monotonic time:** Used to measure elapsed durations and timeouts without being affected by wall-clock adjustments.

Leap seconds are another edge case. Operating systems and runtimes may handle them through time smearing, adjustment, or other implementation-specific behavior. I avoid assuming every system represents leap seconds identically. For most business applications, I use UTC timestamps and standard runtime abstractions; for systems requiring precise physical-time accuracy, I follow the relevant time-scale and infrastructure requirements explicitly.

### 4. Running scheduled jobs across multiple instances

Suppose my .NET API runs on three instances, and each instance has a background service that checks whether a daily report is due.

Without coordination, all three instances might execute the same job.

There are several ways to prevent this.

**A. Distributed job locking**

I can use a distributed lock backed by a shared database or Redis so only one worker attempts to run a particular job at a time.

The lock needs an expiration or lease to recover if a worker crashes. However, a lease can expire while a slow worker is still running, allowing another worker to start the same job. I use renewal, appropriate timeouts, and fencing tokens where needed.

I do not treat a distributed lock as a complete guarantee of exactly-once execution. The underlying business operation still needs protection against duplicate execution.

**B. Scheduler table in the database**

For many business applications, I prefer a database-backed scheduler table because it provides durable job state and makes execution auditable.

For example, a table might contain:

- `job_id`
- `scheduled_at`
- `status`
- `locked_until`
- `attempt_count`
- `completed_at`

Workers can atomically claim a due job using a transaction and row-level locking. In PostgreSQL, `FOR UPDATE SKIP LOCKED` is useful for allowing multiple workers to claim different jobs without waiting on rows already claimed by other workers.

A worker marks the job as running, executes it, and records the outcome. If the worker crashes, a lease timeout allows another worker to recover the job.

I also use a unique constraint on the logical job identity, such as `(job_type, scheduled_at)`, so multiple scheduler instances cannot create duplicate records for the same scheduled occurrence.

**C. Queue with delayed messages**

For larger systems, I can publish jobs to a queue that supports delayed delivery or scheduled messages.

The queue distributes work across consumers and handles redelivery, retries, and dead-letter processing. This is useful when job execution is independent of the scheduler itself.

However, queues generally provide at-least-once delivery, so a message may be delivered more than once. I still need idempotent handlers and durable job state.

### 5. Can a distributed job run exactly once?

In a practical distributed system, I do not assume that a scheduler, lock, or queue can guarantee exactly-once execution under every failure condition.

Consider a payment reconciliation job:

1. A worker claims the job.
2. It calls an external payment provider.
3. The provider completes the operation.
4. The worker crashes before recording success.
5. Another worker retries the job.

The system cannot necessarily determine from the timeout alone whether the external operation succeeded.

My approach is to combine:
- Durable job state.
- Atomic claiming or distributed coordination.
- Idempotency keys for external operations where supported.
- Unique database constraints for duplicate-sensitive writes.
- Retry policies and recovery for abandoned jobs.
- Auditing and reconciliation for uncertain outcomes.

If the business operation and its completion record can be committed in the same database transaction, I can make those database effects atomic. For external side effects, I use provider-supported idempotency or a reconciliation workflow.

Therefore, I distinguish **exactly-once business effects** from exactly-once code execution. The latter is generally not something I can guarantee across crashes and network failures.

### 6. How I would implement this in a .NET application

For a production .NET API backed by PostgreSQL or MySQL, I would:

1. Store event timestamps in UTC and use IANA time-zone identifiers for recurring local schedules.
2. Calculate each scheduled occurrence with a time-zone-aware library.
3. Create a durable job record with a unique identity for each scheduled occurrence.
4. Let workers atomically claim due jobs using database coordination or a reliable queue.
5. Use leases and recovery for jobs abandoned by crashed workers.
6. Make handlers idempotent and record attempt counts, outcomes, and error details.
7. Monitor scheduling delays, job duration, failures, retries, and clock synchronization.
8. Test DST transitions, duplicate delivery, worker crashes, clock adjustments, and missed scheduling intervals.

For simpler workloads, a mature scheduler such as Hangfire can provide persistent jobs, recurring schedules, retries, and operational visibility. I would still verify its deployment and storage configuration and design business handlers to tolerate retries.

**The key principle is to use UTC for recording moments, explicit time-zone rules for calendar schedules, and durable coordination plus idempotency for distributed jobs. Locks help coordinate workers, but correctness ultimately depends on how the business operation handles retries and failures.**

*Q31. Layered and clean architecture. Compare controller/service/repository layering with hexagonal and clean architecture. How do dependency injection and dependency inversion keep business logic testable, where do validation and transactions belong, and when does extra abstraction become over-engineering?*

In my applications, I use architecture to separate business rules from infrastructure concerns so that the code remains maintainable, testable, and easier to change. I distinguish between traditional layered architecture, hexagonal architecture, and clean architecture, but I don't treat them as mutually exclusive. They share the same fundamental goal: controlling dependencies and separating responsibilities.

### 1. Controller / Service / Repository layering

This is a common architecture in ASP.NET Core and other web applications.

```text
HTTP Request
     ↓
Controller
     ↓
Service / Business Logic
     ↓
Repository
     ↓
Database
```

Each layer has a specific responsibility:

- **Controller:** Handles HTTP requests, model binding, authentication context, and HTTP responses. It should not contain complex business logic.
- **Service:** Coordinates use cases, enforces business rules, and manages the application's workflow.
- **Repository:** Encapsulates data access, queries, and persistence details.
- **Database:** Stores the application's persistent data.

For example, in a restaurant ERP, an `OrderController` receives a request to create an order. An `OrderService` validates business rules, checks menu-item availability, and coordinates order creation. A repository or data-access component persists the order.

The main advantage is simplicity. However, layering alone does not guarantee clean separation. If the service directly depends on a concrete database implementation, business logic can still become tightly coupled to infrastructure.

### 2. Hexagonal architecture

Hexagonal architecture, also called **ports and adapters**, puts the application and its business capabilities at the center.

```text
 HTTP Controller ──┐
                   ↓
              Input Port
                   ↓
          Application Use Case
                   ↓
              Output Port
                   ↓
       Database / External API
```

The key concepts are:

- **Ports:** Interfaces defining how the application interacts with the outside world.
- **Adapters:** Concrete implementations connecting those interfaces to technologies such as HTTP, SQL databases, message queues, or external APIs.

For example, an application use case might depend on an `IOrderRepository` interface. A PostgreSQL implementation provides the actual persistence, while an in-memory implementation can be used in tests.

The application does not need to know which database implements the port.

Hexagonal architecture is particularly useful when an application has multiple external integrations, complex business workflows, or a need to replace infrastructure without rewriting core logic.

### 3. Clean architecture

Clean architecture organizes the code into dependency-controlled layers, often represented as concentric circles.

A common arrangement is:

```text
Presentation / Infrastructure
             ↓
      Application Use Cases
             ↓
        Domain Model
```

The essential rule is the **Dependency Rule**: source-code dependencies point inward toward business policy, not outward toward infrastructure.

Typical responsibilities include:

- **Domain:** Business entities, value objects, domain rules, and domain services.
- **Application:** Use cases, orchestration, application-level validation, and interfaces required by those use cases.
- **Infrastructure:** Database implementations, external API clients, messaging, and other technical details.
- **Presentation:** Controllers, HTTP models, and response formatting.

For example, an `Order` domain entity can enforce that an order cannot transition from `Cancelled` to `Shipped`. That rule should not depend on ASP.NET Core, Entity Framework Core, or a particular database.

Clean architecture often uses dependency injection to connect concrete infrastructure implementations to interfaces defined closer to the application core.

The difference from hexagonal architecture is primarily emphasis: hexagonal architecture focuses on ports and adapters, while clean architecture describes a broader dependency-oriented organization of the system.

### 4. Dependency injection versus dependency inversion

These concepts are related, but they are not the same.

**Dependency inversion principle (DIP)** is a design principle: high-level business policy should not depend directly on low-level infrastructure details. Both should depend on abstractions where appropriate.

**Dependency injection (DI)** is a technique for supplying an object with its dependencies instead of having it construct them itself.

For example:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken);
}

public sealed class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

The service depends on an interface rather than directly constructing a SQL repository.

ASP.NET Core can register the implementation:

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<OrderService>();
```

This improves testability because a unit test can inject a fake or mock repository. The test can exercise business rules without connecting to a real database.

However, adding an interface to every class does not automatically improve the architecture. I introduce abstractions where they provide a meaningful boundary, enable substitution, or isolate a volatile dependency.

### 5. Where should validation belong?

I separate validation into different categories because not all validation serves the same purpose.

**Request validation:** Checks the incoming API contract, such as required fields, valid formats, and numeric ranges. This can be handled through request DTO validation, FluentValidation, or ASP.NET Core validation mechanisms.

**Application-level validation:** Checks whether a use case has the information and permissions needed to proceed. For example, an order request must reference an existing customer and valid menu items.

**Domain invariants:** Protect business rules that must always hold, regardless of whether the operation came from an HTTP request, a background job, or a message consumer. For example, an order cannot contain a negative quantity.

**Database constraints:** Enforce invariants that must hold even under concurrency, such as unique email addresses, foreign keys, and non-null columns.

These layers complement each other. A controller-level check cannot replace a database uniqueness constraint because two concurrent requests could both pass the check before either inserts the record.

### 6. Where should transactions belong?

Transactions should normally align with the **application use case and its consistency boundary**, rather than being opened indiscriminately in every controller or repository method.

For example, creating an order might require inserting an order record and several order-item records. If these writes must succeed or fail together, they should be committed in one database transaction.

The application service or use-case handler can coordinate the operation through a unit-of-work abstraction, or through a transaction mechanism supplied by the persistence layer.

I avoid opening a database transaction in a controller because that couples HTTP handling to persistence concerns. I also avoid letting every repository method independently commit when several operations must be atomic.

For operations involving external systems, such as sending a WhatsApp notification after creating an order, I do not assume a database transaction can roll back the external side effect. I typically commit the order and an outbox event in the same database transaction, then deliver the notification asynchronously.

Transactions should be kept short, with appropriate isolation and concurrency handling.

### 7. How does this architecture improve testing?

I test each level according to its responsibility.

- **Unit tests:** Test domain rules and application use cases using fakes or mocks for external dependencies.
- **Integration tests:** Verify repository queries, transaction behavior, database constraints, and infrastructure integrations against real or appropriately isolated dependencies.
- **API tests:** Verify routing, authentication, request validation, HTTP status codes, serialization, and the full request pipeline.

For example, I can unit-test that an order cannot be placed when a menu item is unavailable without starting the web server or connecting to PostgreSQL. I then use an integration test to verify that the actual repository correctly persists the order and its items.

I do not try to mock every dependency in every test. Mock-heavy tests can become coupled to implementation details and may pass even when real database behavior is incorrect.

### 8. When does abstraction become over-engineering?

Clean architecture is not a requirement to create an interface, factory, repository, mapper, and separate project for every class.

Extra abstraction becomes counterproductive when it adds indirection without protecting a meaningful boundary.

Common examples include:

- Creating a generic repository over a capable ORM without a real need.
- Building interfaces for classes that have no meaningful alternative implementation or testing boundary.
- Splitting a small CRUD feature across numerous layers with almost no business logic.
- Creating separate DTOs and mappings for every internal operation without a clear reason.
- Building elaborate domain models for simple data-management workflows.

For a small CRUD application, a controller, a focused application service, and straightforward persistence may be sufficient. For a large ERP with order processing, inventory deduction, payments, permissions, and external integrations, stronger boundaries become more valuable.

I choose the architecture based on business complexity, change frequency, consistency requirements, team size, and testing needs. I can introduce more structure as those requirements grow rather than implementing every possible abstraction upfront.

### 9. How I apply this in a .NET application

For a restaurant ERP built with React, ASP.NET Core, and PostgreSQL or MySQL, I would generally organize the backend around:

- **API:** Controllers, request/response DTOs, authentication, and HTTP concerns.
- **Application:** Use cases, application services, validation, and transaction coordination.
- **Domain:** Core entities, value objects, and important business invariants.
- **Infrastructure:** EF Core or Dapper implementations, database access, messaging, and external API clients.

I would use dependency injection to connect infrastructure implementations to the application interfaces, keep domain rules independent of the web framework, and test database behavior separately from business logic.

For a straightforward feature, I would keep this structure lightweight. For workflows involving inventory deductions, payment processing, or multiple external systems, I would introduce stronger boundaries and explicit consistency mechanisms.

**The key principle is that architecture should make business rules easier to understand, test, and change. Layering organizes responsibilities, dependency inversion protects the core from infrastructure, and dependency injection connects the pieces. The best architecture is the simplest one that safely handles the system's actual complexity.**

*Q32. Testing backend systems. What is the right mix of unit, integration, contract and end-to-end tests? How do you use real databases through Testcontainers rather than mocks, test transactions and concurrency, handle time and randomness in tests, and detect and fix flaky tests?*

In backend development, I use a testing strategy that balances fast feedback with confidence in the real system. I don't rely only on unit tests or mock every dependency, because a test can pass with mocked behavior while the actual database query, transaction, or API integration is broken.

I generally combine unit, integration, contract, and end-to-end tests, with most tests focused on business logic and critical integration boundaries.

### 1. The right mix of tests

I think of the testing pyramid as a guideline rather than a fixed percentage.

| Test type | What it verifies | Typical speed |
|---|---|---|
| Unit | Business rules and individual components | Fast |
| Integration | Real database, transactions, queues, and framework integration | Medium |
| Contract | Compatibility between services or API consumers and providers | Fast to medium |
| End-to-end (E2E) | Critical workflows across the running system | Slow |

**Unit tests:** I test business rules in isolation. For example, I verify that an order cannot be confirmed if it has no items. These tests should be deterministic and should not require a database or network connection.

**Integration tests:** I verify that components work together with real infrastructure. For example, I test whether an order repository correctly saves an order, whether a unique constraint is enforced, and whether a transaction rolls back after a failure.

**Contract tests:** I verify that an API provider and its consumers agree on request fields, response schemas, status codes, and error formats. These tests are especially useful when independently deployed services depend on each other.

**E2E tests:** I verify complete critical workflows, such as creating an order, deducting inventory, and confirming the final order status through the API. I keep these tests focused because they are slower and more sensitive to environmental failures.

I don't prescribe a universal ratio. I prefer many focused unit tests, a substantial set of integration tests around important infrastructure boundaries, contract tests for service interfaces, and a smaller set of high-value E2E tests.

### 2. Why use real databases with Testcontainers?

Mocks are useful for isolating business logic, but they cannot accurately reproduce database behavior such as SQL syntax, constraints, isolation levels, locking, or query execution.

I use **Testcontainers** to start a disposable instance of the actual database engine during integration tests. For example, if production uses PostgreSQL, I run PostgreSQL in a container rather than substituting an in-memory database.

A typical .NET test setup uses:

- xUnit or NUnit as the test framework.
- Testcontainers for .NET to launch the database.
- EF Core or Dapper to exercise the actual data-access code.
- A dedicated test database and deterministic test data.

The general workflow is:

1. Start the database container.
2. Wait until it is ready to accept connections.
3. Apply the real schema migrations.
4. Create isolated test data.
5. Execute repository or application code against the database.
6. Assert the results, including database state.
7. Clean up the test data and container.

For example, an integration test might verify that inserting two users with the same email fails because of the actual unique constraint. A mocked repository would not prove that the database enforces that constraint.

I also use the same database engine and, where practical, a compatible major version to avoid differences between the test and production environments.

### 3. Testing transactions and rollback behavior

Transactions are important when a business operation changes several records that must remain consistent.

Suppose creating an order involves inserting an order header and multiple order items. If inserting one item fails, the order header should not remain committed on its own.

I test both the successful and failure paths:

- On success, all expected records are committed.
- On failure, all changes covered by the transaction are rolled back.
- Constraints and exceptions behave as expected.
- Transaction boundaries match the intended business operation.

I prefer verifying transaction behavior through the actual application service and database rather than testing only whether a mock's `Commit()` method was called.

One important detail is that a test-wide transaction with automatic rollback is useful for isolation, but it can hide bugs in application-managed transaction boundaries. For critical transactional workflows, I also write tests that exercise the actual commit and rollback behavior.

For external side effects, such as sending a message, I test the transactional outbox separately. A database rollback cannot undo a message already delivered to an external service.

### 4. Testing concurrency and race conditions

Concurrency bugs often occur when two requests read the same state and then try to update it.

For example, suppose a restaurant has five units of an ingredient in stock. Two concurrent orders each attempt to deduct four units. A naive read-then-write implementation might allow both orders to succeed, resulting in negative inventory.

I test this using the real database and multiple concurrent operations.

Depending on the use case, I verify that the implementation correctly uses:

- Atomic conditional updates.
- Optimistic concurrency with version columns or concurrency tokens.
- Appropriate row-level locks.
- Database constraints.
- The required transaction isolation level.

For inventory deduction, an atomic update could look like this:

```sql
UPDATE Inventory
SET Quantity = Quantity - @Requested
WHERE ProductId = @ProductId
  AND Quantity >= @Requested;
```

The application checks the affected-row count. If zero rows were updated, the deduction failed because the inventory was insufficient or the product did not match.

In a concurrency test, I start two operations at approximately the same time and assert that only one succeeds when the available stock is insufficient for both.

I do not rely on arbitrary sleeps to create race conditions. I use synchronization barriers, tasks, or explicit coordination to make the competing operations overlap more reliably.

I also remember that concurrency tests can be sensitive to database isolation, connection configuration, and scheduling. I run them repeatedly and inspect the actual committed state, not just the returned responses.

### 5. Testing time and randomness

Code that directly depends on the system clock or an uncontrolled random generator can produce nondeterministic tests.

**Time:** In .NET, I use `TimeProvider` or an abstraction around the clock so that I can supply a fixed or controllable time in tests.

For example, I can test whether an expired token is rejected by setting the clock to a known instant instead of waiting for real time to pass.

I also test boundary conditions explicitly:

- Exactly at expiration.
- Just before expiration.
- Just after expiration.
- Daylight saving transitions when local scheduling is involved.
- Month-end and year-end boundaries.

I use UTC for actual timestamps and explicit time-zone rules for recurring local schedules.

**Randomness:** If business logic depends on random values, I inject a random-number generator or an abstraction that allows deterministic behavior in tests. For identifiers, I may inject an ID generator when the generated value itself matters to the assertion.

For security-sensitive randomness, I retain a cryptographically secure production implementation and substitute a deterministic test implementation only where appropriate. I never replace secure token generation with a predictable generator in production.

### 6. Contract testing between services

Suppose my frontend or another backend service depends on an order API.

A contract test can verify that the provider returns the expected response structure, including required fields, error responses, and status codes.

For HTTP APIs, I can use schema validation, OpenAPI-based checks, or consumer-driven contract tools such as Pact.

For example, if a consumer expects `orderId` to be a string but a provider changes it to a number, a contract test should detect that incompatibility before deployment.

I focus contract testing on compatibility guarantees rather than duplicating every business rule from unit tests.

For message-driven systems, I also validate event schemas and compatibility when fields are added, removed, or changed.

### 7. Detecting and fixing flaky tests

A flaky test passes sometimes and fails other times without a relevant code change. Common causes include:

- Race conditions and uncontrolled concurrency.
- Dependence on wall-clock time.
- Shared database state between tests.
- Tests running in parallel against the same records.
- Unreliable external services or network calls.
- Random test data without reproducible seeds.
- Fixed sleeps instead of waiting for a specific condition.
- Resource leaks or incorrect asynchronous cleanup.

My approach is to treat a flaky test as a defect rather than simply rerunning it until it passes.

I first collect the failure details, including logs, stack traces, environment information, and test execution order. Then I try to reproduce the failure by repeating the test and running it in parallel.

I fix the underlying cause by isolating test data, injecting time, using deterministic inputs, synchronizing concurrent operations, waiting for explicit conditions, and properly disposing of containers and connections.

I also use CI reporting and repeated-run tools to identify intermittent failures. Retries can be useful temporarily for diagnosis or to reduce disruption from known environmental instability, but they should not hide a test that is genuinely unreliable.

### 8. How I would implement this in a .NET backend

For an ASP.NET Core application with PostgreSQL or MySQL, I would use the following approach:

1. **Unit tests:** xUnit with simple fakes or mocks for business rules and application services.
2. **Integration tests:** Testcontainers running the real database, with production migrations and actual repository implementations.
3. **Transaction tests:** Verify atomic commits and rollbacks using the real database.
4. **Concurrency tests:** Exercise simultaneous requests and validate the resulting database state.
5. **Contract tests:** Validate API schemas and critical interactions between services.
6. **E2E tests:** Run a small number of complete workflows against a deployed test environment.
7. **CI pipeline:** Run fast tests first, then integration and contract tests, followed by critical E2E tests.

I also keep test data isolated, avoid depending on execution order, and make failures diagnosable through structured logs and clear assertions.

**The key principle is to use mocks for isolation, real infrastructure for integration confidence, and end-to-end tests for critical user workflows. A reliable test suite should be deterministic, exercise the same important behaviors as production, and fail when the system is genuinely incorrect—not because of timing or shared state.**