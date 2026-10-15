# http

HTTP/1.1 ([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) / [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112)) and HTTP/2 ([RFC 9113](https://www.rfc-editor.org/rfc/rfc9113), HPACK [RFC 7541](https://www.rfc-editor.org/rfc/rfc7541)) for rok: the message model, a pooling client and an async server, over plaintext or TLS 1.3, built on `rok::std::net` (and `rok::std::net::tls`) with no other dependencies.

- **Model**: `Method`, `StatusCode`, `HeaderMap` (case-insensitive multimap in insertion order), `Uri` (RFC 3986 parsing, normalization, reference resolution, percent-encoding), `application/x-www-form-urlencoded`, cookies (RFC 6265 `Cookie` / `Set-Cookie`), `Request` / `Response` with streaming `Body`s.
- **HTTP/1.1**: keep-alive, pipelining, chunked bodies (extensions and trailers accepted), `Expect: 100-continue`, size limits and timeouts, graceful shutdown. Parsing is strict where leniency enables request smuggling or response splitting (see below).
- **HTTP/2**: client and server with prior knowledge over plaintext (h2c), HPACK with Huffman coding, stream and connection flow control, RFC 9218 priorities (`priority` header and `PRIORITY_UPDATE`) for response scheduling, SETTINGS / PING / GOAWAY / RST_STREAM handling, graceful shutdown with GOAWAY. Server push is not implemented (the client disables it).
- **TLS**: `https` URLs (SNI, certificates checked against the OS trust store or your roots) and a TLS server, both on std's TLS 1.3. ALPN offers `h2` and `http/1.1`; HTTP/2 is used when the server picks it. Servers need an Ed25519 certificate for now (std's limit). See [TLS internals](#tls-internals) for the costs.

## Usage

```rokon
Package {
    deps: { "http": "0.1" },
}
```

### Features

`http1`, `http2` and `tls`, all on by default. Turn off what you do not need (`deps: { "http": Dep::version("0.1", default_features: false, features: ["http1"]) }`): a protocol that is off fails with `ErrorKind::Unsupported` at run time, ALPN offers only the protocols that are on, and without `tls` there are no `Client::with_tls`, `Server::bind_tls` or `serve_tls`.

### Client

```rok
// One-off request with a default client.
let r = try await http::get("http://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());

// A client keeps connections; settings are a struct with defaults.
let c = http::Client::new(http::ClientConfig { timeout: Duration::sec(5), max_redirects: 3u32 });
let req = (try http::Request::post("http://localhost:8080/items"))
    .header("accept", "text/plain")
    .form([("name", "rok"), ("kind", "stone")]);
let resp = try await c.send(req);

// Streaming: read a body chunk by chunk.
while let chunk: List<u8> = try await resp.body.chunk() {
    consume(chunk);
}

// HTTP/2 by prior knowledge (h2c) for http:// URLs.
let h2 = http::Client::new(http::ClientConfig { version: http::Version::Http2 });

// https with your own trust anchors (default: the OS trust store).
let tc = tls::ClientConfig::new();
tc.root_certs = try tls::Certificate::from_pem(try fs::read_to_str("ca.pem"));
let sc = http::Client::with_tls(tc);
let r = try await sc.get("https://localhost:8443/");   // r.version: Http2 or Http11, as ALPN chose
```

### Server

The server is router-free: one handler, `async fn(Request) -> Response`, runs in its own task per request (per stream on HTTP/2). The same port speaks HTTP/1.1 and HTTP/2 with prior knowledge.

```rok
use http::{Request, Response, StatusCode, Body};

async fn handle(req: Request) -> Response {
    if req.uri.path == "/echo" {
        let data = try await req.body.bytes(1 << 20) catch _ { return Response::new(StatusCode::CONTENT_TOO_LARGE); };
        return Response::bytes(data);
    }
    if req.uri.path == "/count" {
        // A streamed response: chunked on HTTP/1.1, DATA frames on HTTP/2.
        let (w, body) = Body::channel();
        task::spawn(async {
            for i in 0..10 { try? await w.send_str(f"{i}\n"); }
            w.finish();
            0
        });
        return Response::new(StatusCode::OK).with_body(body);
    }
    Response::text(f"hello from {req.uri.path} over {req.version}\n")
}

try await http::serve("127.0.0.1:8080", handle);
```

Over TLS (`use rok::std::net::tls;`); ALPN selects HTTP/2 or HTTP/1.1 per connection:

```rok
let chain = try tls::Certificate::from_pem(try fs::read_to_str("cert.pem"));
let key = try tls::PrivateKey::from_pem(try fs::read_to_str("key.pem"));
try await http::serve_tls("0.0.0.0:8443", tls::ServerConfig::new(chain, key), handle);
```

With settings and graceful shutdown:

```rok
let s = try http::Server::bind("0.0.0.0:8080", handle, http::ServerConfig { max_body: 16u64 << 20u64 });
let t = task::spawn(async { try! await s.run(); 0 });
// ...
await s.shutdown(Duration::sec(10));   // stop accepting, finish in-flight requests
```

## API

| Item | Description |
|------|-------------|
| `get(url) -> Response` | GET with a default client (async, throws `HttpError`) |
| `Client::new(cfg?)`, `Client::with_tls(tls_cfg, cfg?)`, `send(req)`, `send_once(req)`, `get(url)` | Pooling client; `send` follows redirects (301/302/303 turn POST into GET, 307/308 resend a replayable body; credentials are dropped across origins) |
| `ClientConfig` | `timeout`, `connect_timeout`, `read_timeout`, `max_redirects`, `version`, `max_idle_per_host`, `idle_timeout`, `user_agent`, `limits`, `h2` |
| `serve(addr, handler)`, `serve_tls(addr, tls_cfg, handler)` | Bind and serve until the process ends |
| `Server::bind(addr, handler, cfg?)`, `Server::bind_tls(addr, handler, tls_cfg, cfg?)`, `run()`, `shutdown(grace)`, `local_addr()` | A server you control |
| `ServerConfig` | `limits`, `header_timeout`, `idle_timeout`, `io_timeout`, `max_body`, `http2`, `h2` |
| `h2::Config` | `max_concurrent_streams`, `initial_window_size`, `connection_window_size`, `max_frame_size`, `max_header_list_size`, `huffman`, `idle_timeout`, `write_timeout` |
| `Request` | `method`, `uri`, `version`, `headers`, `body`; `Request::get/head/post/put/delete(url)`, `.header(k, v)`, `.text(s)`, `.form(pairs)`, `.with_body(b)` |
| `Response` | `status`, `version`, `headers`, `body`; `Response::new(status)`, `text(s)`, `bytes(b)`, `.header(k, v)`, `.with_status(s)`, `.with_body(b)` |
| `Body` | `empty()`, `from_bytes`, `from_str`, `channel(cap, len)` → `(BodyWriter, Body)`; `chunk()`, `bytes(limit)`, `text(limit)`, `len()` |
| `HeaderMap` | `get`, `get_all`, `append`, `insert`, `remove`, `contains`, `entries`, `has_token`, `validate` |
| `Uri` | `parse`, `normalize`, `resolve`, `join`, `authority`, `path_and_query`, `query_pairs`, `port_or_default` |
| `percent_encode/decode`, `form_encode/decode` | Escaping helpers |
| `parse_cookie_header`, `cookie_header`, `SetCookie::parse/to_header` | Cookies |
| `HttpError { kind, message, code }` | `ErrorKind`: `Parse`, `Protocol`, `TooLarge`, `Timeout`, `Closed`, `Reset`, `Redirect`, `Unsupported`, `Tls`, `Io` |

## HTTP/1.1 strictness

Requests that could be read two ways are refused (400, connection closed) instead of guessed at:

- lines must end in CRLF; a bare CR or LF, a NUL or invalid UTF-8 in the head is an error;
- obsolete line folding and whitespace between a field name and its colon are errors;
- `Content-Length` must be one run of digits, given once;
- `Transfer-Encoding` must be exactly `chunked` (other codings: 501), never together with `Content-Length`, never in HTTP/1.0;
- chunk sizes are bounded hex, chunk data must be followed by CRLF, trailers are size-limited and dropped;
- HTTP/1.1 requests need exactly one `Host`; heads over 64 KiB or 100 fields get 431, targets over 8 KiB 414.

Outgoing header values are checked (`HeaderMap::validate`) so CR / LF cannot inject fields.

## TLS internals

`rok::std::net::tls` connections are blocking and must not be read and written from two threads at once, while HTTP/2 does both. So each TLS connection gets its own thread that owns the TLS state and runs one read or write at a time, talking to the network through a loopback socket pair whose other end async tasks copy to and from the real socket. A read is only started once a whole record has arrived, so reads do not wait on the network and writes do not wait behind them.

Costs and limits: one thread and one loopback socket pair per TLS connection, and an extra copy of every byte. If the only records pending carry no application data (a lone KeyUpdate, or session tickets the server sends before anything else), the read that consumes them waits for the peer's next record, and writes wait with it; servers that send tickets right after the handshake also send their first response or HTTP/2 SETTINGS, which ends the wait. A non-blocking or buffer-based API in std's TLS would remove the thread, the pair and this case.

## Development

```bash
rok test        # unit tests (RFC 3986 / 7541 vectors), smuggling cases, loopback h1 and h2, plain and TLS
rok fmt --check
```
