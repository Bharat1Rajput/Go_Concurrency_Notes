In Go, **Context** is the "Passport" of your request. It travels with every function call and carries two critical things:

1. **The Stop Signal:** "Stop working, the user disconnected or the time is up!"
    
2. **The Baggage (Metadata):** "Here is the Request ID or User ID for this specific task."
    

Without Context, if a user cancels a web request, your database might keep grinding on a heavy query for 30 seconds, wasting resources for a result nobody will ever see.

---

## 1. The Four Flavors of Context

You create a context from a parent (usually `context.Background()`) using these functions:

|**Function**|**Purpose**|
|---|---|
|**`WithCancel`**|Provides a `cancel()` function you can call manually to stop workers.|
|**`WithTimeout`**|Automatically stops after a duration (e.g., 2 seconds).|
|**`WithDeadline`**|Automatically stops at a specific clock time (e.g., 5:00 PM).|
|**`WithValue`**|Passes request-scoped data (like a Trace ID) through the call stack.|

---

## 2. How it works: The `Done()` Channel

The heart of context is the `<-ctx.Done()` channel. When a context is cancelled or times out, this channel **closes**. In your Goroutines, you use a `select` statement to watch this channel.

---

## 3. Practical Example: DB Query with Timeout

Imagine an API that must return within **200ms**. If the Database is slow, we want to give up and tell the user "Service Unavailable" rather than making them wait.

Go

```
package main

import (
	"context"
	"fmt"
	"time"
)

func slowDatabaseQuery(ctx context.Context) {
	// Simulate a DB query that takes 500ms
	select {
	case <-time.After(500 * time.Millisecond):
		fmt.Println("Database: Query finished successfully")
	case <-ctx.Done():
		// This triggers if the context times out OR is cancelled
		fmt.Println("Database: Query cancelled! Reason:", ctx.Err())
	}
}

func main() {
	// 1. Create a parent context
	parent := context.Background()

	// 2. Create a context that times out in 200ms
	ctx, cancel := context.WithTimeout(parent, 200*time.Millisecond)
	
	// IMPORTANT: Always call cancel to release resources
	defer cancel()

	go slowDatabaseQuery(ctx)

	// Wait long enough to see what happens
	time.Sleep(1 * time.Second)
}
```

---

## 4. The Dry Run: Visualizing the Shutdown

|**Step**|**Main Thread**|**Worker (DB Query)**|**Context State**|
|---|---|---|---|
|**1**|Starts Context (200ms timer)|Starts waiting for 500ms|Active|
|**2**|`main` hits `time.Sleep`|`select` is waiting|Active|
|**3**|(Clock hits 200ms)|Still waiting...|**EXPIRED**|
|**4**|-|`<-ctx.Done()` case triggers|Expired|
|**5**|-|Prints "Cancelled" and returns|Expired|
|**6**|`main` wakes up and exits|Already finished|-|

---

## 5. Best Practices: The "Rules of Context"

1. **First Argument Rule:** Context should always be the **first parameter** of a function: `func DoWork(ctx context.Context, name string)`.
    
2. **Don't Store in Structs:** Never put a Context inside a `struct`. Pass it explicitly to every function that needs it.
    
3. **The `defer cancel()` Rule:** Whenever you use `WithTimeout` or `WithCancel`, immediately write `defer cancel()`. This prevents memory leaks.
    
4. **Value is for Metadata ONLY:** Use `WithValue` for things like "Transaction ID" or "Auth Token." **Never** use it to pass optional parameters to a function.
    

---

## 6. When NOT to use Context

- **Do not use it for core logic:** If your function needs an `ID` to work, pass the `ID` as a normal integer, not inside a Context.
    
- **Do not use it if the task MUST finish:** If you are saving a critical transaction to a database that should never be interrupted halfway, don't use a context that might cancel it.
    

---

## 7. Context in REST APIs and DBs

Almost every major Go library supports Context:

- **`net/http`**: `request.Context()` gives you a context that cancels if the user closes their browser tab.
    
- **`database/sql`**: Use `QueryContext` instead of `Query`. It allows the DB driver to stop the query if the request times out.
    
- **`gRPC`**: Entirely built on Context for deadlines and metadata.