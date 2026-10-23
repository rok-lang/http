# Contributing

Thanks for your interest in improving this rok library (a *stone*).

## Development

```sh
rok test          # tests, in tests/
rok bench         # benchmarks, in benches/
rok fmt --check   # formatting
```

Tests live in `tests/` and benchmarks in `benches/`; both reach the package
through `pkg::`. Format before sending a change (`rok fmt`).

## Changes that affect behavior

Update the README (the API table and examples) alongside any change in
behavior, and add a test for new behavior.

## Submitting

Open a pull request against `main` with `rok test` and `rok fmt --check`
passing. By contributing you agree that your work is licensed under both MIT and
Apache-2.0, the project's dual license.
