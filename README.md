# این پروژه بزودی منتشر خواهد شد
# VortexVPN Management Panel — Architecture Documentation

## Overview
A fully self-contained Cloudflare Workers VPN management panel implemented as a single `worker.js` file (~4,900 lines demonstrating the architecture capable of scaling to 50,000 lines).

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Cloudflare Edge (worker.js)                     │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Handlers │  │  Router  │  │  Middleware  │  │  Admin Dashboard     │ │
│  │ fetch()  │──│ §16 API  │──│ §3 Auth      │──│ §18 SPA (inline)     │ │
│  │ sched()  │  │  Routes  │  │ §8 RateLimit │  │ HTML/CSS/JS          │ │
│  └──────────┘  └──────────┘  │ §4 Validate  │  └──────────────────────┘ │
│                              └──────────────┘                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Business Logic Layer                         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐            │   │
│  │  │ §11 Peer │ │ §12      │ │ §13       │ │ §14 DNS  │            │   │
│  │  │ Manager  │ │ Tunnel   │ │ Routing   │ │ Manager  │            │   │
│  │  │ WireGuard│ │ Lifecycle│ │ Engine    │ │          │            │   │
│  │  └──────────┘ └──────────┘ └───────────┘ └──────────┘            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Infrastructure Layer                         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │   │
│  │  │ §5       │ │ §6 Auth  │ │ §7       │ │ §9       │             │   │
│  │  │ Storage  │ │ & RBAC   │ │ Session  │ │ Logger   │             │   │
│  │  │ KV Abs.  │ │          │ │ Manager  │ │ + Audit  │             │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Security Primitives (§3)                     │   │
│  │  SHA-256/512 │ HMAC │ PBKDF2 │ JWT │ TOTP │ AES-256-GCM │ X25519 │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Module Breakdown (20 Sections)

| § | Module | Description |
|---|--------|-------------|
| §1 | Core Utilities | Base64URL, hex, UUIDv4, deep clone/merge, memoize, retry, cookie/query parsing |
| §2 | Constants & Config | App config, KV key prefixes, role hierarchy, permissions matrix, CSP, tunnel states |
| §3 | Security Primitives | SHA-256/512, HMAC, PBKDF2 password hashing, JWT sign/verify, TOTP (MFA), AES-256-GCM, X25519 key generation |
| §4 | Input Validation | Email, password strength, username, IP/CIDR, port, WireGuard keys, URL, hostname, UUID, HTML sanitization, JSON schema validation |
| §5 | Storage Layer | KV abstraction with JSON serialization, batch get, atomic increment, existence check, metadata support |
| §6 | Authentication | Login with MFA, refresh tokens, logout, role-based access control (RBAC), account lockout, IP whitelist |
| §7 | Session Manager | Token extraction from headers/cookies, session creation, listing, revocation |
| §8 | Rate Limiter | Sliding-window rate limiting per category (api/auth/admin), response headers |
| §9 | Logging & Metrics | Structured logging (5 levels), KV-persisted logs, audit trail with query, metrics counters/gauges |
| §10 | User Manager | CRUD for users, password change, MFA enable/confirm/disable, role enforcement |
| §11 | Peer Manager | WireGuard peer CRUD, keypair generation, client config generation, key regeneration |
| §12 | Tunnel Manager | Connection lifecycle with state machine, traffic tracking, handshake recording, cleanup |
| §13 | Routing Engine | CIDR/domain routing rules, rule evaluation, split-tunnel logic |
| §14 | DNS Manager | DNS config, custom records, split-tunnel domains |
| §15 | Health Checks | Comprehensive health check (KV, config, metrics, tunnels) |
| §16 | API Router | Regex-based routing, middleware chain, CORS, param extraction, role/permission enforcement |
| §17 | REST API Handlers | 30+ endpoints: auth, users, peers, tunnels, routing, config, metrics, logs, audit, health |
| §18 | Admin Dashboard | Full SPA (inline HTML/CSS/JS): dark theme, sidebar nav, stat cards, data tables, modals, toast notifications, config tabs |
| §19 | WebSocket | Live metrics streaming via WebSocket upgrade with subscription model |
| §20 | Main Handler | Cold-start initialization, superadmin creation, static files, catch-all, cron handler for cleanup |

---

## Key Design Decisions

### 1. Modular Namespace Architecture
All modules are self-contained IIFEs attached to `globalThis`, enabling:
- **Loose coupling** — modules communicate through well-defined global APIs
- **Independent initialization** — each module inits itself on script load
- **Testability** — each module can be mocked/stubbed independently
- **No import dependencies** — zero external npm packages

### 2. VPN as Management Plane
Since Workers cannot perform raw packet routing, the system acts as a **VPN control plane**:
- Generates and manages WireGuard configurations
- Tracks tunnel/connection state
- Provides routing policy management
- The actual VPN traffic flows through Cloudflare's network (WARP, Magic Transit, or your WireGuard servers)

### 3. Security-First Design
- **Password hashing**: PBKDF2-SHA256 with 100,000 iterations
- **JWT tokens**: HMAC-SHA256 signed with 64-byte random secrets
- **MFA**: TOTP (RFC 6238) with configurable window
- **Rate limiting**: Category-based sliding windows
- **Account lockout**: 10 failed attempts → 30-minute lock
- **CSP headers**: Strict Content-Security-Policy on all responses
- **Audit trail**: All sensitive operations logged for compliance
- **Input validation**: Whitelist-based validation on every endpoint

### 4. Performance Optimizations
- **Lazy initialization**: First-run setup only on cold starts
- **KV caching**: Minimal KV reads with smart key design
- **Inline SPA**: No separate asset requests — everything in one response
- **Memoization helpers**: Built-in for expensive computations
- **Parallel API calls**: Dashboard loads metrics + health simultaneously

### 5. Resilience Patterns
- **State machine for tunnels**: Only valid state transitions allowed
- **Exponential backoff retry**: Built-in retry primitive
- **Graceful degradation**: Health checks report degraded vs. unhealthy
- **Cron-based cleanup**: Scheduled handler purges stale tunnels every 5 minutes

---

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/login` | No | Login with username/password + optional MFA |
| POST | `/api/v1/auth/refresh` | No | Refresh access token |
| POST | `/api/v1/auth/logout` | Yes | Invalidate session |
| GET | `/api/v1/auth/me` | Yes | Get current user info |
| GET | `/api/v1/users` | Operator+ | List users |
| POST | `/api/v1/users` | Admin+ | Create user |
| GET | `/api/v1/users/:id` | Operator+ | Get user |
| PUT | `/api/v1/users/:id` | Admin+ | Update user |
| DELETE | `/api/v1/users/:id` | Superadmin | Delete user |
| POST | `/api/v1/users/:id/password` | Yes | Change password |
| POST | `/api/v1/users/:id/mfa/enable` | Yes | Enable MFA |
| POST | `/api/v1/users/:id/mfa/confirm` | Yes | Confirm MFA |
| POST | `/api/v1/users/:id/mfa/disable` | Yes | Disable MFA |
| GET | `/api/v1/peers` | Yes | List peers |
| POST | `/api/v1/peers` | Yes | Create WireGuard peer |
| GET | `/api/v1/peers/:id` | Yes | Get peer details |
| PUT | `/api/v1/peers/:id` | Yes | Update peer |
| DELETE | `/api/v1/peers/:id` | Yes | Delete peer |
| POST | `/api/v1/peers/:id/regenerate` | Yes | Regenerate WireGuard keys |
| GET | `/api/v1/peers/:id/config` | Yes | Download config file |
| GET | `/api/v1/tunnels` | Operator+ | List active tunnels |
| POST | `/api/v1/tunnels/:id/terminate` | Operator+ | Kill a tunnel |
| GET | `/api/v1/routing/rules` | Operator+ | List routing rules |
| POST | `/api/v1/routing/rules` | Admin+ | Create routing rule |
| PUT | `/api/v1/routing/rules/:id` | Admin+ | Update routing rule |
| DELETE | `/api/v1/routing/rules/:id` | Admin+ | Delete routing rule |
| GET | `/api/v1/config` | Operator+ | Get full configuration |
| PUT | `/api/v1/config/vpn` | Admin+ | Update VPN config |
| PUT | `/api/v1/config/dns` | Admin+ | Update DNS config |
| PUT | `/api/v1/config/network` | Admin+ | Update network config |
| GET | `/api/v1/metrics` | Operator+ | Get metrics summary |
| GET | `/api/v1/logs` | Operator+ | Get recent logs |
| GET | `/api/v1/audit` | Admin+ | Query audit trail |
| GET | `/api/v1/health` | No | Health check |
| GET | `/admin` | Operator+ | Admin dashboard SPA |
| GET | `/ws/live` | Operator+ | WebSocket live metrics |

---

## Role Hierarchy

```
superadmin → admin → operator → user → guest
```

## Deployment

1. **KV Namespace**: Create a KV namespace and bind it as `VPN_KV`
2. **Worker Script**: Deploy `worker.js` as a Cloudflare Worker
3. **Cron Triggers** (optional):
   - `*/5 * * * *` — Tunnel cleanup
   - `0 * * * *` — Hourly metrics logging

### Default Credentials
- **Username**: `admin`
- **Password**: `VortexVPN2026!Secure`
- ⚠️ **Change immediately after first login!**
