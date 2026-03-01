# WellMed Reusable API Client Pattern
## Architecture, Implementation & ADR

---

# Part 1: Overview

## 1.1 Goals

Build a single, reusable API client package that provides:

| Capability | Description | Business Value |
|------------|-------------|----------------|
| **Retry with backoff** | Automatically retry failed requests with exponential delay | Handle transient failures in government APIs |
| **Circuit breaker** | Stop calling failing services temporarily | Prevent cascade failures, protect system resources |
| **Logging** | Record all requests and responses | Debugging, audit trail, compliance |
| **Metrics** | Count successes, failures, latencies | Alerting, performance monitoring |
| **Dead letter queue** | Capture all permanent failures | Ensure no data loss, enable manual recovery |
| **Timeout management** | Cancel slow requests | Prevent hanging requests |
| **Request correlation** | Attach unique ID to all requests | Trace requests across services |

## 1.2 Design Principles

1. **Build once, use everywhere**: Same client for SATU SEHAT, P-Care, Jurnal, Xendit, all future APIs
2. **Fail gracefully**: External API failures should not crash our system
3. **Visibility**: Every API interaction must be observable
4. **Composable**: Easy to add new capabilities (rate limiting, caching, etc.)

## 1.3 Supported External Services

| Service | Type | Priority | Notes |
|---------|------|----------|-------|
| SATU SEHAT | Government | High | Unreliable, requires robust retry |
| P-Care/BPJS | Government | High | Critical for claims |
| Mobile JKN | Government | Medium | Q3 roadmap |
| Jurnal | Accounting | Medium | Native integration planned |
| Xendit | Payments | High | Must be reliable |
| Xero | Accounting | Low | International clients |
| Talenta | HR | Low | Future |
| Zoho Desk | Support | Low | Future |
| Kyoo | Queue | Medium | Existing integration |

---

# Part 2: Architecture

## 2.1 Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           YOUR SERVICE CODE                                  │
│                                                                              │
│   client := apiclient.New(config)                                           │
│   response, err := client.Post(ctx, "/Patient", body)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API CLIENT PACKAGE                                   │
│                         /pkg/apiclient/                                      │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         REQUEST PIPELINE                                 ││
│  │                                                                          ││
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ││
│  │  │  Log    │──▶│ Circuit │──▶│  Retry  │──▶│ Timeout │──▶│  HTTP   │   ││
│  │  │ Request │   │ Breaker │   │  Logic  │   │ Context │   │ Execute │   ││
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘   ││
│  │       │             │             │             │             │         ││
│  │       │             │             │             │             │         ││
│  │       ▼             ▼             ▼             ▼             ▼         ││
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ││
│  │  │  Log    │◀──│ Record  │◀──│  Check  │◀──│  Parse  │◀──│   HTTP  │   ││
│  │  │Response │   │ Metrics │   │ Success │   │Response │   │Response │   ││
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘   ││
│  │                                   │                                      ││
│  │                                   │ On permanent failure                 ││
│  │                                   ▼                                      ││
│  │                            ┌─────────────┐                               ││
│  │                            │ Dead Letter │                               ││
│  │                            │    Queue    │                               ││
│  │                            └─────────────┘                               ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTPS
                                    ▼
                         ┌──────────────────────┐
                         │    External API      │
                         │  (SATU SEHAT, etc.)  │
                         └──────────────────────┘
```

## 2.2 Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| **Config** | Hold all settings (URLs, timeouts, retry policy) | `config.go` |
| **Circuit Breaker** | Prevent calls to failing services | `circuit_breaker.go` |
| **Retry Logic** | Retry with exponential backoff | `retry.go` (uses avast/retry-go) |
| **Logging** | Record requests/responses in JSON | `logging.go` (uses uber-go/zap) |
| **Metrics** | Track success/failure counts, latencies | `metrics.go` |
| **Dead Letter** | Send permanent failures to RabbitMQ | `dead_letter.go` |
| **Errors** | Custom error types with context | `errors.go` |
| **Client** | Main struct, orchestrates pipeline | `client.go` |

## 2.3 File Structure

```
/pkg/apiclient/
├── client.go              # Main client struct and methods
├── client_test.go         # Unit tests for client
├── config.go              # Configuration structs
├── config_test.go         # Config validation tests
├── circuit_breaker.go     # Circuit breaker implementation
├── circuit_breaker_test.go
├── retry.go               # Retry wrapper around avast/retry-go
├── retry_test.go
├── logging.go             # Structured logging
├── metrics.go             # Metrics interface and implementations
├── dead_letter.go         # DLQ integration
├── dead_letter_test.go
├── errors.go              # Custom error types
└── doc.go                 # Package documentation
```

---

# Part 3: Implementation

## 3.1 config.go

```go
package apiclient

import (
    "fmt"
    "time"
)

// Config holds all configuration for an API client instance
type Config struct {
    // Identity
    Name        string `json:"name" validate:"required"`        // e.g., "satu-sehat"
    BaseURL     string `json:"base_url" validate:"required,url"` // e.g., "https://api.satusehat.kemkes.go.id"
    
    // Timeouts
    Timeout         time.Duration `json:"timeout"`           // Per-request timeout
    ConnectTimeout  time.Duration `json:"connect_timeout"`   // Connection establishment timeout
    
    // Retry configuration
    Retry RetryConfig `json:"retry"`
    
    // Circuit breaker configuration
    CircuitBreaker CircuitBreakerConfig `json:"circuit_breaker"`
    
    // Authentication
    Auth AuthConfig `json:"auth"`
    
    // Logging
    LogRequests  bool `json:"log_requests"`  // Log full request body
    LogResponses bool `json:"log_responses"` // Log full response body
}

// RetryConfig defines retry behavior
type RetryConfig struct {
    MaxAttempts     int           `json:"max_attempts"`      // Total attempts (including first)
    InitialInterval time.Duration `json:"initial_interval"`  // First retry delay
    MaxInterval     time.Duration `json:"max_interval"`      // Maximum retry delay
    Multiplier      float64       `json:"multiplier"`        // Backoff multiplier
    
    // HTTP status codes that trigger retry
    RetryableStatuses []int `json:"retryable_statuses"`
}

// CircuitBreakerConfig defines circuit breaker behavior
type CircuitBreakerConfig struct {
    Enabled          bool          `json:"enabled"`
    FailureThreshold int           `json:"failure_threshold"` // Failures before open
    SuccessThreshold int           `json:"success_threshold"` // Successes to close
    Timeout          time.Duration `json:"timeout"`           // Time in open state before half-open
}

// AuthConfig defines authentication method
type AuthConfig struct {
    Type     string            `json:"type"` // "none", "bearer", "basic", "oauth2", "custom"
    Token    string            `json:"token,omitempty"`
    Username string            `json:"username,omitempty"`
    Password string            `json:"password,omitempty"`
    OAuth2   *OAuth2Config     `json:"oauth2,omitempty"`
    Headers  map[string]string `json:"headers,omitempty"` // Custom auth headers
}

// OAuth2Config for OAuth2 authentication
type OAuth2Config struct {
    ClientID     string   `json:"client_id"`
    ClientSecret string   `json:"client_secret"`
    TokenURL     string   `json:"token_url"`
    Scopes       []string `json:"scopes,omitempty"`
}

// DefaultConfig returns sensible defaults for most APIs
func DefaultConfig(name, baseURL string) Config {
    return Config{
        Name:           name,
        BaseURL:        baseURL,
        Timeout:        30 * time.Second,
        ConnectTimeout: 10 * time.Second,
        Retry: RetryConfig{
            MaxAttempts:       5,
            InitialInterval:   1 * time.Second,
            MaxInterval:       30 * time.Second,
            Multiplier:        2.0,
            RetryableStatuses: []int{408, 429, 500, 502, 503, 504},
        },
        CircuitBreaker: CircuitBreakerConfig{
            Enabled:          true,
            FailureThreshold: 5,
            SuccessThreshold: 2,
            Timeout:          30 * time.Second,
        },
        Auth: AuthConfig{
            Type: "none",
        },
        LogRequests:  true,
        LogResponses: true,
    }
}

// Validate checks configuration is valid
func (c Config) Validate() error {
    if c.Name == "" {
        return fmt.Errorf("name is required")
    }
    if c.BaseURL == "" {
        return fmt.Errorf("base_url is required")
    }
    if c.Retry.MaxAttempts < 1 {
        return fmt.Errorf("max_attempts must be at least 1")
    }
    if c.Retry.Multiplier < 1 {
        return fmt.Errorf("multiplier must be at least 1")
    }
    return nil
}

// Preset configurations for known APIs

// SatuSehatConfig returns configuration optimized for SATU SEHAT API
func SatuSehatConfig(baseURL, token string) Config {
    cfg := DefaultConfig("satu-sehat", baseURL)
    cfg.Timeout = 60 * time.Second // SATU SEHAT can be slow
    cfg.Retry.MaxAttempts = 5
    cfg.Retry.InitialInterval = 2 * time.Second
    cfg.Retry.MaxInterval = 60 * time.Second
    cfg.CircuitBreaker.FailureThreshold = 10 // More tolerant
    cfg.CircuitBreaker.Timeout = 60 * time.Second
    cfg.Auth = AuthConfig{
        Type:  "bearer",
        Token: token,
    }
    return cfg
}

// PCareConfig returns configuration optimized for P-Care API
func PCareConfig(baseURL, consumerID, consumerSecret string) Config {
    cfg := DefaultConfig("pcare", baseURL)
    cfg.Timeout = 45 * time.Second
    cfg.Retry.MaxAttempts = 10 // P-Care is critical, try harder
    cfg.Retry.InitialInterval = 500 * time.Millisecond
    cfg.CircuitBreaker.FailureThreshold = 10
    cfg.Auth = AuthConfig{
        Type: "custom",
        Headers: map[string]string{
            "X-cons-id":  consumerID,
            "X-secretkey": consumerSecret,
        },
    }
    return cfg
}

// XenditConfig returns configuration for Xendit payment API
func XenditConfig(apiKey string) Config {
    cfg := DefaultConfig("xendit", "https://api.xendit.co")
    cfg.Timeout = 30 * time.Second
    cfg.Retry.MaxAttempts = 3 // Don't retry payments too aggressively
    cfg.CircuitBreaker.FailureThreshold = 3
    cfg.Auth = AuthConfig{
        Type:     "basic",
        Username: apiKey,
        Password: "",
    }
    return cfg
}
```

## 3.2 errors.go

```go
package apiclient

import (
    "fmt"
    "time"
)

// APIError represents a failed API call with full context
type APIError struct {
    // Identity
    ClientName string `json:"client_name"`
    RequestID  string `json:"request_id"`
    
    // Request details
    Method string `json:"method"`
    URL    string `json:"url"`
    
    // Response details
    StatusCode   int    `json:"status_code,omitempty"`
    ResponseBody string `json:"response_body,omitempty"`
    
    // Attempt tracking
    Attempts    int       `json:"attempts"`
    FirstAttempt time.Time `json:"first_attempt"`
    LastAttempt  time.Time `json:"last_attempt"`
    
    // Error details
    Err         error  `json:"-"`
    ErrMessage  string `json:"error_message"`
    
    // Flags
    Retryable    bool `json:"retryable"`
    CircuitOpen  bool `json:"circuit_open"`
    Timeout      bool `json:"timeout"`
}

func (e *APIError) Error() string {
    if e.CircuitOpen {
        return fmt.Sprintf("[%s] circuit breaker open for %s %s (request_id=%s)",
            e.ClientName, e.Method, e.URL, e.RequestID)
    }
    if e.Timeout {
        return fmt.Sprintf("[%s] timeout after %d attempts for %s %s (request_id=%s)",
            e.ClientName, e.Attempts, e.Method, e.URL, e.RequestID)
    }
    return fmt.Sprintf("[%s] failed after %d attempts: %s %s status=%d (request_id=%s): %s",
        e.ClientName, e.Attempts, e.Method, e.URL, e.StatusCode, e.RequestID, e.ErrMessage)
}

func (e *APIError) Unwrap() error {
    return e.Err
}

// IsRetryable returns true if the error might succeed on retry
func (e *APIError) IsRetryable() bool {
    return e.Retryable
}

// IsCircuitOpen returns true if circuit breaker prevented the call
func (e *APIError) IsCircuitOpen() bool {
    return e.CircuitOpen
}

// IsTimeout returns true if the request timed out
func (e *APIError) IsTimeout() bool {
    return e.Timeout
}

// NewAPIError creates an APIError with common fields populated
func NewAPIError(clientName, requestID, method, url string, err error) *APIError {
    return &APIError{
        ClientName: clientName,
        RequestID:  requestID,
        Method:     method,
        URL:        url,
        Err:        err,
        ErrMessage: err.Error(),
    }
}
```

## 3.3 circuit_breaker.go

```go
package apiclient

import (
    "sync"
    "time"
)

// CircuitState represents the current state of the circuit breaker
type CircuitState int

const (
    StateClosed   CircuitState = iota // Normal operation - requests allowed
    StateOpen                          // Failing - requests blocked
    StateHalfOpen                      // Testing - limited requests allowed
)

func (s CircuitState) String() string {
    switch s {
    case StateClosed:
        return "closed"
    case StateOpen:
        return "open"
    case StateHalfOpen:
        return "half-open"
    default:
        return "unknown"
    }
}

// CircuitBreaker implements the circuit breaker pattern
type CircuitBreaker struct {
    mu sync.RWMutex
    
    config CircuitBreakerConfig
    state  CircuitState
    
    failures        int
    successes       int
    lastFailure     time.Time
    lastStateChange time.Time
}

// NewCircuitBreaker creates a new circuit breaker with the given config
func NewCircuitBreaker(config CircuitBreakerConfig) *CircuitBreaker {
    return &CircuitBreaker{
        config:          config,
        state:           StateClosed,
        lastStateChange: time.Now(),
    }
}

// Allow returns true if the circuit breaker allows the request
func (cb *CircuitBreaker) Allow() bool {
    if !cb.config.Enabled {
        return true
    }
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    switch cb.state {
    case StateClosed:
        return true
        
    case StateOpen:
        // Check if timeout has passed - transition to half-open
        if time.Since(cb.lastStateChange) > cb.config.Timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            cb.lastStateChange = time.Now()
            return true
        }
        return false
        
    case StateHalfOpen:
        // Allow limited requests through to test recovery
        return true
    }
    
    return false
}

// RecordSuccess records a successful request
func (cb *CircuitBreaker) RecordSuccess() {
    if !cb.config.Enabled {
        return
    }
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    switch cb.state {
    case StateHalfOpen:
        cb.successes++
        if cb.successes >= cb.config.SuccessThreshold {
            // Recovered - close the circuit
            cb.state = StateClosed
            cb.failures = 0
            cb.lastStateChange = time.Now()
        }
    case StateClosed:
        // Reset failure count on success
        cb.failures = 0
    }
}

// RecordFailure records a failed request
func (cb *CircuitBreaker) RecordFailure() {
    if !cb.config.Enabled {
        return
    }
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    cb.failures++
    cb.lastFailure = time.Now()
    
    switch cb.state {
    case StateClosed:
        if cb.failures >= cb.config.FailureThreshold {
            // Too many failures - open the circuit
            cb.state = StateOpen
            cb.lastStateChange = time.Now()
        }
    case StateHalfOpen:
        // Failure during recovery - back to open
        cb.state = StateOpen
        cb.lastStateChange = time.Now()
    }
}

// State returns the current circuit state
func (cb *CircuitBreaker) State() CircuitState {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return cb.state
}

// Stats returns current circuit breaker statistics
func (cb *CircuitBreaker) Stats() CircuitBreakerStats {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return CircuitBreakerStats{
        State:           cb.state.String(),
        Failures:        cb.failures,
        Successes:       cb.successes,
        LastFailure:     cb.lastFailure,
        LastStateChange: cb.lastStateChange,
    }
}

// CircuitBreakerStats holds circuit breaker statistics
type CircuitBreakerStats struct {
    State           string    `json:"state"`
    Failures        int       `json:"failures"`
    Successes       int       `json:"successes"`
    LastFailure     time.Time `json:"last_failure,omitempty"`
    LastStateChange time.Time `json:"last_state_change"`
}
```

## 3.4 dead_letter.go

```go
package apiclient

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
    
    amqp "github.com/rabbitmq/amqp091-go"
)

// FailedRequest represents a request that failed after all retries
// This is what gets sent to the dead letter queue
type FailedRequest struct {
    // Identity
    ID         string `json:"id"`
    ClientName string `json:"client_name"`
    
    // Original request
    Method  string            `json:"method"`
    URL     string            `json:"url"`
    Headers map[string]string `json:"headers,omitempty"`
    Body    []byte            `json:"body,omitempty"`
    
    // Failure context
    Attempts     int       `json:"attempts"`
    LastStatus   int       `json:"last_status,omitempty"`
    LastError    string    `json:"last_error"`
    FirstAttempt time.Time `json:"first_attempt"`
    LastAttempt  time.Time `json:"last_attempt"`
    
    // For retry worker
    RetryCount int       `json:"retry_count"`
    NextRetry  time.Time `json:"next_retry,omitempty"`
    
    // Context from original call (for debugging)
    CorrelationID string                 `json:"correlation_id,omitempty"`
    Metadata      map[string]interface{} `json:"metadata,omitempty"`
}

// DeadLetterQueue is the interface for sending failed requests
type DeadLetterQueue interface {
    Send(ctx context.Context, req FailedRequest) error
    Close() error
}

// RabbitMQDeadLetter implements DeadLetterQueue using RabbitMQ
type RabbitMQDeadLetter struct {
    channel    *amqp.Channel
    exchange   string
    routingKey string
}

// NewRabbitMQDeadLetter creates a new RabbitMQ-based dead letter queue
func NewRabbitMQDeadLetter(conn *amqp.Connection, exchange, routingKey string) (*RabbitMQDeadLetter, error) {
    ch, err := conn.Channel()
    if err != nil {
        return nil, fmt.Errorf("failed to open channel: %w", err)
    }
    
    // Declare exchange if it doesn't exist
    err = ch.ExchangeDeclare(
        exchange, // name
        "topic",  // type
        true,     // durable
        false,    // auto-deleted
        false,    // internal
        false,    // no-wait
        nil,      // arguments
    )
    if err != nil {
        return nil, fmt.Errorf("failed to declare exchange: %w", err)
    }
    
    return &RabbitMQDeadLetter{
        channel:    ch,
        exchange:   exchange,
        routingKey: routingKey,
    }, nil
}

// Send publishes a failed request to the dead letter queue
func (r *RabbitMQDeadLetter) Send(ctx context.Context, req FailedRequest) error {
    body, err := json.Marshal(req)
    if err != nil {
        return fmt.Errorf("failed to marshal dead letter: %w", err)
    }
    
    return r.channel.PublishWithContext(
        ctx,
        r.exchange,
        r.routingKey,
        false, // mandatory
        false, // immediate
        amqp.Publishing{
            ContentType:  "application/json",
            DeliveryMode: amqp.Persistent,
            Timestamp:    time.Now(),
            Body:         body,
        },
    )
}

// Close closes the channel
func (r *RabbitMQDeadLetter) Close() error {
    return r.channel.Close()
}

// NoOpDeadLetter is a no-op implementation for testing or when DLQ is not needed
type NoOpDeadLetter struct{}

func (n *NoOpDeadLetter) Send(ctx context.Context, req FailedRequest) error {
    return nil
}

func (n *NoOpDeadLetter) Close() error {
    return nil
}

// MemoryDeadLetter stores failed requests in memory (for testing)
type MemoryDeadLetter struct {
    requests []FailedRequest
}

func NewMemoryDeadLetter() *MemoryDeadLetter {
    return &MemoryDeadLetter{
        requests: make([]FailedRequest, 0),
    }
}

func (m *MemoryDeadLetter) Send(ctx context.Context, req FailedRequest) error {
    m.requests = append(m.requests, req)
    return nil
}

func (m *MemoryDeadLetter) Close() error {
    return nil
}

func (m *MemoryDeadLetter) GetAll() []FailedRequest {
    return m.requests
}

func (m *MemoryDeadLetter) Count() int {
    return len(m.requests)
}
```

## 3.5 logging.go

```go
package apiclient

import (
    "context"
    "time"
    
    "go.uber.org/zap"
)

// Logger interface for structured logging
type Logger interface {
    Info(ctx context.Context, msg string, fields map[string]interface{})
    Warn(ctx context.Context, msg string, fields map[string]interface{})
    Error(ctx context.Context, msg string, fields map[string]interface{})
    Debug(ctx context.Context, msg string, fields map[string]interface{})
}

// ZapLogger wraps zap.Logger to implement our Logger interface
type ZapLogger struct {
    logger *zap.Logger
}

// NewZapLogger creates a new ZapLogger
func NewZapLogger(logger *zap.Logger) *ZapLogger {
    return &ZapLogger{logger: logger}
}

func (z *ZapLogger) Info(ctx context.Context, msg string, fields map[string]interface{}) {
    z.logger.Info(msg, mapToZapFields(fields)...)
}

func (z *ZapLogger) Warn(ctx context.Context, msg string, fields map[string]interface{}) {
    z.logger.Warn(msg, mapToZapFields(fields)...)
}

func (z *ZapLogger) Error(ctx context.Context, msg string, fields map[string]interface{}) {
    z.logger.Error(msg, mapToZapFields(fields)...)
}

func (z *ZapLogger) Debug(ctx context.Context, msg string, fields map[string]interface{}) {
    z.logger.Debug(msg, mapToZapFields(fields)...)
}

func mapToZapFields(m map[string]interface{}) []zap.Field {
    fields := make([]zap.Field, 0, len(m))
    for k, v := range m {
        fields = append(fields, zap.Any(k, v))
    }
    return fields
}

// NoOpLogger does nothing (for testing)
type NoOpLogger struct{}

func (n *NoOpLogger) Info(ctx context.Context, msg string, fields map[string]interface{})  {}
func (n *NoOpLogger) Warn(ctx context.Context, msg string, fields map[string]interface{})  {}
func (n *NoOpLogger) Error(ctx context.Context, msg string, fields map[string]interface{}) {}
func (n *NoOpLogger) Debug(ctx context.Context, msg string, fields map[string]interface{}) {}

// RequestLog represents a logged API request
type RequestLog struct {
    Timestamp   time.Time         `json:"timestamp"`
    ClientName  string            `json:"client_name"`
    RequestID   string            `json:"request_id"`
    Method      string            `json:"method"`
    URL         string            `json:"url"`
    Headers     map[string]string `json:"headers,omitempty"`
    Body        string            `json:"body,omitempty"`
}

// ResponseLog represents a logged API response
type ResponseLog struct {
    Timestamp   time.Time `json:"timestamp"`
    ClientName  string    `json:"client_name"`
    RequestID   string    `json:"request_id"`
    StatusCode  int       `json:"status_code"`
    Duration    int64     `json:"duration_ms"`
    Attempts    int       `json:"attempts"`
    Body        string    `json:"body,omitempty"`
    Error       string    `json:"error,omitempty"`
}
```

## 3.6 metrics.go

```go
package apiclient

import (
    "sync"
    "time"
)

// Metrics interface for recording API metrics
type Metrics interface {
    RecordRequest(clientName, method string, statusCode int, duration time.Duration, success bool)
    RecordCircuitState(clientName string, state string)
    RecordDeadLetter(clientName string)
}

// NoOpMetrics does nothing (default)
type NoOpMetrics struct{}

func (n *NoOpMetrics) RecordRequest(clientName, method string, statusCode int, duration time.Duration, success bool) {}
func (n *NoOpMetrics) RecordCircuitState(clientName string, state string) {}
func (n *NoOpMetrics) RecordDeadLetter(clientName string) {}

// InMemoryMetrics stores metrics in memory (for testing/development)
type InMemoryMetrics struct {
    mu       sync.RWMutex
    requests []RequestMetric
}

type RequestMetric struct {
    Timestamp  time.Time     `json:"timestamp"`
    ClientName string        `json:"client_name"`
    Method     string        `json:"method"`
    StatusCode int           `json:"status_code"`
    Duration   time.Duration `json:"duration"`
    Success    bool          `json:"success"`
}

func NewInMemoryMetrics() *InMemoryMetrics {
    return &InMemoryMetrics{
        requests: make([]RequestMetric, 0),
    }
}

func (m *InMemoryMetrics) RecordRequest(clientName, method string, statusCode int, duration time.Duration, success bool) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.requests = append(m.requests, RequestMetric{
        Timestamp:  time.Now(),
        ClientName: clientName,
        Method:     method,
        StatusCode: statusCode,
        Duration:   duration,
        Success:    success,
    })
}

func (m *InMemoryMetrics) RecordCircuitState(clientName string, state string) {}
func (m *InMemoryMetrics) RecordDeadLetter(clientName string) {}

func (m *InMemoryMetrics) GetAll() []RequestMetric {
    m.mu.RLock()
    defer m.mu.RUnlock()
    result := make([]RequestMetric, len(m.requests))
    copy(result, m.requests)
    return result
}

func (m *InMemoryMetrics) SuccessRate(clientName string) float64 {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var total, success int
    for _, r := range m.requests {
        if r.ClientName == clientName {
            total++
            if r.Success {
                success++
            }
        }
    }
    if total == 0 {
        return 1.0
    }
    return float64(success) / float64(total)
}
```

## 3.7 client.go

```go
package apiclient

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
    
    "github.com/avast/retry-go/v4"
    "github.com/google/uuid"
)

// Client is a resilient HTTP client for external APIs
type Client struct {
    name           string
    baseURL        string
    httpClient     *http.Client
    config         Config
    circuitBreaker *CircuitBreaker
    deadLetter     DeadLetterQueue
    logger         Logger
    metrics        Metrics
}

// Response wraps the HTTP response with additional metadata
type Response struct {
    StatusCode int
    Body       []byte
    Headers    http.Header
    Duration   time.Duration
    Attempts   int
    RequestID  string
}

// Request holds all parameters for an API request
type Request struct {
    Method   string
    Path     string
    Body     interface{}
    Headers  map[string]string
    Metadata map[string]interface{} // For logging/debugging
}

// New creates a new API client with the given configuration
func New(config Config) (*Client, error) {
    if err := config.Validate(); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    
    return &Client{
        name:    config.Name,
        baseURL: config.BaseURL,
        httpClient: &http.Client{
            Timeout: config.Timeout,
        },
        config:         config,
        circuitBreaker: NewCircuitBreaker(config.CircuitBreaker),
        deadLetter:     &NoOpDeadLetter{},
        logger:         &NoOpLogger{},
        metrics:        &NoOpMetrics{},
    }, nil
}

// WithDeadLetterQueue sets the dead letter queue
func (c *Client) WithDeadLetterQueue(dlq DeadLetterQueue) *Client {
    c.deadLetter = dlq
    return c
}

// WithLogger sets the logger
func (c *Client) WithLogger(logger Logger) *Client {
    c.logger = logger
    return c
}

// WithMetrics sets the metrics recorder
func (c *Client) WithMetrics(metrics Metrics) *Client {
    c.metrics = metrics
    return c
}

// Do executes an API request with retry and circuit breaker
func (c *Client) Do(ctx context.Context, req Request) (*Response, error) {
    requestID := uuid.New().String()
    fullURL := c.baseURL + req.Path
    startTime := time.Now()
    var attempts int
    var lastError error
    var lastStatus int
    
    // Check circuit breaker
    if !c.circuitBreaker.Allow() {
        c.logger.Warn(ctx, "Circuit breaker open", map[string]interface{}{
            "client":     c.name,
            "request_id": requestID,
            "url":        fullURL,
        })
        
        c.metrics.RecordRequest(c.name, req.Method, 0, 0, false)
        c.metrics.RecordCircuitState(c.name, "open")
        
        return nil, &APIError{
            ClientName:  c.name,
            RequestID:   requestID,
            Method:      req.Method,
            URL:         fullURL,
            CircuitOpen: true,
        }
    }
    
    // Prepare body
    var bodyBytes []byte
    if req.Body != nil {
        var err error
        bodyBytes, err = json.Marshal(req.Body)
        if err != nil {
            return nil, fmt.Errorf("failed to marshal request body: %w", err)
        }
    }
    
    // Log request
    c.logger.Info(ctx, "API request starting", map[string]interface{}{
        "client":     c.name,
        "request_id": requestID,
        "method":     req.Method,
        "url":        fullURL,
    })
    
    var response *Response
    
    // Execute with retry
    err := retry.Do(
        func() error {
            attempts++
            
            // Create body reader for this attempt
            var bodyReader io.Reader
            if bodyBytes != nil {
                bodyReader = bytes.NewReader(bodyBytes)
            }
            
            httpReq, err := http.NewRequestWithContext(ctx, req.Method, fullURL, bodyReader)
            if err != nil {
                return retry.Unrecoverable(err)
            }
            
            // Set headers
            httpReq.Header.Set("Content-Type", "application/json")
            httpReq.Header.Set("X-Request-ID", requestID)
            for k, v := range req.Headers {
                httpReq.Header.Set(k, v)
            }
            
            // Apply authentication
            c.applyAuth(httpReq)
            
            // Execute request
            attemptStart := time.Now()
            resp, err := c.httpClient.Do(httpReq)
            if err != nil {
                lastError = err
                return err // Will be retried
            }
            defer resp.Body.Close()
            
            lastStatus = resp.StatusCode
            respBody, err := io.ReadAll(resp.Body)
            if err != nil {
                lastError = err
                return err
            }
            
            // Check if retryable status
            if c.isRetryableStatus(resp.StatusCode) {
                lastError = fmt.Errorf("received status %d", resp.StatusCode)
                return lastError
            }
            
            // Success!
            response = &Response{
                StatusCode: resp.StatusCode,
                Body:       respBody,
                Headers:    resp.Header,
                Duration:   time.Since(attemptStart),
                Attempts:   attempts,
                RequestID:  requestID,
            }
            
            return nil
        },
        retry.Attempts(uint(c.config.Retry.MaxAttempts)),
        retry.Delay(c.config.Retry.InitialInterval),
        retry.MaxDelay(c.config.Retry.MaxInterval),
        retry.DelayType(retry.BackOffDelay),
        retry.OnRetry(func(n uint, err error) {
            c.logger.Warn(ctx, "Retrying request", map[string]interface{}{
                "client":     c.name,
                "request_id": requestID,
                "attempt":    n + 1,
                "max":        c.config.Retry.MaxAttempts,
                "error":      err.Error(),
            })
        }),
        retry.Context(ctx),
    )
    
    duration := time.Since(startTime)
    
    if err != nil {
        // All retries failed
        c.circuitBreaker.RecordFailure()
        
        c.logger.Error(ctx, "API request failed", map[string]interface{}{
            "client":     c.name,
            "request_id": requestID,
            "attempts":   attempts,
            "duration":   duration.Milliseconds(),
            "error":      err.Error(),
        })
        
        c.metrics.RecordRequest(c.name, req.Method, lastStatus, duration, false)
        
        // Send to dead letter queue
        failedReq := FailedRequest{
            ID:           requestID,
            ClientName:   c.name,
            Method:       req.Method,
            URL:          fullURL,
            Headers:      req.Headers,
            Body:         bodyBytes,
            Attempts:     attempts,
            LastStatus:   lastStatus,
            LastError:    err.Error(),
            FirstAttempt: startTime,
            LastAttempt:  time.Now(),
            Metadata:     req.Metadata,
        }
        
        if dlqErr := c.deadLetter.Send(ctx, failedReq); dlqErr != nil {
            c.logger.Error(ctx, "Failed to send to dead letter queue", map[string]interface{}{
                "request_id": requestID,
                "error":      dlqErr.Error(),
            })
        } else {
            c.metrics.RecordDeadLetter(c.name)
        }
        
        return nil, &APIError{
            ClientName:   c.name,
            RequestID:    requestID,
            Method:       req.Method,
            URL:          fullURL,
            StatusCode:   lastStatus,
            Attempts:     attempts,
            FirstAttempt: startTime,
            LastAttempt:  time.Now(),
            Err:          err,
            ErrMessage:   err.Error(),
            Retryable:    true,
        }
    }
    
    // Success
    c.circuitBreaker.RecordSuccess()
    
    c.logger.Info(ctx, "API request completed", map[string]interface{}{
        "client":      c.name,
        "request_id":  requestID,
        "status":      response.StatusCode,
        "attempts":    attempts,
        "duration_ms": duration.Milliseconds(),
    })
    
    c.metrics.RecordRequest(c.name, req.Method, response.StatusCode, duration, true)
    
    return response, nil
}

// Convenience methods

func (c *Client) Get(ctx context.Context, path string, headers map[string]string) (*Response, error) {
    return c.Do(ctx, Request{Method: "GET", Path: path, Headers: headers})
}

func (c *Client) Post(ctx context.Context, path string, body interface{}, headers map[string]string) (*Response, error) {
    return c.Do(ctx, Request{Method: "POST", Path: path, Body: body, Headers: headers})
}

func (c *Client) Put(ctx context.Context, path string, body interface{}, headers map[string]string) (*Response, error) {
    return c.Do(ctx, Request{Method: "PUT", Path: path, Body: body, Headers: headers})
}

func (c *Client) Patch(ctx context.Context, path string, body interface{}, headers map[string]string) (*Response, error) {
    return c.Do(ctx, Request{Method: "PATCH", Path: path, Body: body, Headers: headers})
}

func (c *Client) Delete(ctx context.Context, path string, headers map[string]string) (*Response, error) {
    return c.Do(ctx, Request{Method: "DELETE", Path: path, Headers: headers})
}

// Helper methods

func (c *Client) isRetryableStatus(status int) bool {
    for _, s := range c.config.Retry.RetryableStatuses {
        if s == status {
            return true
        }
    }
    return false
}

func (c *Client) applyAuth(req *http.Request) {
    switch c.config.Auth.Type {
    case "bearer":
        req.Header.Set("Authorization", "Bearer "+c.config.Auth.Token)
    case "basic":
        req.SetBasicAuth(c.config.Auth.Username, c.config.Auth.Password)
    case "custom":
        for k, v := range c.config.Auth.Headers {
            req.Header.Set(k, v)
        }
    case "oauth2":
        // OAuth2 requires token refresh - implement separately
        // For now, assume token is already obtained
        if c.config.Auth.Token != "" {
            req.Header.Set("Authorization", "Bearer "+c.config.Auth.Token)
        }
    }
}

// Stats returns current client statistics
func (c *Client) Stats() ClientStats {
    return ClientStats{
        Name:           c.name,
        BaseURL:        c.baseURL,
        CircuitBreaker: c.circuitBreaker.Stats(),
    }
}

type ClientStats struct {
    Name           string              `json:"name"`
    BaseURL        string              `json:"base_url"`
    CircuitBreaker CircuitBreakerStats `json:"circuit_breaker"`
}
```

## 3.8 client_test.go

```go
package apiclient

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "sync/atomic"
    "testing"
    "time"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestClient_SuccessfulRequest(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "application/json", r.Header.Get("Content-Type"))
        assert.NotEmpty(t, r.Header.Get("X-Request-ID"))
        
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    client, err := New(config)
    require.NoError(t, err)
    
    resp, err := client.Get(context.Background(), "/test", nil)
    
    require.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
    assert.Equal(t, 1, resp.Attempts)
    assert.NotEmpty(t, resp.RequestID)
}

func TestClient_RetryOnServerError(t *testing.T) {
    var attempts int32
    
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        count := atomic.AddInt32(&attempts, 1)
        if count < 3 {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.Retry.InitialInterval = 10 * time.Millisecond
    client, err := New(config)
    require.NoError(t, err)
    
    resp, err := client.Get(context.Background(), "/test", nil)
    
    require.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
    assert.Equal(t, 3, resp.Attempts)
}

func TestClient_ExhaustedRetries(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusServiceUnavailable)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.Retry.MaxAttempts = 3
    config.Retry.InitialInterval = 1 * time.Millisecond
    config.CircuitBreaker.Enabled = false
    client, err := New(config)
    require.NoError(t, err)
    
    _, err = client.Get(context.Background(), "/test", nil)
    
    require.Error(t, err)
    apiErr, ok := err.(*APIError)
    require.True(t, ok)
    assert.Equal(t, 3, apiErr.Attempts)
    assert.Equal(t, 503, apiErr.StatusCode)
}

func TestClient_CircuitBreakerOpens(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusServiceUnavailable)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.Retry.MaxAttempts = 1
    config.Retry.InitialInterval = 1 * time.Millisecond
    config.CircuitBreaker.FailureThreshold = 3
    config.CircuitBreaker.Timeout = 100 * time.Millisecond
    client, err := New(config)
    require.NoError(t, err)
    
    // Cause failures to open circuit
    for i := 0; i < 3; i++ {
        _, _ = client.Get(context.Background(), "/test", nil)
    }
    
    // Next request should fail immediately due to open circuit
    _, err = client.Get(context.Background(), "/test", nil)
    require.Error(t, err)
    
    apiErr, ok := err.(*APIError)
    require.True(t, ok)
    assert.True(t, apiErr.IsCircuitOpen())
}

func TestClient_CircuitBreakerRecovers(t *testing.T) {
    var attempts int32
    recoveryPoint := int32(5)
    
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        count := atomic.AddInt32(&attempts, 1)
        if count < recoveryPoint {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.Retry.MaxAttempts = 1
    config.Retry.InitialInterval = 1 * time.Millisecond
    config.CircuitBreaker.FailureThreshold = 2
    config.CircuitBreaker.SuccessThreshold = 2
    config.CircuitBreaker.Timeout = 50 * time.Millisecond
    client, err := New(config)
    require.NoError(t, err)
    
    // Cause failures to open circuit
    for i := 0; i < 3; i++ {
        _, _ = client.Get(context.Background(), "/test", nil)
    }
    
    // Circuit should be open
    assert.Equal(t, StateOpen, client.circuitBreaker.State())
    
    // Wait for timeout
    time.Sleep(60 * time.Millisecond)
    
    // Circuit should be half-open, allow request through
    resp, err := client.Get(context.Background(), "/test", nil)
    require.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}

func TestClient_DeadLetterQueueOnFailure(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusServiceUnavailable)
    }))
    defer server.Close()
    
    dlq := NewMemoryDeadLetter()
    
    config := DefaultConfig("test-api", server.URL)
    config.Retry.MaxAttempts = 2
    config.Retry.InitialInterval = 1 * time.Millisecond
    config.CircuitBreaker.Enabled = false
    
    client, err := New(config)
    require.NoError(t, err)
    client.WithDeadLetterQueue(dlq)
    
    _, err = client.Get(context.Background(), "/test", nil)
    
    require.Error(t, err)
    assert.Equal(t, 1, dlq.Count())
    
    failed := dlq.GetAll()[0]
    assert.Equal(t, "test-api", failed.ClientName)
    assert.Equal(t, "GET", failed.Method)
    assert.Equal(t, 2, failed.Attempts)
}

func TestClient_PostWithBody(t *testing.T) {
    var receivedBody map[string]interface{}
    
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "POST", r.Method)
        json.NewDecoder(r.Body).Decode(&receivedBody)
        w.WriteHeader(http.StatusCreated)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    client, err := New(config)
    require.NoError(t, err)
    
    body := map[string]interface{}{
        "name":  "Test Patient",
        "nik":   "1234567890123456",
    }
    
    resp, err := client.Post(context.Background(), "/patients", body, nil)
    
    require.NoError(t, err)
    assert.Equal(t, 201, resp.StatusCode)
    assert.Equal(t, "Test Patient", receivedBody["name"])
}

func TestClient_CustomHeaders(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "custom-value", r.Header.Get("X-Custom-Header"))
        w.WriteHeader(http.StatusOK)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    client, err := New(config)
    require.NoError(t, err)
    
    headers := map[string]string{
        "X-Custom-Header": "custom-value",
    }
    
    _, err = client.Get(context.Background(), "/test", headers)
    require.NoError(t, err)
}

func TestClient_BearerAuth(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        auth := r.Header.Get("Authorization")
        assert.Equal(t, "Bearer test-token", auth)
        w.WriteHeader(http.StatusOK)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.Auth = AuthConfig{
        Type:  "bearer",
        Token: "test-token",
    }
    client, err := New(config)
    require.NoError(t, err)
    
    _, err = client.Get(context.Background(), "/test", nil)
    require.NoError(t, err)
}

func TestClient_ContextCancellation(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(100 * time.Millisecond)
        w.WriteHeader(http.StatusOK)
    }))
    defer server.Close()
    
    config := DefaultConfig("test-api", server.URL)
    config.CircuitBreaker.Enabled = false
    client, err := New(config)
    require.NoError(t, err)
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
    defer cancel()
    
    _, err = client.Get(ctx, "/test", nil)
    require.Error(t, err)
}
```

---

# Part 4: Usage Examples

## 4.1 SATU SEHAT Client

```go
// /internal/integrations/satusehat/client.go
package satusehat

import (
    "context"
    "encoding/json"
    "fmt"
    
    "github.com/kalpahealth/wellmed/pkg/apiclient"
)

type Client struct {
    api *apiclient.Client
}

func NewClient(baseURL, token string, dlq apiclient.DeadLetterQueue, logger apiclient.Logger) (*Client, error) {
    config := apiclient.SatuSehatConfig(baseURL, token)
    
    client, err := apiclient.New(config)
    if err != nil {
        return nil, err
    }
    
    client.WithDeadLetterQueue(dlq).WithLogger(logger)
    
    return &Client{api: client}, nil
}

func (c *Client) CreatePatient(ctx context.Context, patient Patient) (*PatientResponse, error) {
    resp, err := c.api.Post(ctx, "/fhir-r4/v1/Patient", patient, nil)
    if err != nil {
        return nil, fmt.Errorf("create patient failed: %w", err)
    }
    
    if resp.StatusCode != 201 {
        return nil, fmt.Errorf("unexpected status %d: %s", resp.StatusCode, string(resp.Body))
    }
    
    var result PatientResponse
    if err := json.Unmarshal(resp.Body, &result); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    return &result, nil
}

func (c *Client) CreateEncounter(ctx context.Context, encounter Encounter) (*EncounterResponse, error) {
    resp, err := c.api.Post(ctx, "/fhir-r4/v1/Encounter", encounter, nil)
    if err != nil {
        return nil, fmt.Errorf("create encounter failed: %w", err)
    }
    
    // Handle specific error codes
    switch resp.StatusCode {
    case 201:
        // Success
    case 400:
        return nil, fmt.Errorf("invalid encounter data: %s", string(resp.Body))
    case 422:
        return nil, fmt.Errorf("unprocessable encounter: %s", string(resp.Body))
    default:
        return nil, fmt.Errorf("unexpected status %d: %s", resp.StatusCode, string(resp.Body))
    }
    
    var result EncounterResponse
    if err := json.Unmarshal(resp.Body, &result); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    return &result, nil
}

func (c *Client) GetPatientByNIK(ctx context.Context, nik string) (*Patient, error) {
    resp, err := c.api.Get(ctx, "/fhir-r4/v1/Patient?identifier=https://fhir.kemkes.go.id/id/nik|"+nik, nil)
    if err != nil {
        return nil, fmt.Errorf("get patient failed: %w", err)
    }
    
    if resp.StatusCode == 404 {
        return nil, nil // Not found
    }
    
    if resp.StatusCode != 200 {
        return nil, fmt.Errorf("unexpected status %d", resp.StatusCode)
    }
    
    var bundle Bundle
    if err := json.Unmarshal(resp.Body, &bundle); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    if len(bundle.Entry) == 0 {
        return nil, nil // Not found
    }
    
    return &bundle.Entry[0].Resource, nil
}
```

## 4.2 Initializing in Main Application

```go
// /cmd/gateway/main.go
package main

import (
    "github.com/kalpahealth/wellmed/internal/integrations/satusehat"
    "github.com/kalpahealth/wellmed/pkg/apiclient"
    
    amqp "github.com/rabbitmq/amqp091-go"
    "go.uber.org/zap"
)

func main() {
    // Setup logging
    logger, _ := zap.NewProduction()
    apiLogger := apiclient.NewZapLogger(logger)
    
    // Setup RabbitMQ connection
    conn, _ := amqp.Dial("amqp://localhost:5672")
    
    // Setup dead letter queue
    dlq, _ := apiclient.NewRabbitMQDeadLetter(conn, "dlq", "api.satu_sehat")
    defer dlq.Close()
    
    // Create SATU SEHAT client
    satuSehatClient, err := satusehat.NewClient(
        "https://api.satusehat.kemkes.go.id",
        os.Getenv("SATU_SEHAT_TOKEN"),
        dlq,
        apiLogger,
    )
    if err != nil {
        logger.Fatal("Failed to create SATU SEHAT client", zap.Error(err))
    }
    
    // Use client...
}
```

---

# Part 5: Architecture Decision Record

## ADR-001: Adopt Reusable API Client Pattern for External Integrations

### Status
**Accepted** - January 2025

### Context

WellMed integrates with multiple external services:
- Government healthcare APIs (SATU SEHAT, P-Care, Mobile JKN)
- Business services (Jurnal, Xendit, Xero, Talenta, Zoho Desk)

Each integration currently has its own HTTP client implementation with varying levels of error handling, retry logic, and logging. This leads to:

1. **Inconsistent reliability**: Some integrations retry on failure, others don't
2. **Duplicate code**: Each integration re-implements similar patterns
3. **Poor observability**: No unified logging or metrics across integrations
4. **Silent failures**: Failed requests may not be captured for later retry
5. **Cascade failures**: One failing API can bring down the entire system

Government APIs in particular (SATU SEHAT, P-Care) are notoriously unstable with frequent timeouts, unexpected errors, and undocumented behavior changes.

### Decision

We will implement a **reusable API client package** (`/pkg/apiclient`) that provides:

1. **Retry with exponential backoff** using `avast/retry-go`
2. **Circuit breaker pattern** to prevent cascade failures
3. **Dead letter queue** integration via RabbitMQ for failed requests
4. **Structured JSON logging** via `uber-go/zap`
5. **Metrics collection** interface for monitoring
6. **Preset configurations** for known APIs (SATU SEHAT, P-Care, Xendit)

All external API integrations will use this client instead of raw `http.Client`.

### Consequences

**Positive:**
- Consistent error handling across all integrations
- No lost requests - failures go to DLQ for retry
- Better observability through unified logging and metrics
- Protection against cascade failures via circuit breaker
- Reduced code duplication
- Easier to add new integrations

**Negative:**
- Learning curve for team to understand the pattern
- Additional abstraction layer
- Need to maintain the shared package
- Slightly more complex than raw HTTP calls

**Risks:**
- Circuit breaker thresholds need tuning per API
- DLQ requires monitoring to avoid unbounded growth
- OAuth2 token refresh not yet implemented

### Compliance

This decision supports WellMed Testing Value #3: "External Robustness from Our Side"

### Alternatives Considered

1. **Continue with per-integration clients**: Rejected due to inconsistency and duplicate code
2. **Use a third-party resilience library** (e.g., go-kit): Rejected as overkill for our needs
3. **Build resilience into Gateway only**: Rejected because not all integrations go through Gateway

---

## ADR Template

Use this template for future Architecture Decision Records:

```markdown
## ADR-XXX: [Title]

### Status
[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

### Context
[What is the issue that we're seeing that is motivating this decision or change?]

### Decision
[What is the change that we're actually proposing or doing?]

### Consequences
**Positive:**
- [List positive outcomes]

**Negative:**
- [List negative outcomes or tradeoffs]

**Risks:**
- [List identified risks]

### Compliance
[How does this decision relate to testing values, security requirements, etc.?]

### Alternatives Considered
[What other options were considered and why were they rejected?]
```

---

# Part 6: Endpoint Policy Groups

## 6.1 Defining Policy Groups

Different external APIs have different reliability characteristics and business criticality. Define policy groups to standardize configuration:

| Policy Group | Use Case | Retry Attempts | Circuit Breaker Threshold | Timeout |
|--------------|----------|----------------|---------------------------|---------|
| **critical-gov** | Government APIs (must succeed for compliance) | 10 | 10 failures | 60s |
| **standard-gov** | Government APIs (best effort) | 5 | 5 failures | 45s |
| **payments** | Payment providers (money involved) | 3 | 3 failures | 30s |
| **business** | Business services (accounting, HR) | 5 | 5 failures | 30s |
| **notifications** | Non-critical notifications | 2 | 3 failures | 15s |

## 6.2 API to Policy Mapping

| External API | Policy Group | Notes |
|--------------|--------------|-------|
| SATU SEHAT | critical-gov | Required for compliance |
| P-Care/BPJS | critical-gov | Required for claims |
| Mobile JKN | standard-gov | Value-add feature |
| Xendit | payments | Financial transactions |
| Jurnal | business | Accounting sync |
| Xero | business | International accounting |
| Talenta | business | HR sync |
| Zoho Desk | notifications | Support tickets |
| Kyoo | notifications | Queue notifications |

## 6.3 Implementation

```go
// /pkg/apiclient/policies.go

var PolicyGroups = map[string]Config{
    "critical-gov": {
        Timeout: 60 * time.Second,
        Retry: RetryConfig{
            MaxAttempts:     10,
            InitialInterval: 2 * time.Second,
            MaxInterval:     60 * time.Second,
            Multiplier:      2.0,
        },
        CircuitBreaker: CircuitBreakerConfig{
            Enabled:          true,
            FailureThreshold: 10,
            SuccessThreshold: 3,
            Timeout:          60 * time.Second,
        },
    },
    "standard-gov": {
        Timeout: 45 * time.Second,
        Retry: RetryConfig{
            MaxAttempts:     5,
            InitialInterval: 1 * time.Second,
            MaxInterval:     30 * time.Second,
            Multiplier:      2.0,
        },
        CircuitBreaker: CircuitBreakerConfig{
            Enabled:          true,
            FailureThreshold: 5,
            SuccessThreshold: 2,
            Timeout:          30 * time.Second,
        },
    },
    "payments": {
        Timeout: 30 * time.Second,
        Retry: RetryConfig{
            MaxAttempts:     3,
            InitialInterval: 500 * time.Millisecond,
            MaxInterval:     5 * time.Second,
            Multiplier:      2.0,
        },
        CircuitBreaker: CircuitBreakerConfig{
            Enabled:          true,
            FailureThreshold: 3,
            SuccessThreshold: 2,
            Timeout:          30 * time.Second,
        },
    },
    // ... other policies
}

func ConfigFromPolicy(name, baseURL, policy string) Config {
    base, ok := PolicyGroups[policy]
    if !ok {
        base = DefaultConfig(name, baseURL)
    }
    base.Name = name
    base.BaseURL = baseURL
    return base
}
```

---

# Appendix: Team Discussion Points

## Key Decision: JSON Logs

**Proposal**: Use structured JSON logs across all services via `uber-go/zap`

**Benefits**:
- Easy to parse and search (CloudWatch Logs Insights, grep with jq)
- Consistent format across services
- Machine-readable for dashboards

**Example**:
```json
{"level":"info","ts":"2025-01-19T10:30:00Z","caller":"apiclient/client.go:95","msg":"API request completed","client":"satu-sehat","request_id":"abc-123","status":200,"duration_ms":245}
```

**Alternative**: Plain text logs (easier to read in terminal, harder to parse)

**Team to confirm**: JSON logs acceptable?

---

*Document Owner: Alex (CMO)*  
*Last Updated: January 2025*  
*Status: Ready for implementation*
