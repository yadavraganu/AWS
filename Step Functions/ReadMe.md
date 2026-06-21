## **AWS Step Functions State Types**

| **State Type** | **Purpose** | **How It Works** | **Example Use Case** | **Example Snippet** |
|----------------|-------------|------------------|-----------------------|----------------------|
| **Task** | Executes a unit of work by calling an AWS service (e.g., Lambda, Glue, ECS) | Sends input to the service, waits for result | Call a Lambda to process user data | `"Type": "Task", "Resource": "arn:aws:lambda:..."` |
| **Pass** | Passes input to output without doing anything | Can modify or inject static data | Placeholder or testing state | `"Type": "Pass", "Result": {"status": "ok"}` |
| **Wait** | Pauses execution for a fixed time or until a timestamp | Useful for delays, polling, or throttling | Wait 5 seconds before retrying | `"Type": "Wait", "Seconds": 5` |
| **Choice** | Adds conditional branching logic | Evaluates conditions and routes to different states | Route based on user type or status | `"Type": "Choice", "Choices": [{"Variable": "$.status", "StringEquals": "active", "Next": "ActiveFlow"}]` |
| **Succeed** | Marks the workflow as successfully completed | Ends execution with success | Final state after successful processing | `"Type": "Succeed"` |
| **Fail** | Marks the workflow as failed | Ends execution with error details | Used when a critical error occurs | `"Type": "Fail", "Error": "ValidationError", "Cause": "Missing user ID"` |
| **Parallel** | Runs multiple branches concurrently | Each branch is a mini workflow | Run data enrichment and logging in parallel | `"Type": "Parallel", "Branches": [...]` |
| **Map** | Iterates over items in an array and runs a sub-workflow for each | Like a loop over a list | Process each record in a batch | `"Type": "Map", "ItemsPath": "$.records", "Iterator": {...}` |

### Additional Notes:

- **Task** is the most versatile and commonly used state.
- **Choice** is like an `if-else` or `switch` statement.
- **Map** is like a `for-each` loop, ideal for batch processing.
- **Parallel** is great for concurrent tasks like logging and notifications.
- **Pass**, **Succeed**, and **Fail** are control flow helpers.

## **Step Functions Variable Control Table**

| Field         | Purpose | Default Behavior | Example | Result |
|---------------|---------|------------------|---------|--------|
| **InputPath** | Filters the input to the state | Entire input is passed | `"InputPath": "$.user"` | Only `user` object is passed to the state |
| **Parameters** | Constructs custom input using JSONPath | Input (after InputPath) is passed as-is | `"Parameters": { "id.$": "$.user.id" }` | Input reshaped to `{ "id": 1 }` |
| **ResultPath** | Specifies where to place the result | Result replaces input | `"ResultPath": "$.result"` | Merges result under `result` key |
| **OutputPath** | Filters output passed to next state | Entire result is passed | `"OutputPath": "$.result.status"` | Only `status` value is passed forward |

