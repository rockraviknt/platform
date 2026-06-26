
> In code reviews I prioritize correctness first because readable but incorrect code still fails. Then I review maintainability and non-functional concerns such as test coverage, performance, and security. I look for issues like race conditions, N+1 queries, SQL injection, missing pagination, hardcoded secrets, retry behavior, and observability gaps. I try to give actionable comments rather than subjective opinions.

Then walk through each category with examples.

# 1. Readability

Questions I ask:

* Is the code easy to understand?
* Are names meaningful?
* Is a method doing too many things?
* Is logic duplicated?
* Can another engineer maintain this six months later?

Example code:

```java
public void process(User u){
   //validate
   //transform
   //save
   //send notification
}
```

Review comment:

> This method handles multiple responsibilities (validation, transformation, persistence, notification). Consider extracting separate methods or services to improve readability and maintainability.

Another example:

```java
int x=5;
```

Comment:

> `x` is not descriptive. Consider a name like `maxRetryAttempts` to make intent clear.

---

# 2. Correctness

Questions:

* Does the code actually solve the problem?
* Any edge cases?
* Any race conditions?
* Null handling?
* Boundary conditions?

Example:

```java
public double divide(int a,int b){
   return a/b;
}
```

Review comment:

> Missing validation for `b=0`; this can cause runtime exceptions.

Example:

```java
inventoryCount--;
```

Comment:

> This update may cause race conditions under concurrent requests. Consider optimistic locking or atomic operations.

---

# 3. Test coverage

Questions:

* Are both happy path and failure path tested?
* Edge cases?
* Timeouts?
* Retry behavior?

Example PR:

```java
public Booking createBooking()
```

Only test:

```java
shouldCreateBooking()
```

Review comment:

> Missing tests for failure scenarios such as payment timeout, invalid input, and downstream service failures.

Another:

> Retry logic was introduced but I don't see tests validating maximum retry attempts.

---

# 4. Performance

Questions:

* N+1 queries?
* Excessive object creation?
* Unnecessary loops?
* Expensive DB calls?
* Memory concerns?

Bad code:

```java
for(User u:users){
     orderRepository.findByUserId(u.getId());
}
```

Review comment:

> This may create an N+1 query problem. Consider fetching data in batches or using a join.

Bad code:

```java
Connection c=
DriverManager.getConnection();
```

inside API request:

Review comment:

> Creating a DB connection for every request is expensive. Use connection pooling.

Another:

> This API calls downstream services sequentially. If dependencies are independent, parallel execution using `CompletableFuture` may reduce latency.

---

# 5. Security

This area often separates senior engineers from mid-level engineers.

Questions:

* SQL injection?
* Hardcoded secrets?
* Authentication/authorization?
* Sensitive logging?
* Input validation?

Bad code:

```java
String sql=
"SELECT * FROM users WHERE id="+id;
```

Review comment:

> Potential SQL injection risk. Use parameterized queries or prepared statements.

Bad code:

```java
String apiKey=
"ABC123SECRET";
```

Review comment:

> Secrets should not be hardcoded. Store them in a secrets manager or environment configuration.

Bad logging:

```java
log.info("User password {}",password);
```

Review comment:

> Avoid logging sensitive information such as passwords or tokens.

---

### SQL Injection

```java
String q=
"SELECT * FROM user where id="+id;
```

Comment:

> Potential SQL injection; use prepared statements.

---

### Expensive DB initialization per request

```java
public User getUser(){

Connection c=
DriverManager.getConnection();

...
}
```

Comment:

> Database connection creation per request is expensive. Consider connection pooling.

---

### Missing pagination

```java
GET /orders
```

returns all records.

Comment:

> Returning all records can lead to memory and latency issues. Consider cursor-based pagination.

---

### Hardcoded vendor keys

```java
String stripeKey="secret_key";
```

Comment:

> Move credentials to a secrets manager.

---

### Missing timeout

```java
paymentClient.call();
```

Comment:

> Missing timeout handling may cause request threads to hang under downstream failures.

---

### Missing retry strategy

```java
notificationService.send();
```

Comment:

> Consider retry with exponential backoff and max retry limits for transient failures.

---

### Missing observability

```java
catch(Exception e){
}
```

Comment:

> Exceptions are swallowed without logging or metrics; troubleshooting production issues will be difficult.

---

