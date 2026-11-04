# Changelog

All notable changes to this stone. Versions follow semver, independent of the Rok toolchain.

## 0.2.0 (unreleased)

Needs Rok 0.2.

### Changed

- A TLS setup failure in `Client` surfaces as an `ErrorKind::Tls` `HttpError` on the first
  `https` request instead of at construction.
- `Server::run` fails with `ErrorKind::Unsupported` when called a second time;
  `Server::local_addr` returns the bound address.
- The HTTP/1.1 parsing, framing and response items of `h1` exist only with the `http1`
  feature. Every feature set builds without warnings.
- `Handler` is `fn(Request) -> AsyncFuture<Response> + Send + Sync`: the server calls one
  handler from many tasks, and Rok 0.2's `fn(..) + Send` alone is not `Sync`.
- `HeaderMap::remove` is `#[discardable]`.
- The default `user_agent` is `rok-http/0.2`.

## 0.1.0

First release.
