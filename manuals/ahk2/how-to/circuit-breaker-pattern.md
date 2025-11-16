# How-To: Implement Circuit Breaker for Failing Services

## Problem Statement

When an external service (API, database, file server) is consistently failing, continuing to retry every request wastes resources, increases latency, and can make the problem worse by overwhelming a struggling service. You need a mechanism to detect persistent failures, stop sending requests to give the service time to recover, and automatically test for recovery.

The circuit breaker pattern prevents cascading failures and allows graceful degradation when dependencies are unavailable.

## Circuit Breaker States

A circuit breaker has three states:

**CLOSED (Normal Operation)**
- Requests pass through normally
- Failures are counted
- When failure threshold is reached → Open circuit

**OPEN (Failing Fast)**
- Requests are immediately rejected without attempting the operation
- No load on failing service
- After timeout period → Half-Open to test recovery

**HALF_OPEN (Testing Recovery)**
- Limited requests allowed through to test if service recovered
- If requests succeed → Close circuit (recovery confirmed)
- If requests fail → Open circuit again (still broken)

## Basic Implementation

```ahk
class CircuitBreaker {
    static State := {CLOSED: 0, OPEN: 1, HALF_OPEN: 2}

    __New(failureThreshold := 5, timeout := 60000, successThreshold := 2) {
        this.failureThreshold := failureThreshold    ; Failures before opening
        this.timeout := timeout                      ; Wait time before retry (ms)
        this.successThreshold := successThreshold    ; Successes to close circuit

        this.failureCount := 0
        this.successCount := 0
        this.state := CircuitBreaker.State.CLOSED
        this.lastFailureTime := 0
    }

    Call(operation) {
        ; If circuit open, check if timeout elapsed
        if (this.state == CircuitBreaker.State.OPEN) {
            if (A_TickCount - this.lastFailureTime > this.timeout) {
                this.state := CircuitBreaker.State.HALF_OPEN
                this.successCount := 0
            } else {
                throw Error("Circuit breaker OPEN - service unavailable")
            }
        }

        ; Try operation
        try {
            result := operation()
            this.OnSuccess()
            return result
        } catch as e {
            this.OnFailure()
            throw e
        }
    }

    OnSuccess() {
        this.failureCount := 0

        if (this.state == CircuitBreaker.State.HALF_OPEN) {
            this.successCount++
            if (this.successCount >= this.successThreshold)
                this.state := CircuitBreaker.State.CLOSED  ; Recovered
        }
    }

    OnFailure() {
        this.lastFailureTime := A_TickCount
        this.failureCount++

        if (this.failureCount >= this.failureThreshold)
            this.state := CircuitBreaker.State.OPEN  ; Trip circuit
    }

    GetState() {
        states := ["CLOSED", "OPEN", "HALF_OPEN"]
        return states[this.state + 1]
    }
}
```

## Usage Example

```ahk
#Requires AutoHotkey v2.0

; Create circuit breaker for API calls
apiBreaker := CircuitBreaker(5, 60000, 2)  ; 5 failures, 60s timeout, 2 successes to close

; Make requests with circuit breaker protection
loop {
    try {
        result := apiBreaker.Call(() => FetchFromAPI())
        ProcessResult(result)
        TrayTip "Success", "API call succeeded. Circuit: " apiBreaker.GetState()

    } catch as e {
        if (InStr(e.Message, "Circuit breaker OPEN")) {
            TrayTip "Service Unavailable", "API temporarily unavailable. Retrying in 60s...", 3
        } else {
            TrayTip "API Error", "API call failed: " e.Message, 3
        }
    }

    Sleep(5000)  ; Poll every 5 seconds
}

FetchFromAPI() {
    ; Simulate API call that might fail
    response := Download("https://api.example.com/data")
    return response
}

ProcessResult(data) {
    MsgBox "Received data: " StrLen(data) " bytes"
}
```

## State Transitions Explained

**CLOSED → OPEN (Trip)**
1. Service starts failing
2. Failure count increments with each failure
3. When `failureCount >= failureThreshold` → Circuit opens
4. All subsequent requests fail immediately without attempting operation

**OPEN → HALF_OPEN (Timeout)**
1. Circuit has been open for `timeout` milliseconds
2. Next request changes state to HALF_OPEN
3. Request is allowed through to test service

**HALF_OPEN → CLOSED (Recovery)**
1. In HALF_OPEN state, successful requests increment `successCount`
2. When `successCount >= successThreshold` → Circuit closes
3. Service is considered recovered, normal operation resumes

**HALF_OPEN → OPEN (Still Broken)**
1. In HALF_OPEN state, any failure resets recovery
2. Circuit immediately opens again
3. Must wait another timeout period

## Enhanced Version with Logging

```ahk
class CircuitBreakerWithLogging extends CircuitBreaker {
    __New(name, failureThreshold := 5, timeout := 60000, successThreshold := 2) {
        super.__New(failureThreshold, timeout, successThreshold)
        this.name := name
    }

    Call(operation) {
        currentState := this.GetState()

        try {
            result := super.Call(operation)
            this.Log("SUCCESS", "Operation completed. State: " this.GetState())
            return result

        } catch as e {
            newState := this.GetState()
            if (currentState != newState) {
                this.Log("STATE_CHANGE", Format("Circuit changed: {1} → {2}",
                    currentState, newState))
            }

            if (InStr(e.Message, "Circuit breaker OPEN")) {
                this.Log("REJECTED", "Request rejected - circuit OPEN")
            } else {
                this.Log("FAILURE", "Operation failed: " e.Message)
            }
            throw e
        }
    }

    Log(level, message) {
        timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
        logLine := Format("[{1}] [{2}] {3}: {4}`n",
            timestamp, level, this.name, message)
        FileAppend logLine, "circuit_breaker.log"
    }
}

; Usage
apiBreaker := CircuitBreakerWithLogging("ExternalAPI", 3, 30000, 2)
```

## Multiple Circuit Breakers for Different Services

Manage multiple dependencies with separate circuit breakers:

```ahk
; Different services have different reliability characteristics
breakers := Map(
    "PaymentAPI",    CircuitBreaker(3, 120000, 2),  ; Critical, long timeout
    "RecommendAPI",  CircuitBreaker(5, 30000, 1),   ; Non-critical, short timeout
    "LoggingAPI",    CircuitBreaker(10, 60000, 3)   ; Best-effort, tolerant
)

CallService(serviceName, operation) {
    if (!breakers.Has(serviceName)) {
        throw ValueError("Unknown service: " serviceName)
    }

    breaker := breakers[serviceName]

    try {
        return breaker.Call(operation)
    } catch as e {
        if (InStr(e.Message, "Circuit breaker OPEN")) {
            ; Service unavailable - handle gracefully
            switch serviceName {
                case "PaymentAPI":
                    MsgBox "Payment service unavailable. Please try again later.", "Error", "IconX"
                    return ""

                case "RecommendAPI":
                    ; Non-critical - use fallback
                    return GetCachedRecommendations()

                case "LoggingAPI":
                    ; Best-effort - fail silently
                    OutputDebug "Logging service unavailable: " e.Message
                    return ""
            }
        }
        throw e
    }
}

; Usage
paymentResult := CallService("PaymentAPI", () => ProcessPayment(amount))
```

## Integration with Retry Logic

Combine circuit breaker with retry for maximum resilience:

```ahk
RetryWithCircuitBreaker(breaker, operation, maxRetries := 3) {
    attempts := 0

    loop maxRetries {
        attempts++

        try {
            return breaker.Call(operation)

        } catch as e {
            if (InStr(e.Message, "Circuit breaker OPEN")) {
                ; Circuit open - don't retry immediately
                throw e
            }

            ; Other error - retry with backoff
            if (attempts >= maxRetries) {
                throw e
            }

            delay := 1000 * (2 ** (attempts - 1))
            jitter := Random(-250, 250)
            Sleep(delay + jitter)
        }
    }
}

; Usage
apiBreaker := CircuitBreaker(5, 60000)
result := RetryWithCircuitBreaker(apiBreaker, () => FetchData(), 3)
```

## Complete Example: API Client with Circuit Breaker

```ahk
#Requires AutoHotkey v2.0

class ResilientAPIClient {
    __New(baseURL, failureThreshold := 5, timeout := 60000) {
        this.baseURL := baseURL
        this.breaker := CircuitBreaker(failureThreshold, timeout)
        this.requestCount := 0
        this.failureCount := 0
    }

    Get(endpoint) {
        this.requestCount++
        url := this.baseURL . endpoint

        try {
            result := this.breaker.Call(() => Download(url))
            return result

        } catch as e {
            this.failureCount++

            if (InStr(e.Message, "Circuit breaker OPEN")) {
                ; Circuit open - service unavailable
                return this.HandleServiceUnavailable()
            }

            ; Other error
            ErrorHandler.Log("API request failed", url ": " e.Message)
            throw e
        }
    }

    HandleServiceUnavailable() {
        ; Graceful degradation strategy
        MsgBox Format("API service temporarily unavailable.`n"
            "Success rate: {1}%`n"
            "Circuit state: {2}",
            Round((this.requestCount - this.failureCount) / this.requestCount * 100),
            this.breaker.GetState()), "Service Unavailable", "Iconi T10"

        return ""  ; Return empty result or cached data
    }

    GetStats() {
        return {
            requests: this.requestCount,
            failures: this.failureCount,
            successRate: Round((this.requestCount - this.failureCount) / this.requestCount * 100, 2),
            circuitState: this.breaker.GetState()
        }
    }
}

; Usage
client := ResilientAPIClient("https://api.example.com", 3, 30000)

; Make requests
try {
    data := client.Get("/users/123")
    ProcessUserData(data)
} catch as e {
    MsgBox "Failed to get user data: " e.Message
}

; Check stats
stats := client.GetStats()
MsgBox Format("Requests: {1}`nFailures: {2}`nSuccess Rate: {3}%`nCircuit: {4}",
    stats.requests, stats.failures, stats.successRate, stats.circuitState)
```

## Troubleshooting

### Circuit opens too quickly

**Problem:** Normal transient failures trip the circuit.

**Solution:** Increase `failureThreshold`:

```ahk
; Too sensitive - opens after 3 failures
breaker := CircuitBreaker(3, 60000)

; Better - tolerates more transient issues
breaker := CircuitBreaker(10, 60000)
```

### Circuit stays open too long

**Problem:** Service recovers but circuit stays open.

**Solution:** Reduce `timeout`:

```ahk
; Too long - stays open 5 minutes
breaker := CircuitBreaker(5, 300000)

; Better - tests recovery after 30s
breaker := CircuitBreaker(5, 30000)
```

### Circuit flaps between states

**Problem:** Circuit opens and closes repeatedly.

**Solution:** Increase `successThreshold` to require more proof of recovery:

```ahk
; Flaps - one success closes circuit
breaker := CircuitBreaker(5, 60000, 1)

; Stable - requires 3 consecutive successes
breaker := CircuitBreaker(5, 60000, 3)
```

### Half-open state lasts too long

**Problem:** In HALF_OPEN, waiting for success threshold takes many requests.

**Solution:** Lower `successThreshold` or implement timeout for HALF_OPEN state:

```ahk
class CircuitBreakerWithHalfOpenTimeout extends CircuitBreaker {
    __New(failureThreshold := 5, timeout := 60000, successThreshold := 2, halfOpenTimeout := 10000) {
        super.__New(failureThreshold, timeout, successThreshold)
        this.halfOpenTimeout := halfOpenTimeout
        this.halfOpenStartTime := 0
    }

    Call(operation) {
        ; Check if HALF_OPEN timeout exceeded
        if (this.state == CircuitBreaker.State.HALF_OPEN) {
            if (A_TickCount - this.halfOpenStartTime > this.halfOpenTimeout) {
                ; Taking too long - go back to OPEN
                this.state := CircuitBreaker.State.OPEN
                throw Error("Circuit breaker OPEN - half-open timeout exceeded")
            }
        }

        return super.Call(operation)
    }

    OnFailure() {
        super.OnFailure()
        if (this.state == CircuitBreaker.State.HALF_OPEN) {
            this.halfOpenStartTime := A_TickCount
        }
    }
}
```

## Performance Considerations

**Circuit breaker overhead:**
- State check: ~1 microsecond
- Success/failure tracking: ~2 microseconds
- Total overhead: Negligible (<5 microseconds per call)

**Memory usage:**
- Each circuit breaker: ~100 bytes
- Can create hundreds without performance impact

**When to use multiple breakers:**
- Different services: Always use separate breakers
- Different endpoints on same service: Usually share one breaker
- Different request types (critical vs optional): Consider separate breakers

## Related Concepts

- docs/manuals/ahk2/how-to/retry-network-operations.md - Retry with exponential backoff
- docs/manuals/ahk2/how-to/custom-error-handler.md - Centralized error logging
- docs/source-reports/ahk2/ahk2_arch_04_error.md - Error handling architecture

## Key Takeaway

The circuit breaker pattern prevents cascading failures by detecting persistent problems, failing fast to reduce load on struggling services, and automatically testing for recovery. Use circuit breakers for all external service dependencies to build resilient systems that gracefully degrade instead of catastrophically failing.
