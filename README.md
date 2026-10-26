# http

HTTP/1.1 and HTTP/2 for Rok: a message model, a pooling client and an async
server, over plaintext or TLS 1.3. Built on `rok::std::net` with no other
dependencies. Parsing is strict where leniency would allow request smuggling or
response splitting.

## Add it

```
rok add http
```

Features `http1`, `http2` and `tls` are on by default; turn off what you do not
need with `default_features: false`.

## Usage

```rok
// One-off request with a default client.
let r = try await http::get("http://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

The server is router-free: one `async fn(Request) -> Response` handler, served
with `http::serve` / `http::serve_tls`. See the API docs for the client, server
and TLS details.

## Docs

https://docs.rok-lang.org

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option.
