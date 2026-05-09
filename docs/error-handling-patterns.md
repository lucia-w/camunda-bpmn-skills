# Error Handling Patterns

When to use boundary events, retries, and incidents — and how to combine them correctly.

## The Three Mechanisms

### 1. Retries
The worker throws a retryable exception. Zeebe decrements the retry counter and retries after the backoff interval. When retries reach 0, the job creates an incident.

Use retries for: transient failures — network timeouts, downstream service temporarily unavailable, rate limits.

```xml
<serviceTask id="fetchCreditScore">
  <extensionElements>
    <zeebe:taskDefinition type="credit-score-fetch" retries="3" />
    <zeebe:taskHeaders>
      <zeebe:header key="retryBackoff" value="PT10S" />
    </zeebe:taskHeaders>
  </extensionElements>
</serviceTask>
```

### 2. BPMN Error Boundary Events
The worker throws a BPMN error (not a retryable exception). Zeebe catches it on the boundary event and routes to the error handling flow. Retries are not attempted.

Use error boundaries for: business errors — validation failure, external rejection, state conflict that should trigger a different process path.

```xml
<boundaryEvent attachedToRef="validateIdentity" cancelActivity="true">
  <errorEventDefinition errorRef="identityValidationFailed" />
</boundaryEvent>
<error id="identityValidationFailed" errorCode="IDENTITY_VALIDATION_FAILED" />
```

### 3. Incidents
When retries are exhausted and no boundary event catches the error, Zeebe creates an incident. The process pauses at that task. A human operator must resolve it in Operate.

Use incidents for: unexpected failures you didn't anticipate — they surface in Operate and require investigation. Don't treat incidents as a normal flow path.

## Decision Guide

```
Worker throws exception
        │
        ├─ Is it transient (will succeed if retried)?
        │         └─ Yes → set retries + backoff, no boundary event needed
        │
        ├─ Is it a known business error with a defined recovery path?
        │         └─ Yes → throw BPMN error, catch with boundary event
        │
        └─ Is it unexpected / requires human investigation?
                  └─ Yes → let it incident (retries=0 or exhausted)
```

## Common Mistakes

### Catch-all boundary event
```xml
<!-- BAD: catches everything, including programming errors -->
<boundaryEvent attachedToRef="fetchCreditScore">
  <errorEventDefinition />  <!-- no errorRef -->
</boundaryEvent>
```

A catch-all swallows unexpected exceptions (null pointers, config errors) and routes them as if they were known business errors. Always specify an `errorCode`.

### Double-handling (retries + boundary event for same error)
```xml
<!-- BAD: confusing — does it retry first, then go to boundary? -->
<serviceTask retries="3">
<boundaryEvent errorCode="CREDIT_SERVICE_UNAVAILABLE" />
```

A BPMN error thrown by the worker bypasses the retry mechanism entirely — it goes straight to the boundary event. So retries only help for retryable (non-BPMN) exceptions. Combining both for the same failure mode is confusing. Pick one.

### Silent incident with no operator runbook
If a task will incident on failure, document in the process or in an ops runbook what an operator should do when they see that incident in Operate. Incidents without runbooks become mystery production issues.

## Retry Backoff

Always set `retryBackoff` when retries > 0. Without it, all retries fire immediately, hammering a failing downstream service.

```xml
<zeebe:header key="retryBackoff" value="PT30S" />  <!-- 30 seconds between retries -->
```

Use exponential backoff where the service might be recovering:
- Retry 1: PT10S
- Retry 2: PT30S
- Retry 3: PT90S

Zeebe does not natively support exponential backoff in task headers — implement it in the worker by calculating the next backoff and setting it on the job completion, or use a fixed interval as a reasonable approximation.
