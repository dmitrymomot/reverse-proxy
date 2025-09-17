# Lightweight Multi-Tenant Proxy Server Requirements

## Project Overview

A modular reverse proxy system that can run as a standalone server OR be embedded as a package within applications. Features automatic certificate management, load balancing, circuit breaker protection, and tenant identification via headers. Supports pluggable storage adapters with bbolt as the default embedded database. No manual configuration files required - all configuration via API.

## Functional Requirements

### 1. Request Routing & Load Balancing

**FR-1.1** Route incoming HTTP/HTTPS requests based on exact domain match
**FR-1.2** Support multiple target instances per domain with round-robin load balancing
**FR-1.3** Automatically remove failed instances from rotation (circuit breaker)
**FR-1.4** Automatically restore instances when they recover
**FR-1.5** Add custom header X-Tenant-ID for custom domain requests
**FR-1.6** Add custom header X-Original-Host with original requested domain
**FR-1.7** Pass through all original headers except hop-by-hop headers
**FR-1.8** Preserve client IP via X-Forwarded-For and X-Real-IP headers
**FR-1.9** Support WebSocket upgrades with sticky sessions
**FR-1.10** Return 404 for any unregistered domain or subdomain
**FR-1.11** Return 502 when all instances for a domain are unavailable
**FR-1.12** Support HTTP to HTTPS automatic redirect
**FR-1.13** Handle CNAME chains where custom domains point to app subdomains
**FR-1.14** Support subdomains as custom domains with tenant identification

### 2. Certificate Management

**FR-2.1** Automatically provision Let's Encrypt certificate for each registered domain/subdomain
**FR-2.2** Use HTTP-01 challenge for domain validation
**FR-2.3** Store certificates on disk in configurable directory with domain-based structure
**FR-2.4** Automatically renew certificates 30 days before expiration
**FR-2.5** Serve existing valid certificates from disk storage
**FR-2.6** Each tenant subdomain receives individual certificate (no wildcards)
**FR-2.7** Queue certificate generation to respect Let's Encrypt rate limits
**FR-2.8** Provide certificate expiry information via API
**FR-2.9** Skip certificate generation if domain fails validation
**FR-2.10** Clean up orphaned certificates for deleted routes
**FR-2.11** Generate separate certificates for CNAME source and target domains

### 3. Health Checks & Circuit Breaker

**FR-3.1** Active health checks every 10 seconds per instance
**FR-3.2** Mark instance as unhealthy after 3 consecutive failed checks
**FR-3.3** Remove unhealthy instances from load balancer rotation
**FR-3.4** Mark instance as healthy after 2 consecutive successful checks
**FR-3.5** Support custom health check endpoint per application
**FR-3.6** Default health check via TCP connection if no endpoint specified
**FR-3.7** Circuit breaker opens after 5 consecutive request failures
**FR-3.8** Circuit breaker attempts recovery after 30 seconds
**FR-3.9** Exponential backoff for recovery attempts (30s, 60s, 120s)
**FR-3.10** Instance-specific circuit breakers (not domain-wide)

### 4. App Registration & Configuration API

**FR-4.1** Apps self-register on startup via POST /api/apps/register
**FR-4.2** Support multiple instances registering for same app
**FR-4.3** Heartbeat endpoint for instance health (POST /api/apps/heartbeat)
**FR-4.4** Automatic deregistration on missing heartbeats (configurable timeout)
**FR-4.5** Instance deregistration on shutdown (DELETE /api/apps/instances)
**FR-4.6** List all registered apps (GET /api/apps)
**FR-4.7** Get app details with instances (POST /api/apps/get)

### 4.1. Tenant Domain Management API

**FR-4.1.1** Add tenant subdomain (POST /api/tenants/domains)
**FR-4.1.2** Add custom domain with tenant mapping (POST /api/tenants/custom-domains)
**FR-4.1.3** Update domain configuration (PUT /api/tenants/domains)
**FR-4.1.4** Delete domain (DELETE /api/tenants/domains)
**FR-4.1.5** List all tenant domains (GET /api/tenants/domains)
**FR-4.1.6** Get domain details (POST /api/tenants/domains/get)
**FR-4.1.7** Validate domain ownership before certificate generation
**FR-4.1.8** Apply all changes immediately without proxy restart
**FR-4.1.9** Return appropriate HTTP status codes for all operations

### 5. Data Storage

**FR-5.1** Storage interface for pluggable backend implementations
**FR-5.2** Built-in bbolt adapter for embedded database
**FR-5.3** Support for custom storage adapters (PostgreSQL, Redis, etc.)
**FR-5.4** Store route metadata (domain, tenant_id, app_id, created_at)
**FR-5.5** Store instance metadata (instance_id, target_url, health_status)
**FR-5.6** Store certificate paths/data based on adapter capabilities
**FR-5.7** Atomic operations for data consistency
**FR-5.8** Support concurrent reads from multiple goroutines
**FR-5.9** In-memory cache layer above storage adapter
**FR-5.10** Automatic cache refresh on configuration changes
**FR-5.11** Storage adapter configuration via options pattern
**FR-5.12** Migration support between storage backends

### 6. Status & Webhooks

### 6. Status & Webhooks

**FR-6.1** Status endpoint showing proxy health (GET /api/status)
**FR-6.2** Status endpoint showing certificate status per domain
**FR-6.3** Status endpoint showing all instances and their health per domain
**FR-6.4** Configure webhook URL per application
**FR-6.5** Send webhook on successful certificate generation
**FR-6.6** Send webhook on certificate renewal
**FR-6.7** Send webhook on certificate failure
**FR-6.8** Send webhook when all instances for a domain are down
**FR-6.9** Include domain and status in webhook payload
**FR-6.10** Retry webhooks with exponential backoff on failure
**FR-6.11** Request ID generation and propagation for tracing

### 7. Architectural Modularity

**FR-7.1** Core proxy engine as embeddable Go package
**FR-7.2** Storage interface for pluggable backend implementations
**FR-7.3** Default bbolt storage adapter included
**FR-7.4** Certificate manager as separate module
**FR-7.5** Load balancer as separate module
**FR-7.6** Health checker as separate module
**FR-7.7** API server as optional module (for embedded use)
**FR-7.8** Standalone binary combining all modules
**FR-7.9** Programmatic API for embedded usage
**FR-7.10** Event hooks for custom middleware injection

## Non-Functional Requirements

### 1. Performance

**NFR-1.1** Route lookup latency < 1ms at 99th percentile (from cache)
**NFR-1.2** Support minimum 5,000 concurrent connections
**NFR-1.3** Handle 10,000 requests/second on 2-core CPU
**NFR-1.4** Memory usage < 300MB for 1,000 routes with 5,000 instances
**NFR-1.5** Database operations < 50ms at 95th percentile
**NFR-1.6** Certificate operations non-blocking to request routing
**NFR-1.7** Startup time < 2 seconds including cache warming
**NFR-1.8** Configuration changes apply within 100ms
**NFR-1.9** Health check operations non-blocking to request routing
**NFR-1.10** Load balancer decision < 0.1ms per request

### 2. Reliability

**NFR-2.1** 99.9% uptime for routing functionality
**NFR-2.2** Automatic recovery from panic in request handlers
**NFR-2.3** Retry failed certificate renewals with exponential backoff
**NFR-2.4** Continue serving routes if certificate generation fails
**NFR-2.5** Maximum 5 second timeout for upstream connections
**NFR-2.6** Circuit breaker per instance with automatic recovery
**NFR-2.7** Failover to healthy instances within 100ms
**NFR-2.8** Webhook delivery retry up to 3 times
**NFR-2.9** Database corruption detection on startup
**NFR-2.10** Memory limit enforcement to prevent OOM
**NFR-2.11** Graceful handling of instance registration/deregistration
**NFR-2.12** Sticky sessions for WebSocket connections

### 3. Security

**NFR-3.1** Support TLS 1.2 and 1.3 only
**NFR-3.2** Strong cipher suites following Mozilla modern configuration
**NFR-3.3** Certificate private keys with file permissions 0600
**NFR-3.4** API accessible only from localhost by default
**NFR-3.5** Rate limiting for Let's Encrypt API calls
**NFR-3.6** No sensitive data in logs (keys, tenant IDs in detail)
**NFR-3.7** Validate all input domains against DNS before certificate generation
**NFR-3.8** Directory traversal protection for certificate storage

### 4. Scalability

**NFR-4.1** Support up to 5,000 unique domains per proxy instance
**NFR-4.2** Support up to 100 app instances per domain
**NFR-4.3** Database size < 100MB for 5,000 routes with instances
**NFR-4.4** Certificate storage < 500MB for 5,000 domains
**NFR-4.5** O(1) route lookup using hash map (exact domain matching)
**NFR-4.6** O(1) round-robin instance selection
**NFR-4.7** Webhook queue up to 1,000 pending deliveries
**NFR-4.8** Concurrent certificate generation limit (5 parallel)
**NFR-4.9** Health check pool limited to 100 concurrent checks

### 5. Maintainability

**NFR-5.1** Standalone binary < 20MB, embeddable package < 5MB
**NFR-5.2** Zero runtime dependencies for core modules
**NFR-5.3** Configuration via environment variables or programmatic API
**NFR-5.4** Structured JSON logging with configurable output
**NFR-5.5** Clear error messages with actionable information
**NFR-5.6** Storage adapter migration support
**NFR-5.7** Health check endpoint for monitoring tools
**NFR-5.8** Semantic versioning for packages
**NFR-5.9** Each module independently testable
**NFR-5.10** Mock implementations for all interfaces

### 6. Operational

**NFR-6.1** Run as unprivileged user with CAP_NET_BIND_SERVICE
**NFR-6.2** Certificate directory configurable via environment
**NFR-6.3** Database location configurable via environment
**NFR-6.4** Automatic certificate renewal without manual intervention
**NFR-6.5** ACME challenge serving on port 80
**NFR-6.6** IPv4 and IPv6 dual-stack support
**NFR-6.7** Signal handling for graceful shutdown (SIGTERM, SIGINT)
**NFR-6.8** Systemd notify support for service management

## Use Cases

### Multi-Tenant SaaS with Self-Registering Applications

**Scenario**: Multiple SaaS applications deploy independently, register themselves, and support multiple instances with automatic failover

1. **Application self-registration on startup**:

    ```bash
    # App instance 1 starts and registers itself
    POST /api/register
    {
      "domain": "app1.com",
      "instance_id": "app1-prod-1",
      "target": "http://10.0.0.5:3001",
      "app_id": "app1",
      "health_check": "/health"
    }

    # App instance 2 starts and registers for same domain
    POST /api/register
    {
      "domain": "app1.com",
      "instance_id": "app1-prod-2",
      "target": "http://10.0.0.6:3001",
      "app_id": "app1",
      "health_check": "/health"
    }
    # Result: Load balancing between both instances
    ```

2. **Tenant subdomain registration by app**:

    ```bash
    # App registers new tenant subdomain
    POST /api/register
    {
      "domain": "tenant1.app1.com",
      "instance_id": "app1-prod-1",
      "target": "http://10.0.0.5:3001",
      "app_id": "app1"
    }
    # Each instance serving this domain must register separately
    ```

3. **Custom domain with tenant identification**:

    ```bash
    # App registers customer's custom domain
    POST /api/routes
    {
      "domain": "customer-brand.com",
      "app_id": "app1",
      "tenant_id": "tenant-123"
    }
    # All app1 instances automatically serve this domain
    # Adds X-Tenant-ID: tenant-123 header to requests
    ```

4. **Subdomain as custom domain (CNAME chain)**:

    ```bash
    # Register app subdomain as a custom domain target
    POST /api/routes
    {
      "domain": "tenant123.app1.com",
      "app_id": "app1",
      "tenant_id": "tenant-123"
    }

    # Customer creates CNAME: customer.com → tenant123.app1.com
    # Then register the customer's domain
    POST /api/routes
    {
      "domain": "customer.com",
      "app_id": "app1",
      "tenant_id": "tenant-123"
    }

    # Result: Both customer.com and tenant123.app1.com route to app1
    # Both get X-Tenant-ID: tenant-123 header
    # Each domain gets its own certificate
    ```

5. **Automatic failover workflow**:
    - Instance 1 stops responding to health checks
    - Proxy marks instance 1 as unhealthy after 3 failures
    - Traffic automatically routes only to instance 2
    - Instance 1 recovers and passes 2 health checks
    - Traffic resumes to both instances

6. **Zero-downtime deployment**:
    - Deploy new instance 3 with updated code
    - Instance 3 registers itself
    - Traffic starts routing to all 3 instances
    - Gracefully shutdown instance 1 (sends DELETE /api/register)
    - Instance 1 removed from rotation
    - Repeat for instance 2
    - Result: Updated without dropping requests

## Configuration

### Environment Variables Only (No Config Files)

```bash
# Proxy startup configuration via environment
PROXY_HTTP_PORT=80
PROXY_HTTPS_PORT=443
PROXY_API_PORT=8080
PROXY_API_BIND=127.0.0.1  # Local only by default

# Storage paths
PROXY_DB_PATH=/var/lib/proxy/routes.db
PROXY_CERT_PATH=/var/lib/proxy/certs

# Let's Encrypt settings
ACME_EMAIL=admin@example.com
ACME_STAGING=false  # Use production

# Health check settings
HEALTH_CHECK_INTERVAL=10s
HEALTH_CHECK_TIMEOUT=3s
HEARTBEAT_TIMEOUT=30s  # Mark instance dead after this

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# Start proxy with zero routes - apps will register themselves
./proxy-server
```

## Package Architecture

### Module Structure

```
github.com/yourorg/proxy/
├── cmd/proxy-server/        # Standalone server binary
├── pkg/proxy/               # Core proxy engine (embeddable)
│   ├── proxy.go            # Main proxy struct and methods
│   ├── router.go           # Request routing logic
│   └── middleware.go       # HTTP middleware chain
├── pkg/storage/            # Storage abstraction
│   ├── interface.go        # Storage interface definition
│   ├── bbolt/             # Default bbolt adapter
│   ├── memory/            # In-memory adapter (testing)
│   └── postgres/          # PostgreSQL adapter (optional)
├── pkg/certmanager/        # Certificate management
│   ├── manager.go         # Certificate manager
│   └── letsencrypt.go     # Let's Encrypt integration
├── pkg/loadbalancer/       # Load balancing
│   ├── balancer.go        # Load balancer interface
│   └── roundrobin.go      # Round-robin implementation
├── pkg/healthcheck/        # Health checking
│   ├── checker.go         # Health check manager
│   └── circuit.go         # Circuit breaker
└── pkg/api/               # REST API server
    ├── server.go          # API server
    ├── apps.go           # App registration endpoints
    └── tenants.go        # Tenant domain endpoints
```

### Standalone Usage

```bash
# Install standalone proxy server
go install github.com/yourorg/proxy/cmd/proxy-server

# Run with default settings
proxy-server

# Or with custom storage
proxy-server --storage=postgres --storage-dsn="postgres://..."
```

### Embedded Usage

```go
package main

import (
    "github.com/yourorg/proxy/pkg/proxy"
    "github.com/yourorg/proxy/pkg/storage/bbolt"
    "github.com/yourorg/proxy/pkg/certmanager"
)

func main() {
    // Create storage adapter
    store, err := bbolt.New("/var/lib/myapp/proxy.db")
    if err != nil {
        log.Fatal(err)
    }

    // Create certificate manager
    certMgr := certmanager.New(certmanager.Options{
        Email: "admin@example.com",
        Storage: "/var/lib/myapp/certs",
        Staging: false,
    })

    // Create proxy instance
    p := proxy.New(proxy.Options{
        Storage: store,
        CertManager: certMgr,
        HealthCheck: true,
        LoadBalance: true,
    })

    // Programmatic route registration
    p.RegisterApp("app1", "http://localhost:3001")
    p.RegisterDomain("customer.com", "app1", "tenant-123")

    // Start proxy
    go p.ListenAndServeTLS(":443")
    p.ListenAndServe(":80")
}
```

### Custom Storage Adapter

```go
package mystorage

import "github.com/yourorg/proxy/pkg/storage"

type MyCustomStorage struct {
    // your storage implementation
}

func (s *MyCustomStorage) GetRoute(domain string) (*storage.Route, error) {
    // Implementation
}

func (s *MyCustomStorage) SaveRoute(route *storage.Route) error {
    // Implementation
}

func (s *MyCustomStorage) DeleteRoute(domain string) error {
    // Implementation
}

// ... implement remaining interface methods
```

### Storage Interface

```go
type Storage interface {
    // Route management
    GetRoute(domain string) (*Route, error)
    SaveRoute(route *Route) error
    DeleteRoute(domain string) error
    ListRoutes() ([]*Route, error)

    // Instance management
    RegisterInstance(appID string, instance *Instance) error
    DeregisterInstance(instanceID string) error
    GetInstances(appID string) ([]*Instance, error)
    UpdateInstanceHealth(instanceID string, healthy bool) error

    // Certificate references
    SaveCertRef(domain string, path string) error
    GetCertRef(domain string) (string, error)

    // Transactional operations
    Transaction(fn func(tx Transaction) error) error
}
```

## API Examples

### App Registration (Called by App on Startup)

```bash
POST /api/apps/register
{
  "app_id": "app1",
  "app_name": "Main Application",
  "instance": {
    "instance_id": "app1-prod-node-1",
    "target": "http://10.0.0.5:3001",
    "health_check": "/health"
  }
}
# Returns: 201 Created
# App registered with first instance
```

### Add Additional Instance

```bash
POST /api/apps/instances
{
  "app_id": "app1",
  "instance_id": "app1-prod-node-2",
  "target": "http://10.0.0.6:3001",
  "health_check": "/health"
}
# Returns: 201 Created
# Load balancing now active between instances
```

### Instance Heartbeat

```bash
POST /api/apps/heartbeat
{
  "instance_id": "app1-prod-node-1"
}
# Returns: 204 No Content
# Keeps instance marked as alive
```

### Deregister Instance (Graceful Shutdown)

```bash
DELETE /api/apps/instances
{
  "instance_id": "app1-prod-node-1"
}
# Returns: 204 No Content
# Immediately removes from load balancer
```

### Register Tenant Subdomain

```bash
POST /api/tenants/domains
{
  "domain": "tenant1.app1.com",
  "app_id": "app1",
  "type": "subdomain"
}
# Returns: 201 Created
# Certificate generation starts automatically
```

### Add Custom Domain with Tenant Mapping

```bash
POST /api/tenants/custom-domains
{
  "domain": "customer-brand.com",
  "app_id": "app1",
  "tenant_id": "tenant-123",
  "cname_target": "tenant-123.app1.com"  # Optional CNAME hint
}
# Returns: 201 Created
# X-Tenant-ID: tenant-123 will be added to requests
```

### Update Domain Configuration

```bash
PUT /api/tenants/domains
{
  "domain": "customer-brand.com",
  "app_id": "app1",
  "tenant_id": "tenant-456"  # Update tenant mapping
}
# Returns: 200 OK
```

### Delete Domain

```bash
DELETE /api/tenants/domains
{
  "domain": "old-customer.com"
}
# Returns: 204 No Content
# Domain and certificate removed
```

### Get Domain Details

```bash
POST /api/tenants/domains/get
{
  "domain": "customer-brand.com"
}
# Returns: 200 OK
{
  "domain": "customer-brand.com",
  "app_id": "app1",
  "tenant_id": "tenant-123",
  "certificate_status": "active",
  "certificate_expiry": "2025-03-15T00:00:00Z",
  "type": "custom",
  "created_at": "2025-01-01T00:00:00Z"
}
```

### Use Subdomain as CNAME Target

```bash
# Step 1: Register subdomain with tenant mapping
POST /api/tenants/domains
{
  "domain": "tenant-123.app1.com",
  "app_id": "app1",
  "tenant_id": "tenant-123",
  "type": "cname_target"
}

# Step 2: Customer creates CNAME record
# customer.com CNAME → tenant-123.app1.com

# Step 3: Register customer's domain
POST /api/tenants/custom-domains
{
  "domain": "customer.com",
  "app_id": "app1",
  "tenant_id": "tenant-123",
  "cname_target": "tenant-123.app1.com"
}
```

### Check App Status

```bash
POST /api/apps/get
{
  "app_id": "app1"
}
# Returns: 200 OK
{
  "app_id": "app1",
  "app_name": "Main Application",
  "instances": [
    {
      "instance_id": "app1-prod-node-1",
      "target": "http://10.0.0.5:3001",
      "health": "healthy",
      "circuit_breaker": "closed",
      "last_heartbeat": "2025-01-15T12:00:00Z"
    },
    {
      "instance_id": "app1-prod-node-2",
      "target": "http://10.0.0.6:3001",
      "health": "healthy",
      "circuit_breaker": "closed",
      "last_heartbeat": "2025-01-15T12:00:05Z"
    }
  ],
  "domains": ["app1.com", "tenant1.app1.com", "customer-brand.com"],
  "created_at": "2025-01-01T00:00:00Z"
}
```

## API Endpoints

### App Registration & Management

- `POST /api/apps/register` - Register app with initial instance
- `POST /api/apps/instances` - Add new instance to app
- `DELETE /api/apps/instances` - Deregister instance (instance_id in payload)
- `POST /api/apps/heartbeat` - Keep instance alive
- `GET /api/apps` - List all registered apps
- `POST /api/apps/get` - Get app details with instances (app_id in payload)
- `POST /api/apps/instances/list` - List app instances (app_id in payload)

### Tenant Domain Management

- `POST /api/tenants/domains` - Add tenant subdomain
- `POST /api/tenants/custom-domains` - Add custom domain with tenant mapping
- `PUT /api/tenants/domains` - Update domain configuration (domain in payload)
- `DELETE /api/tenants/domains` - Delete domain (domain in payload)
- `GET /api/tenants/domains` - List all domains
- `POST /api/tenants/domains/get` - Get domain details (domain in payload)
- `POST /api/tenants/domains/verify` - Verify domain ownership (domain in payload)

### Status & Monitoring

- `GET /api/status` - Proxy health and statistics
- `GET /api/status/certificates` - Certificate status for all domains
- `GET /api/status/instances` - All instances and their health
- `POST /api/status/app` - App-specific health status (app_id in payload)

### Webhook Management

- `POST /api/webhooks` - Register webhook for app
- `PUT /api/webhooks` - Update webhook URL (app_id in payload)
- `DELETE /api/webhooks` - Remove webhook (app_id in payload)
- `GET /api/webhooks` - List all webhooks

## Data Models

### App Registration

```json
{
    "app_id": "app1",
    "app_name": "Main Application",
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-01-15T12:00:00Z"
}
```

### App Instance

```json
{
    "instance_id": "app1-prod-node-1",
    "app_id": "app1",
    "target": "http://10.0.0.5:3001",
    "health_check": "/health",
    "health": "healthy",
    "circuit_breaker": "closed",
    "last_heartbeat": "2025-01-15T12:00:00Z",
    "registered_at": "2025-01-01T00:00:00Z"
}
```

### Tenant Domain

```json
{
    "domain": "tenant1.app1.com",
    "app_id": "app1",
    "tenant_id": null, // Set for custom domains
    "type": "subdomain", // subdomain, custom, cname_target
    "cname_target": null, // For custom domains using CNAME
    "certificate_status": "active",
    "certificate_expiry": "2025-03-15T00:00:00Z",
    "created_at": "2025-01-01T00:00:00Z"
}
```

### Custom Domain

```json
{
    "domain": "customer-brand.com",
    "app_id": "app1",
    "tenant_id": "tenant-123",
    "type": "custom",
    "cname_target": "tenant-123.app1.com", // Optional CNAME target
    "certificate_status": "pending",
    "certificate_expiry": null,
    "verified": false,
    "created_at": "2025-01-15T12:00:00Z"
}
```

### Webhook Payload

```json
{
    "event": "certificate_issued", // or certificate_renewed, certificate_failed, all_instances_down, instance_registered
    "app_id": "app1",
    "domain": "customer.com",
    "tenant_id": "tenant-123",
    "timestamp": "2025-01-15T12:00:00Z",
    "details": {
        "expires_at": "2025-04-15T00:00:00Z",
        "healthy_instances": 2,
        "total_instances": 3
    }
}
```

## Constraints & Assumptions

### Technical Constraints

- Pure Go implementation (no CGO for core modules)
- bbolt adapter requires CGO-free environment
- Single proxy instance (proxy itself not clustered)
- **Let's Encrypt Production rate limits:**
    - 50 certificates per registered domain per week
    - 5 duplicate certificates (same exact domains) per week
    - 300 new orders per account per 3 hours
    - 100 domains per single certificate (SAN limit)
    - 10 accounts per IP address per 3 hours
- **Let's Encrypt Staging (for testing):**
    - 30,000 certificates per registered domain per week
    - 30,000 duplicate certificates per week
    - Use staging during development: acme-staging-v02.api.letsencrypt.org
    - Staging certificates not trusted by browsers (testing only)
- Each subdomain requires individual certificate (no wildcard certificates)
- Certificate generation is sequential to avoid rate limits
- Round-robin load balancing only (no weighted or least-connections)
- Storage adapters must implement full interface or return errors

### Operational Assumptions

- DNS already configured before adding route
- **Standalone mode**: Apps self-register via API on startup
- **Embedded mode**: Host app manages registration programmatically
- Applications send heartbeats or handle health check endpoints
- Applications deregister on graceful shutdown
- Each app instance has unique instance_id
- Proxy runs on dedicated ports 80/443 (configurable in embedded mode)
- Certificate storage has sufficient disk space
- Applications handle tenant routing internally after receiving headers
- Network allows proxy to reach app instances (no NAT issues)
- Apps deployed with service discovery aware of proxy API
- **CNAME records pointing to subdomains are configured by customers**
- **Proxy handles both direct domains and CNAME targets**
- **Storage adapter choice made at initialization time**
- **Embedded apps responsible for storage adapter lifecycle**

## Out of Scope

### General Limitations

- Wildcard domain routing (each subdomain must be registered individually)
- Weighted or least-connections load balancing (round-robin only)
- Session affinity beyond WebSocket connections
- Request/response body modification
- Authentication for proxied requests (handled by apps)
- SSL termination for upstream (HTTP only to apps)
- Multiple proxy instances (single proxy only)
- Service mesh features (retry, timeout per route)
- Request logging beyond basic access logs
- Certificate management for non-Let's Encrypt CAs
- DNS challenge for certificate validation
- Backup/restore functionality (handled externally)
- Wildcard certificates (\*.domain.com)
- Automatic subdomain discovery or registration
- Configuration files (all config via API/environment/code)
- Manual route configuration

### Embedded Package Exclusions

- API server (optional, must be explicitly included)
- Signal handling (host app responsibility)
- Process management (host app responsibility)
- Log output management (host app configures)
- Storage adapter implementations beyond bbolt
- Metrics collection (host app can hook into events)

## Testing Requirements

### Unit Testing

- Each module independently testable with mocks
- Storage interface with in-memory test adapter
- Mock Let's Encrypt server for certificate tests
- Simulated health check failures for circuit breaker
- Load balancer testing with mock instances

### Integration Testing

- Standalone binary with all modules
- bbolt storage adapter persistence
- Real Let's Encrypt staging environment
- Multi-instance registration and failover
- CNAME resolution and routing

### Embedded Testing

- Example embedded applications
- Custom storage adapter examples
- Programmatic API usage examples
- Event hook integration tests
- Memory leak detection under load
