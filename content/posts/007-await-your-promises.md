---
title: 'Await your Promises'
date: '2025-04-02T12:00:00Z'
draft: false
slug: 'await-your-promises'
keywords: [javascript, typescript, cloudflare, async, await, promises, workers, serverless]
tags: [cloudflare workers, javascript, async]
---

After years of reviewing codebases and debugging mysterious production issues at Cloudflare, I've become somewhat of a Promise detective. One pattern I've seen repeatedly causing headaches is improper Promise handling. It's 2 AM, and I'm staring at logs from a serverless function that mysteriously failed in production - all because someone forgot to await a Promise. _Again._

Let's fix this problem once and for all.

## WTF is a Promise anyway?

A Promise in JavaScript represents a value that might not be available yet but will eventually resolve (or explode in your face). Promises have three states:

- **Pending**: Initial state, neither fulfilled nor rejected
- **Fulfilled**: Operation completed successfully
- **Rejected**: Operation failed (and your user is probably seeing an error page)

```javascript
// A simple Promise that resolves after a delay
function delay(ms) {
	return new Promise((resolve) => {
		// After the timeout, this Promise will move from pending to fulfilled
		setTimeout(resolve, ms);
	});
}
```

## Why You Should Always Await Promises

I've spent countless hours debugging issues that could have been avoided with proper Promise handling. Here are the problems I keep encountering:

### 1. Execution Order Becomes Unpredictable

```javascript
// ❌ PROBLEMATIC - No awaiting
function processData() {
	const data = fetchData(); // Returns a Promise
	return processResult(data); // Likely error - data is a Promise, not the resolved value
}

// ✅ CORRECT - Using await
async function processData() {
	// Wait for the data to be fetched before processing
	const data = await fetchData();
	return processResult(data); // Now data contains the actual resolved value
}
```

### 2. Error Handling Becomes Difficult

I've traced production outages back to this exact pattern:

```javascript
// ❌ PROBLEMATIC - Errors are swallowed
function saveUserData(user) {
	// This Promise might reject, but we'll never know
	saveToDatabase(user);
	console.log("User saved successfully"); // This will log even if the save fails
}

// ✅ CORRECT - Errors are caught
async function saveUserData(user) {
	try {
		await saveToDatabase(user);
		console.log("User saved successfully"); // Only logs on success
	} catch (error) {
		console.error("Failed to save user:", error);
		// Handle the error appropriately
	}
}
```

### 3. Serverless Runtime Problems

In my work with Cloudflare Workers, I've repeatedly encountered this issue:

```javascript
// ❌ PROBLEMATIC - In Cloudflare Workers
export default {
	async fetch(request, env) {
		// Worker might terminate before this completes
		updateAnalytics(request);
		return new Response("Hello World");
	},
};

// ✅ CORRECT - Awaiting in Workers
export default {
	async fetch(request, env, ctx) {
		// Either await the Promise...
		await updateAnalytics(request);
		return new Response("Hello World");

		// ...or explicitly tell Workers to "wait"
		// using waitUntil (if you don't need to await the result)
		const analyticsPromise = updateAnalytics(request);
		ctx.waitUntil(analyticsPromise);
		return new Response("Hello World");
	},
};
```

## Common Issues When Not Awaiting Promises

### Race Conditions

I've debugged this scenario more times than I can count:

```javascript
// ❌ PROBLEMATIC - Race condition
async function updateUserAndPreferences(userId) {
	// These two operations run in parallel without coordination
	updateUser(userId, { lastLogin: new Date() });
	updatePreferences(userId, { theme: "dark" });

	// What if updateUser fails? updatePreferences might still run
	// What if updatePreferences relies on updateUser being completed first?
}

// ✅ CORRECT - Controlled execution
async function updateUserAndPreferences(userId) {
	// First update the user
	await updateUser(userId, { lastLogin: new Date() });

	// Then update preferences (only happens if updateUser succeeds)
	await updatePreferences(userId, { theme: "dark" });
}

// ✅ CORRECT - Parallel execution when appropriate
async function updateUserAndPreferences(userId) {
	// Run in parallel if they don't depend on each other
	const [userResult, prefResult] = await Promise.all([
		updateUser(userId, { lastLogin: new Date() }),
		updatePreferences(userId, { theme: "dark" }),
	]);

	// Both operations completed successfully
	return { userResult, prefResult };
}
```

### Unhandled Promise Rejections

Unhandled rejections can cause cascading failures that are difficult to trace:

```javascript
// ❌ PROBLEMATIC - Unhandled rejection
function processPayment(paymentDetails) {
	validatePayment(paymentDetails) // Returns a Promise
		.then(() => {
			// Process payment logic
		});
	// No .catch() means rejection will be unhandled
}

// ✅ CORRECT - Handling rejections
async function processPayment(paymentDetails) {
	try {
		await validatePayment(paymentDetails);
		// Process payment logic
	} catch (error) {
		// Handle validation errors
		throw new Error(`Payment validation failed: ${error.message}`);
	}
}
```

### Memory Leaks in Long-Running Applications

Not awaiting background scheduled tasks can lead to memory leaks. 

```javascript
// ❌ PROBLEMATIC - Potential memory leak
function startPeriodicSync() {
	setInterval(() => {
		// This creates a new Promise chain every 30 seconds
		// but doesn't ensure it completes or handle errors
		syncData().then(processSyncResults);
	}, 30000);
}

// ✅ CORRECT - Proper lifecycle management
function startPeriodicSync() {
	setInterval(async () => {
		try {
			const data = await syncData();
			await processSyncResults(data);
		} catch (error) {
			console.error("Sync failed:", error);
			// Maybe implement backoff or recovery logic
		}
	}, 30000);
}
```

## Beyond Promise.all: Using Modern Promise Combinators

Sometimes you need more control than just "wait for everything." Modern JavaScript Promise APIs have got you covered:

```javascript
// Promise.allSettled - Waits for all promises regardless of fulfillment or rejection
async function fetchAllUserData(userIds) {
	const promises = userIds.map((id) => fetchUserData(id));
	const results = await Promise.allSettled(promises);

	// Process all results, including both successes and failures
	const successfulResults = results
		.filter((result) => result.status === "fulfilled")
		.map((result) => result.value);

	const failedResults = results
		.filter((result) => result.status === "rejected")
		.map((result) => result.reason);

	console.log(
		`Successfully fetched ${successfulResults.length} of ${userIds.length} users`
	);
	return { successful: successfulResults, failed: failedResults };
}

// Promise.any - Returns first fulfilled promise (ignores rejections unless all reject)
async function fetchFromFastestMirror(mirrors) {
	try {
		const response = await Promise.any(
			mirrors.map((url) =>
				fetch(url).then((res) => {
					if (!res.ok)
						throw new Error(`Failed with status ${res.status}`);
					return res;
				})
			)
		);
		return response;
	} catch (error) {
		// AggregateError is thrown if all promises reject
		console.error(
			`All mirrors failed: ${error.errors
				.map((e) => e.message)
				.join(", ")}`
		);
		throw error;
	}
}
```

## When It's Actually OK Not to Await Promises

Despite my rant, there are legitimate scenarios where not awaiting a Promise is acceptable. Here are examples I've seen work well:

### 1. When Using Explicit Promise Management Patterns

```javascript
// Fire and collect pattern
function processInBatches(items) {
	// Start all operations in parallel
	const promises = items.map((item) => processItem(item));

	// But await all results before continuing
	return Promise.all(promises);
}
```

### 2. When Using Specialized APIs That Handle Promises

Many frameworks provide mechanisms to handle background tasks:

```javascript
// Using Cloudflare Workers' waitUntil (I use this pattern daily)
export default {
	async fetch(request, env, ctx) {
		// We don't await because we don't need the result to respond
		// But we DO properly tell the runtime about the Promise
		const backgroundTask = logRequestDetails(request);
		ctx.waitUntil(backgroundTask);

		return new Response("Hello!");
	},
};
```

### 3. For Truly Non-Critical Background Tasks (With Caution)

```javascript
// Fire-and-forget for analytics (with error handling)
function logPageView(pageData) {
	// We don't await, but we DO handle errors
	sendAnalytics(pageData).catch((error) => {
		// Log error but don't let it affect the main application flow
		console.error("Analytics error:", error);
	});
}
```

### 4. In Event-Driven Architectures

When using event listeners and callbacks:

```javascript
// Event-driven architecture
eventEmitter.on("user-login", (user) => {
	// This runs asynchronously when the event fires
	// We handle errors within this scope, but don't block the event emitter
	updateUserStatus(user).catch((error) =>
		reportError("user-status-update-failed", error)
	);
});
```

## Practical Techniques for Debugging Promise Issues

Promises can be notoriously difficult to debug. These techniques have saved my sanity:

### 1. Add Intermediate Console Logs

Like any good developer, printing to the console is the most effective method of debugging.

```javascript
fetchData()
	.then((data) => {
		console.log("Data received:", data);
		return processData(data);
	})
	.then((result) => {
		console.log("Processing complete:", result);
		return saveResult(result);
	})
	.catch((error) => {
		// Log the full error object, not just message
		console.error("Operation failed:", error);
	});
```

### 2. Create Debugging Wrappers

For complex applications, create utility functions that add debugging statements for you:

```javascript
function debugPromise(promise, label) {
	const start = performance.now();
	console.log(`${label}: Started`);

	return promise
		.then((result) => {
			const duration = performance.now() - start;
			console.log(
				`${label}: Resolved after ${duration.toFixed(2)}ms`,
				result
			);
			return result;
		})
		.catch((error) => {
			const duration = performance.now() - start;
			console.error(
				`${label}: Rejected after ${duration.toFixed(2)}ms`,
				error
			);
			throw error; // Re-throw to maintain the rejection
		});
}

// Usage
const userPromise = debugPromise(fetchUserData(userId), "UserFetch");
```

## Best Practices for Promise Management

After reviewing hundreds of PRs and fixing dozens of Promise-related bugs, here are the practices I've found most effective:

### Use Promise-Specific ESLint Rules

These ESLint rules have been invaluable:

```javascript
// .eslintrc.js
module.exports = {
	rules: {
		// Requires await or Promise handling for async calls
		"require-await": "error",

		// Ensures promises have error handling
		"promise/catch-or-return": "error",

		// Prevents misuse of Promise executor functions
		"promise/no-new-statics": "error",

		// Disallows unnecessary Promise nesting
		"promise/no-return-wrap": "error",
	},
};
```

### Use TypeScript for Better Static Analysis

TypeScript has saved me countless hours of debugging:

```typescript
// TypeScript can help catch Promise-related errors
async function getUserData(userId: string): Promise<UserData> {
  const response = await fetch(`/api/users/${userId}`);

  if (!response.ok) {
    throw new Error(`Failed to fetch user data: ${response.statusText}`);
  }

  // TypeScript ensures this function returns a Promise<UserData>
  return response.json();
}
```

### Implement Global Unhandled Rejection Handlers

I call this the "safety net" approach, and I've seen it catch issues in every project I've implemented it in:

```javascript
// In browser environments or Cloudflare Workers
window.addEventListener("unhandledrejection", (event) => {
	console.error("Unhandled promise rejection:", event.reason);
	// Report to error monitoring service
	errorMonitoringService.report(event.reason);
});

// In Node.js
process.on("unhandledRejection", (reason, promise) => {
	console.error("Unhandled Rejection at:", promise, "reason:", reason);
	// Report to monitoring service or exit process with error
});
```

### Use Utility Functions for Common Patterns

After writing similar error handling code repeatedly, I started creating utility functions:

```javascript
// Utility for timeout and error handling
async function withTimeout(promise, timeoutMs, errorMessage) {
	// Create a Promise that rejects after timeoutMs
	const timeoutPromise = new Promise((_, reject) => {
		setTimeout(() => reject(new Error(errorMessage)), timeoutMs);
	});

	// Race the original Promise against the timeout
	return Promise.race([promise, timeoutPromise]);
}

// Usage
async function fetchWithTimeout(url) {
	return withTimeout(
		fetch(url),
		5000,
		`Fetch request to ${url} timed out after 5 seconds`
	);
}
```

## Specific Considerations for Serverless Environments

After building numerous applications on Cloudflare Workers, I've developed these patterns for reliable Promise handling:

### Pattern 1: Separate Core Logic from Background Tasks

```javascript
export default {
	async fetch(request, env, ctx) {
		// Handle the main request flow
		try {
			const result = await handleMainRequest(request, env);

			// Return response quickly, handle non-critical tasks in background
			const response = new Response(JSON.stringify(result), {
				headers: { "Content-Type": "application/json" },
			});

			// Background task happens after response is sent
			ctx.waitUntil(recordAnalytics(request, result));

			return response;
		} catch (error) {
			return handleError(error, ctx);
		}
	},
};
```

### Pattern 2: Use Dedicated Error Handling

```javascript
function handleError(error, ctx) {
	// Log and report the error in the background
	ctx.waitUntil(
		reportErrorToMonitoring(error).catch((reportingError) => {
			// Always log if the reporting itself fails
			console.error("Failed to report error:", reportingError);
		})
	);

	// Return an appropriate error response
	return new Response(
		JSON.stringify({ error: "An unexpected error occurred" }),
		{
			status: 500,
			headers: { "Content-Type": "application/json" },
		}
	);
}
```

### Pattern 3: Implement Timeout Protection

```javascript
// Utility for ensuring requests don't exceed runtime limits
async function withWorkerTimeout(promise, timeoutMs = 50000) {
	const timeoutPromise = new Promise((_, reject) => {
		setTimeout(() => {
			reject(new Error(`Operation timed out after ${timeoutMs}ms`));
		}, timeoutMs);
	});

	return Promise.race([promise, timeoutPromise]);
}

// Usage in a Worker
export default {
	async fetch(request, env, ctx) {
		try {
			// Ensure we complete before the Worker's time limit
			const result = await withWorkerTimeout(
				processRequest(request, env),
				30000 // 30 second timeout (Workers have 50ms limit)
			);

			return new Response(JSON.stringify(result));
		} catch (error) {
			if (error.message.includes("timed out")) {
				// Special handling for timeout errors
				return new Response(
					JSON.stringify({
						error: "Request processing took too long",
					}),
					{ status: 408 }
				);
			}
			return new Response(JSON.stringify({ error: error.message }), {
				status: 500,
			});
		}
	},
};
```

## Conclusion

After spending countless hours debugging Promise-related issues, I've become somewhat of an evangelist for proper Promise handling. Promises aren't just syntax sugar – they're contracts for async behavior that need to be honored.

The golden rules I now use in every codebase I work on:

1. Always await Promises in the main execution path
2. Use proper error handling for all asynchronous operations
3. For background tasks, explicitly use the appropriate pattern for your environment (like `ctx.waitUntil` in Cloudflare Workers)
4. When intentionally not awaiting, always add error handling
5. Use tools like ESLint and TypeScript to catch Promise-related issues early

By following these guidelines, you'll avoid many of the problems I've spent way too much time debugging over the years, and build more reliable, maintainable asynchronous JavaScript applications.
