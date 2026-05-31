# Warpgate

A self-hosted HTTP edge runtime written in [Rux](https://rux-lang.org), targeting Windows with zero stdlib dependencies (Win32/Winsock2 FFI only).

Use Warpgate as a **library** — bring your own request handler, configure the rest via `warpgate.toml`.

## Features

- **Transports** — IOCP (default), HTTP.sys (kernel-mode), Thread-pool
- **HTTP/2** — h2 (ALPN over TLS) + h2c (plain), HPACK static table, all frame types
- **Inbound TLS** — SChannel server-side via `.pfx` certificate, ALPN negotiation
- **Reverse proxy** — multi-upstream with round-robin / least-connections LB, health checks, WebSocket relay, upstream TLS
- **Static files** — ETag/304, gzip, chunked transfer
- **Rate limiting** — token-bucket per client IP (512-slot LRU hash table)
- **JWT auth** — HS256/HMAC-SHA256, `exp` claim validation, per-prefix enforcement
- **SQLite** — embedded query/exec endpoints via `sqlite3.dll`
- **Gzip** — response compression via `zlib1.dll`
- **Access logging** — stdout + append-mode file, microsecond latency
- **Hot-reload config** — polls `warpgate.toml` for changes, no restart needed
- **Graceful shutdown** — drains active connections on Ctrl+C

## Quick start

```rux
// Src/Main.rux in your app
func MyHandler(sock: int, req: *Request, ctx: *ServerCtx, tls: *opaque) -> bool {
    if req.method == Method.GET && PathEq(req, c8"/hello".data, 6) {
        SendText(sock, tls, c8"200 OK".data, 6,
                 c8"Hello from my app!\n".data, 19, req.keep_alive);
        req.status_code = 200;
        return req.keep_alive;
    }
    return false; // Warpgate sends 404
}

func Main() -> int {
    return WarpgateRun(MyHandler as *opaque);
}
```

Your handler is called for every request that does not match a built-in route. Return `true` to keep the connection alive, `false` to close it.

## Built-in routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Liveness probe |
| `GET` | `/health` | JSON — queue depth, worker count, upstream count |
| `GET` | `/metrics` | Prometheus-format counters |
| `POST` | `/echo` | Echoes request body |
| `GET` | `/db/query?sql=...` | SQLite SELECT (requires `[Database] Enabled=true`) |
| `POST` | `/db/exec` | SQLite write (requires `[Database] Enabled=true`) |

Proxy and static paths are also handled before your handler is called.

## Configuration (`warpgate.toml`)

```toml
[Server]
Port      = 8080
Workers   = 8
Transport = IOCP        # IOCP | HTTPSys | ThreadPool
Gzip            = true
GzipMinBytes    = 512
HotReload         = false
HotReloadInterval = 30  # seconds

[Proxy]
Enabled     = false
Upstreams   = ""        # "host:port, host:port:tls"
Prefix      = "/api"
Strip       = true
LbMode      = RoundRobin  # RoundRobin | LeastConn
HealthCheck = 10          # seconds

[Static]
Enabled = false
Dir     = "./public"
Prefix  = "/"

[Database]
Enabled = false
Path    = "warpgate.db"

[RateLimit]
Enabled = false
Rps     = 100
Burst   = 200

[Auth]
JwtEnabled = false
JwtSecret  = "change-me-to-a-strong-secret"
JwtPrefix  = "/api"

[Tls]
Enabled  = false
CertFile = "server.pfx"
CertPass = ""

[Log]
File = ""  # e.g. "access.log"
```

## Public API

| Symbol | Description |
|--------|-------------|
| `WarpgateRun(handler: *opaque) -> int` | Start server, read `warpgate.toml` |
| `WarpgateDefaultConfig() -> Config` | Get default config values |
| `WarpgateParseConfig(path: *char8) -> Config` | Parse a toml config file |
| `SendText(...)` | Send a plain-text HTTP response |
| `SendJson(...)` | Send a JSON HTTP response |
| `ConnSend(...)` | Send raw bytes on the connection |
| `BuildResponse(...)` | Build a raw HTTP/1.1 response into a buffer |
| `GetHeader(req, name, len)` | Look up a request header value |

Handler signature:
```rux
func MyHandler(sock: int, req: *Request, ctx: *ServerCtx, tls: *opaque) -> bool
```

## Source layout

| File | Role |
|------|------|
| `Src/Warpgate.rux` | Public API surface |
| `Src/Main.rux` | `WarpgateRun`, hot-reload thread, graceful shutdown |
| `Src/Router.rux` | `Dispatch` — rate limit, JWT, built-in routes, user handler |
| `Src/Http.rux` | HTTP/1.1 parser, response builder |
| `Src/Http2.rux` | HTTP/2 framing, HPACK, stream handling |
| `Src/Tls.rux` | SChannel inbound/outbound TLS, ALPN |
| `Src/Proxy.rux` | Reverse proxy, WebSocket relay |
| `Src/Static.rux` | Static file serving, ETag, gzip |
| `Src/Iocp.rux` | IOCP transport |
| `Src/Pool.rux` | `ServerCtx`, thread-pool transport |
| `Src/Httpsys.rux` | HTTP.sys transport |
| `Src/Config.rux` | `warpgate.toml` parser |
| `Src/RateLimit.rux` | Token-bucket rate limiter |
| `Src/Jwt.rux` | HS256 JWT validation |
| `Src/Db.rux` | SQLite3 bindings |
| `Src/Ffi.rux` | All Win32/Winsock2 FFI declarations |

## Known limitations

- HTTP/2 Huffman decoding not implemented — Chrome/Firefox will get `COMPRESSION_ERROR`; `curl --http2` works fine
- IOCP transport handles TLS connections synchronously (ties up one worker per TLS connection)
- HTTP/2 stream multiplexing is sequential (streams are queued, not concurrent)
- No HTTP/2 server push
- Log file path requires a restart to change (not hot-reloaded)

## Requirements

- Windows (Win32/Winsock2)
- Rux compiler
- `zlib1.dll` in PATH for gzip support
- `sqlite3.dll` in PATH for database support
