# How-To: Implement Retry Logic with Backoff and Jitter

## Problem Statement

Network operations, file operations, and external service calls can fail transiently due to temporary issues like network blips, file locks, or resource contention. Immediately failing on these temporary errors results in poor user experience. You need a resilient retry mechanism that automatically attempts failed operations with intelligent delay strategies.

## Basic Retry Pattern (Naive)

The simplest approach retries a fixed number of times with a constant delay:

```ahk
RetrySimple(operation, maxRetries := 3) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()  ; Success - return result
        } catch as e {
            if (attempts >= maxRetries)
                throw e  ; Out of retries
            Sleep(100)  ; Brief pause
        }
    }
}

; Usage
result := RetrySimple(() => Download("https://example.com/file.zip", "file.zip"))
```

**Limitations:**
- Fixed delay doesn't adapt to load
- Multiple clients retrying simultaneously create "thundering herd"
- No distinction between transient and permanent failures

## Exponential Backoff Pattern

Exponential backoff increases delay between retries to give systems time to recover:

```ahk
RetryExponentialBackoff(operation, maxRetries := 5, baseDelay := 1000) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            if (attempts >= maxRetries)
                throw e

            delay := baseDelay * (2 ** (attempts - 1))  ; 1s, 2s, 4s, 8s, 16s
            Sleep(delay)
        }
    }
}

; Usage
data := RetryExponentialBackoff(() => Download("https://api.example.com/data"), 5, 2000)
```

**How it works:**
- Attempt 1 fails: Wait 2s (2000 × 2⁰)
- Attempt 2 fails: Wait 4s (2000 × 2¹)
- Attempt 3 fails: Wait 8s (2000 × 2²)
- Attempt 4 fails: Wait 16s (2000 × 2³)
- Attempt 5 fails: Throw error

**When to use:** Network operations, API calls, database connections where the system needs time to recover.

## Production-Ready: Exponential Backoff with Jitter (Recommended)

Adding random jitter prevents synchronized retries from multiple clients overwhelming a recovering service:

```ahk
RetryWithJitter(operation, maxRetries := 5, baseDelay := 1000, maxDelay := 32000) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            if (attempts >= maxRetries)
                throw e

            ; Exponential backoff
            exponentialDelay := baseDelay * (2 ** (attempts - 1))
            cappedDelay := Min(exponentialDelay, maxDelay)

            ; Add random jitter (±25%)
            jitter := Random(-0.25 * cappedDelay, 0.25 * cappedDelay)
            actualDelay := Integer(cappedDelay + jitter)

            Sleep(actualDelay)
        }
    }
}

; Usage
data := RetryWithJitter(() => FetchFromAPI("https://api.example.com/data"), 5, 2000, 30000)
```

**Why jitter matters:** When 1000 clients all fail at the same time and retry with identical exponential backoff, they hit the server simultaneously again. Jitter spreads out retry attempts across a time window, preventing thundering herd.

**Example with jitter:**
- Attempt 1 fails: Wait 1.5-2.5s (2s ± 25%)
- Attempt 2 fails: Wait 3-5s (4s ± 25%)
- Attempt 3 fails: Wait 6-10s (8s ± 25%)

**When to use:** Production systems, distributed services, any scenario with multiple concurrent clients.

## Conditional Retry: Only Retry Specific Errors

Not all errors are transient. Retry only recoverable errors:

```ahk
RetryOnSpecificErrors(operation, maxRetries := 3, retryableErrors*) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            ; Check if error is retryable
            isRetryable := false
            for errorType in retryableErrors {
                if (e is %errorType%) {
                    isRetryable := true
                    break
                }
            }

            if (!isRetryable || attempts >= maxRetries)
                throw e  ; Don't retry this error type

            ; Exponential backoff with jitter
            baseDelay := 1000
            exponentialDelay := baseDelay * (2 ** (attempts - 1))
            jitter := Random(-250, 250)
            Sleep(exponentialDelay + jitter)
        }
    }
}

; Usage - retry only transient errors
result := RetryOnSpecificErrors(
    () => NetworkOperation(),
    5,
    OSError,      ; Retry OS/network errors
    TimeoutError  ; Retry timeouts
)
; But don't retry ValueError (bad input data - permanent failure)
```

**When to use:** When certain errors are transient (network issues, locks) but others are permanent (validation errors, missing data).

## Complete Example: Download with Retry

```ahk
#Requires AutoHotkey v2.0

DownloadWithRetry(url, destination, maxRetries := 3) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            Download(url, destination)
            return true  ; Success

        } catch OSError as err {
            LogError(Format("Download attempt {1}/{2} failed: {3}",
                attempts, maxRetries, err.Message))

            if (attempts >= maxRetries) {
                ; Out of retries - notify user
                MsgBox Format("Failed to download after {1} attempts.`n"
                    "Error: {2}", maxRetries, err.Message), "Download Failed", "IconX"
                return false
            }

            ; Exponential backoff with jitter
            baseDelay := 1000 * (2 ** (attempts - 1))
            jitter := Random(-250, 250)
            Sleep(baseDelay + jitter)
        }
    }
}

LogError(message) {
    timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
    FileAppend Format("[{1}] {2}`n", timestamp, message), "download_errors.log"
}

; Usage
if DownloadWithRetry("https://example.com/largefile.zip", "largefile.zip", 5) {
    MsgBox "Download successful!"
} else {
    MsgBox "Download failed after all retries"
}
```

## Integration with Centralized Error Handler

Combine retry logic with centralized error logging:

```ahk
RetryWithLogging(operation, operationName, maxRetries := 3) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            result := operation()
            if (attempts > 1)
                ErrorHandler.Log(Format("{1} succeeded on attempt {2}", operationName, attempts))
            return result

        } catch as e {
            ErrorHandler.LogWithRetry(
                Format("{1} failed on attempt {2}/{3}",
                    operationName, attempts, maxRetries),
                e,
                Format("Attempt {1}, Error: {2}", attempts, Type(e))
            )

            if (attempts >= maxRetries)
                throw e  ; Let global handler deal with final failure

            ; Exponential backoff with jitter
            baseDelay := 1000
            exponentialDelay := baseDelay * (2 ** (attempts - 1))
            jitter := Random(-250, 250)
            Sleep(exponentialDelay + jitter)
        }
    }
}

; Usage
config := RetryWithLogging(() => FileRead("config.ini"), "Load config", 3)
```

## Troubleshooting

### Retries happen too fast/slow

**Problem:** Default delays don't match your use case.

**Solution:** Adjust `baseDelay` and `maxDelay` parameters:

```ahk
; Fast retries for local operations
RetryWithJitter(() => FileRead("data.txt"), 3, 100, 2000)  ; 100ms base

; Slow retries for external APIs
RetryWithJitter(() => FetchAPI(), 7, 5000, 120000)  ; 5s base, 2min max
```

### Non-transient errors get retried

**Problem:** Retrying permanent failures wastes time and resources.

**Solution:** Use `RetryOnSpecificErrors` to only retry known transient errors:

```ahk
; Only retry network/timeout errors
RetryOnSpecificErrors(
    () => ProcessData(input),
    5,
    OSError,
    TimeoutError
)
```

### All retries fail with same error

**Problem:** The underlying issue isn't transient - the service is down.

**Solution:** Consider implementing a circuit breaker pattern (see docs/manuals/ahk2/how-to/circuit-breaker-pattern.md) to fail fast after detecting consistent failures.

### Retry logic causes infinite loops

**Problem:** Error handler itself throws errors during retry.

**Solution:** Ensure retry functions don't throw errors internally:

```ahk
RetryWithLogging(operation, operationName, maxRetries := 3) {
    attempts := 0
    loop maxRetries {
        attempts++
        try {
            return operation()
        } catch as e {
            ; Wrap logging in try-catch to prevent infinite loops
            try {
                ErrorHandler.Log("Retry attempt " attempts " failed: " e.Message)
            } catch {
                OutputDebug("Failed to log retry error")
            }

            if (attempts >= maxRetries)
                throw e

            Sleep(1000 * attempts)
        }
    }
}
```

## Performance Considerations

**Total retry time calculation:**

For exponential backoff with base delay B and N retries:
- Total time = B × (2⁰ + 2¹ + 2² + ... + 2⁽ᴺ⁻¹⁾)
- Example: B=1s, N=5 → Total = 1 + 2 + 4 + 8 + 16 = 31 seconds

**Guidelines:**
- **Local file operations:** 3 retries, 100ms base (max ~700ms)
- **Network requests:** 5 retries, 1s base (max ~31s)
- **External APIs:** 5-7 retries, 2-5s base, 60-120s max
- **Background tasks:** 10+ retries, long delays acceptable

## Related Concepts

- docs/manuals/ahk2/how-to/circuit-breaker-pattern.md - Fail fast after repeated failures
- docs/manuals/ahk2/how-to/custom-error-handler.md - Centralized error logging
- docs/source-reports/ahk2/ahk2_arch_04_error.md - Complete error handling architecture

## Key Takeaway

Retry logic transforms brittle scripts into resilient systems. Use exponential backoff with jitter for production code to handle transient failures gracefully while preventing thundering herd problems. Always set maximum retry counts and distinguish between transient errors (retry) and permanent failures (fail fast).
