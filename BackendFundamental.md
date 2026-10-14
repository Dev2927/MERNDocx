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