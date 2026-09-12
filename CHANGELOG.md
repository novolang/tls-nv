# Changelog

All notable changes to tls-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tlsrec` — the record layer, `[]` throughout: five header bytes, the
  two different length ceilings, the inner content type that is the
  last non-zero byte of the plaintext rather than the byte in the
  header, and the all-padding record that is an error rather than an
  empty message.
- `tlsmsg` — the handshake messages and extensions as values, `[]`
  throughout: the version read out of `supported_versions` and never
  out of the version field, the HelloRetryRequest that is a ServerHello
  with a constant in its random, the server-decides ALPN rule, and the
  CertificateVerify context whose label differs by role.
- `tlssuite` — the four cipher suites, `[]` throughout: the per-record
  nonce as the XOR it is rather than the concatenation TLS 1.2 used,
  the associated data that is exactly the record header, and
  `suite_is_available` answering honestly that three of the four have
  no primitive on this grid.
- `tlskey` — RFC 8446 § 7.1's key schedule, `[]` throughout:
  HKDF-Expand-Label with the structure the specification puts in the
  info field, the transcript rewrite a HelloRetryRequest forces, the
  constant-time Finished comparison, and the sequence number that
  resets with every key.
- `tlscert` — certificate chain validation, `[]` plus two
  effect-polymorphic functions: SAN matching with no common-name
  fallback, a wildcard that matches one leftmost label and no dot, a
  path BUILT from the chain rather than walked, and `TlsSigCheck[e]`
  for the signature check that has no primitive on this grid yet.
- `tlshs` — the handshake state machine, client and server, `[]`
  throughout: `TlsAction` as the load-bearing interface, with the
  randomness, the key exchange, the certificate verdict and the
  signature all arriving as arguments.
- `tlsconn` — the host half: `dial`, `send`, `recv` and `close` in the
  shape `std.tls` already has, `handshake_over` effect-polymorphic over
  a caller's byte pipe, one `[time]` function and one `[fs]` function.
- API tests in `tests/tlswire_tests.nv` and `tests/tlshs_tests.nv`,
  red until the bodies land.

### Named as missing

Four primitives this package needs and this grid does not have, argued
in the README: ECDSA sign and verify over P-256, RSA verification, an
AES block cipher with GCM over it, and SHA-384.  And one package row —
`tls-core-nv`, the six `[]` modules — which the README recommends
taking.
