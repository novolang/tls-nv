# tls-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

TLS 1.3 (RFC 8446), written in novo-lang: the record layer, the
handshake message codec, RFC 8446 § 7.1's key schedule, certificate
chain validation against a root store the caller supplies, and a client
and server handshake state machine that performs nothing at all.  The
connection half sits over `std.net` and answers the shape `std.tls`
already has, so the host clients published this week can pass it where
they declared a TLS hook.

It is TLS 1.3 and nothing else.  There is no TLS 1.2, no renegotiation,
no compression, no session cache and no 0-RTT, and the section at the
bottom says why for each.

## Adding it, and checking it

```bash
novo pkg add tls-nv          # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/tlswire_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: tls-nv.<module>.<fn>`.  They turn
green one at a time as bodies land.

## The one example that will work

```novo
use tlscert
use tlshs
use tlsconn

// Fetch one page over TLS 1.3, verifying the chain against the
// platform's trust store.
//
// `checker` is not optional and there is no default: see "What is
// missing" — no ECDSA or RSA signature verification is published on
// this grid yet, so the caller supplies one.
fn fetch(host: Str, checker: TlsFnSigCheck) -> Result<Bytes, TlsConnFault> [net, time, fs]
    let roots = tlsconn.system_roots()!
    let cfg = tlshs.with_alpn(tlshs.client_config(host, roots), ["http/1.1"])

    let conn = tlsconn.dial(host, 443, cfg, checker)!
    let _ = tlsconn.send(conn, bytes.from_str("GET / HTTP/1.1\r\nHost: ${host}\r\n\r\n"))!
    let body = tlsconn.recv(conn, 16384)!
    let _ = tlsconn.close(conn)
    Ok(body)
```

## The layer, and why

`host`, and six of the seven modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `tlsrec` | `[]` throughout | the record layer: five header bytes and two ceilings |
| `tlsmsg` | `[]` throughout | handshake messages and extensions, as values |
| `tlssuite` | `[]` throughout | the cipher suites, the nonce and the AEAD |
| `tlskey` | `[]` throughout | RFC 8446 § 7.1's key schedule and the transcript |
| `tlscert` | `[]`, plus `[e]` on two | chain validation; the two are effect-polymorphic over the caller's signature checker |
| `tlshs` | `[]` throughout | the handshake state machine, client and server |
| `tlsconn.dial`, `.accept` | `[net, time]` | the socket, and the clock a validity window needs |
| `tlsconn.send`, `.recv`, `.close` | `[net]` | `std.net`'s own row |
| `tlsconn.now_civil` | `[time]` | the one function in the package that reads a clock |
| `tlsconn.system_roots` | `[fs]` | the one function that reads the platform's trust store |
| `tlsconn.handshake_over` | `[e]` | effect-POLYMORPHIC: whatever the caller's byte pipe costs |

`layer = "host"` and **not** `layer = "core"` with `host_modules`,
because this package's subject is a TLS connection over a socket and a
`core` consumer cannot have one.  The next section is the argument for
moving the six `[]` modules somewhere a `core` consumer can.

## `tls-core-nv` should be a row, and here is the case

**Take it.**  This is the one recommendation in this README that is a
request rather than a description.

The evidence is in the table above: six modules out of seven are `[]`
throughout, and `tlsconn` — the whole of what performs anything — is
about two hundred lines of pumping over an interface the other six
define.  That is not a package with a pure corner; it is a `core`
package with a socket adapter stapled to it.

Three consumers want the `core` half and cannot have this package:

- **A device.**  An embedded client speaking HTTPS over a radio that is
  not `std.net` needs the record layer, the key schedule and the state
  machine, and needs them at `@tier(embedded)`.  It cannot take a
  dependency that names `std.net` anywhere in its assembly, because one
  host-only function in a compilation unit is an undefined symbol at
  embedded link time whether or not the firmware calls it.
- **A server that is not a socket server.**  A TLS terminator in front
  of an existing event loop, or a QUIC stack, wants the handshake and
  the key schedule with its own I/O underneath.
- **A test, a fuzzer and an analysis tool.**  RFC 8448's recorded
  traces are the oracle for every derivation in `tlskey`, and running
  them wants the arithmetic and not a connection.

The split is clean because it was designed to be: `tlsconn` depends on
the other six and nothing depends on it, so `tls-core-nv` is
`tlsrec`, `tlsmsg`, `tlssuite`, `tlskey`, `tlscert` and `tlshs`
unchanged, and `tls-nv` keeps `tlsconn` and takes a dependency on it.
Every module comment already says which side it is on.

What stops the split happening in this release is that a second package
is a second name on the Orbit grid, and what goes on the grid is the
grid's decision rather than one package author's.  So the split is
**argued here rather than made**, and the modules are already arranged
for the day it is taken.

Two smaller notes for whoever takes it.  `tlscert` would need a device
claim of its own or would stay behind — asn1-nv's own README says it
makes no device claim, so a `tls-core-nv` that claimed `@tier(embedded)`
would carry `tlsrec`, `tlssuite`, `tlskey` and `tlshs` and leave
certificate validation with the host.  And `tlshs`'s configuration
structs hold a `TlsRootStore`, which would move with `tlscert`; the
cleaner arrangement is for the session to hold the chain and the caller
to hold the policy, which is already how `TlsAction` is shaped.

## The load-bearing interface

**`TlsAction`**, and the property it encodes is an absence.

A `TlsSession` is a handshake in progress.  It is fed bytes and it
answers one of eight actions:

| action | what the host does |
| --- | --- |
| `TlsWantMoreBytes(least)` | read from the peer and `feed` |
| `TlsSendBytes(length)` | `take_outgoing` and write to the peer |
| `TlsDeliverPlaintext(length)` | `take_plaintext` and hand it to the application |
| `TlsCheckCertificate(count)` | run `tlscert.verify_chain`, answer with `supply_certificate_verdict` |
| `TlsSignTranscript(scheme)` | sign `pending_signature_input`, answer with `supply_signature` |
| `TlsPerformKeyExchange(group)` | run the Diffie-Hellman, answer with `supply_shared_secret` |
| `TlsHandshakeDone` | ask `alpn_of` and `suite_of` what was agreed |
| `TlsPeerClosed` | stop reading |

**There is no variant that reports something this package did.**  That
is the claim: if the enum could say "I sent it" or "I checked it", then
something in here would have a socket or a key, and it does not.  The
randomness, the ephemeral key exchange, the private key and the clock
are all on the other side of that boundary, held by whoever should hold
them.

What it buys is concrete rather than architectural.  A whole handshake
against RFC 8448's recorded traces is a test with no network and no
timing in it that produces the same bytes every run — which is not
something an implementation that generated its own randomness can write.
A device pumps the same value over a radio this package has never heard
of.  A client and a server share one state machine instead of two that
drift apart.  And a secure element that will not export a private key is
an ordinary case rather than a fork: the signature is an argument.

The second decision is **`TlsSigCheck[e]`**, and it is load-bearing for
a reason that is not architecture at all — see the next section.

## What is missing, by name

This package cannot complete a handshake today, and the reason is four
primitives that are not on this grid.  Each one is an honest missing row
rather than something to route around.

- **ECDSA signing and verification over P-256.**  `p256-nv` 0.0.2
  carries ECDH and nothing else: `secret_key_from_bytes`,
  `public_key_of`, `ecdh`, and no `sign` or `verify` anywhere.  Every
  certificate chain on the modern web needs verification and every
  CertificateVerify needs it, so a TLS client without it verifies
  nothing.  The row is **p256-nv growing `ecdsa_sign` and
  `ecdsa_verify`**, or `ecdsa-nv` beside it, crypto/core.
- **RSA, at all.**  `jwt-nv` declares `rsa_signing_key(pkcs1_der)` and
  there is no RSA package under it.  A large share of the public web is
  still RSA-signed, so a client with ECDSA and no RSA fails against real
  servers.  The row is **`rsa-nv`**, crypto/core — PKCS#1 v1.5 and
  RSASSA-PSS verification at minimum, over bigint-nv.
- **An AES block cipher, and AES-GCM over it.**  crypto-nv's plan row
  names "AES-GCM, ChaCha20-Poly1305" as its subject and the published
  0.1.2 carries SHA-1, SHA-256, SHA-512, MD5 and HMAC — no block cipher
  of any kind.  `TLS_AES_128_GCM_SHA256` is the one suite RFC 8446 § 9.1
  says every implementation MUST offer, and `tlssuite.suite_is_available`
  answers false for it.  The row is **AES-128 and AES-256 in crypto-nv,
  and GCM over them**; p256-nv's own plan row already names AES-128 and
  AES-CMAC as content that was meant to arrive with it.
- **SHA-384.**  hkdf-nv carries SHA-256, SHA-512 and SHA-1, and
  crypto-nv under it the same three plus MD5.  `TLS_AES_256_GCM_SHA384`
  has no hash to run its key schedule on and `TlsEcdsaP384Sha384` has
  nothing to digest with, so `tlssuite.suite_hash` answers `?HkdfHash`
  and `None` for the SHA-384 suite rather than substituting one.  The
  row is **SHA-384 in crypto-nv and `HkdfSha384` in hkdf-nv**, which is
  a truncated SHA-512 with different initial values and is the smallest
  of the four.

What this package does instead of pretending: `TlsSigCheck[e]` is a
trait the caller implements, `tlscert.no_checker()` is a checker that
supports nothing and refuses everything, and
`tlssuite.suite_is_available` and `tlsmsg.group_is_available` answer
honestly so a client offers only what it can complete.  When the rows
land, an implementation of `TlsSigCheck` over p256-nv ships with this
package and `no_checker` stops being the sensible default.

## Where a row wanted to widen

Three places, and all three are recorded because they are design input
rather than complaints.

**One effect parameter per function, and this package wanted two.**
`tlsconn.handshake_over` is generic over the caller's byte pipe
(`TlsBytePipe[e]`) and wanted to be generic over the caller's signature
checker at the same time.  The compiler refuses it:

```
effect error: function 'handshake_over' binds 2 effect parameters
(P: TlsBytePipe[e], V: TlsSigCheck[e]) — exactly one is supported.
```

The hint in the refusal is the design that shipped: keep one bound
parameterised and make the other concrete.  So the pipe carries the
polymorphism, the checker is `tlscert.TlsFnSigCheck` — a pair of named
functions with an `impl TlsSigCheck[] for TlsFnSigCheck` — and a caller
whose verification costs something (`[hw]` for a secure element,
`[net]` for a remote signer) drives `tlshs` and `tlscert.verify_chain`
directly, where the effect parameter is free.  That is a good line to
have drawn, and it is still a line the language chose rather than the
design.

**`[fs]` in a package whose row the plan wrote as `[net]`.**
`tlsconn.system_roots` reads the platform's certificate bundle off
disk.  It is one function, it is named after what it does, and a caller
that supplies its own roots never calls it — but the package's declared
union is `[fs, net, time]` and not `[net, time]`.  The alternative was
to leave the system store to the caller entirely, which makes the
first-time user's first connection fail with "no trust anchor" and is a
worse trade.

**`[time]` for one clock read.**  A certificate's validity window is
checked against an instant, and `tlscert` is `[]`, so the instant is an
argument everywhere and `tlsconn.now_civil` is the single function that
produces one.  Same shape as smtp-nv's `now_ms` and postgres-nv's,
and worth recording as a pattern the cohort now has three instances of.

## What this does not do, on purpose

- **No TLS 1.2, and no fallback to it.**  The `supported_versions`
  extension this package writes names 1.3 and nothing else.  Offering
  1.2 beside it is what makes a downgrade something a middlebox can
  force, and there would be nothing to fall back to: 1.2's key schedule,
  its record layer and its handshake are a different protocol that
  happens to share a name.
- **No 0-RTT.**  Early data is not replay protected — the specification
  says so in RFC 8446 § 2.3 and § 8 — and a library that offered it as
  one option among several makes a replayable request the default for
  whoever did not read this paragraph.  Adding it would be a new
  `TlsAction` variant, an anti-replay window the SERVER has to keep, and
  a `TlsClientConfig` field that cannot have a safe default.  Worth
  doing when a consumer needs it and can say what it will send.
- **No renegotiation and no session cache.**  1.3 removed
  renegotiation; resumption is a ticket the caller stores, and
  `tlshs.tickets_of` hands the tickets over.  Where they are kept is a
  policy — memory, a file, a shared store — and a library that chose
  one would be wrong for the other two.
- **No revocation checking.**  OCSP and CRLs are a network fetch and a
  policy about what to do when it fails, which is the question that
  makes revocation hard rather than the parsing.  `tlscert` exposes
  everything a caller needs to do it (`spki_of`, `signed_data`, the
  chain) and does none of it.  An `ocsp-nv` row would be the place.
- **It does not resolve names.**  `tlsconn.dial` takes a host and hands
  it to `std.net`.
- **No device claim.**  The package is `host`.  The `tls-core-nv`
  section above is where the device claim would live.
- **It does not print.**  Every failure is a value with a `message()`.

## The reference implementation

`rustls` for the shape, and its `ClientConnection`/`ServerConnection`
split into a sans-IO core with an I/O adapter over it is the design this
package ports.  RFC 8446 is the specification, RFC 8448 supplies
recorded traces for every derivation in `tlskey`, and RFC 5280 is what
`tlscert` is a policy over.

Four things change in the port.

`rustls` pumps its connection through `read_tls`/`write_tls` against a
`std::io` trait; here the pump answers `TlsAction` values and the host
does the reading, because Novo's effect rows make "this function
performs nothing" a claim the compiler checks and a trait-shaped pump
would have spent it.

`rustls` owns its cryptographic providers behind a `CryptoProvider`
trait with `ring` and `aws-lc-rs` behind it; here the signature checker
is the caller's and the key exchange is an action, for the reason the
"What is missing" section gives — the providers do not exist yet, and a
design that assumed one would have to be unpicked when they arrive.

`rustls` has a `RootCertStore` that can be loaded from
`webpki-roots`, a compiled-in copy of Mozilla's list; here
`TlsRootStore` is a value and the compiled-in list is not this
package's to ship.  A `webpki-roots-nv` row would be a data package with
its own update cadence, the same argument cookie-nv made for the Public
Suffix List.

And `rustls` splits client and server into separate types with a shared
`ConnectionCommon`; here there is one `TlsSession` with a `TlsRole`,
because the two halves of a TLS 1.3 handshake are far more alike than
they were in 1.2 and two types would mean two state machines to keep in
agreement.

## Status

| item | implemented |
| --- | --- |
| `tlsrec` — `TlsContentKind`, `TlsRecordHead`, `TlsRecordStep`, `TlsRecordReader`, `TlsRecordFault` | types only |
| `tlsrec.TLS_RECORD_HEADER_BYTES`, `.TLS_PLAINTEXT_MAX`, `.TLS_CIPHERTEXT_MAX`, `.TLS_LEGACY_RECORD_VERSION` | yes — they are constants |
| `tlsrec.kind_code`, `.kind_of_code`, `.reader`, `.reader_with`, `.feed`, `.take`, `.pending_len`, `.fragment_of` | no |
| `tlsrec.head_at`, `.write_head_into`, `.inner_kind`, `.inner_len`, `.pad_into`, `.ciphertext_len`, `.is_legacy_version`, `.limit_for`, the `message` impl | no |
| `tlssuite` — `TlsCipherSuite`, `TlsAeadFault` | types only |
| `tlssuite.suite_code`, `.suite_of_code`, `.suite_name`, `.suite_hash`, `.suite_key_bytes`, `.suite_iv_bytes`, `.suite_tag_bytes` | no |
| `tlssuite.suite_is_available`, `.default_suites`, `.known_suites`, `.sequence_limit` | no |
| `tlssuite.record_nonce`, `.record_aad`, `.seal_into`, `.open_into`, the `message` impl | no |
| `tlsmsg` — `TlsRole`, `TlsMsgKind`, `TlsNamedGroup`, `TlsSigScheme`, `TlsExtKind`, `TlsExtension`, `TlsKeyShareEntry`, `TlsClientHello`, `TlsServerHello`, `TlsCertEntry`, `TlsCertificateMsg`, `TlsCertVerifyMsg`, `TlsTicket`, `TlsMsgLimits`, `TlsMsgFault` | types only |
| `tlsmsg.msg_kind_code`, `.msg_kind_of_code`, `.group_code`, `.group_of_code`, `.group_key_bytes`, `.group_is_available`, `.default_groups` | no |
| `tlsmsg.scheme_code`, `.scheme_of_code`, `.default_schemes`, `.ext_kind_code`, `.ext_kind_of_code`, `.extension_by_kind` | no |
| `tlsmsg.default_limits`, `.embedded_limits`, `.msg_head_at` | no |
| `tlsmsg.parse_client_hello`, `.parse_server_hello`, `.parse_encrypted_extensions`, `.parse_certificate`, `.parse_certificate_verify`, `.parse_finished`, `.parse_ticket`, `.parse_key_update` | no |
| `tlsmsg.write_client_hello`, `.write_server_hello`, `.write_encrypted_extensions`, `.write_certificate`, `.write_certificate_verify`, `.write_finished`, `.write_key_update` | no |
| `tlsmsg.is_hello_retry_request`, `.hello_retry_random`, `.selected_version`, `.server_name_of`, `.alpn_pick`, `.verify_context` | no |
| `tlsmsg.server_name_extension`, `.alpn_extension`, `.key_share_extension`, `.supported_versions_extension`, the `message` impl | no |
| `tlskey` — `TlsTranscript`, `TlsSecrets`, `TlsTrafficKeys`, `TlsKeyFault` | types only |
| `tlskey.TLS_LABEL_PREFIX`, `.TLS_MESSAGE_HASH_TYPE` | yes — they are constants |
| `tlskey.transcript`, `.transcript_update`, `.transcript_hash`, `.transcript_len`, `.transcript_replace`, `.empty_transcript_hash` | no |
| `tlskey.hkdf_expand_label`, `.derive_secret`, `.schedule`, `.with_psk`, `.without_psk`, `.with_shared_secret` | no |
| `tlskey.advance_to_handshake`, `.advance_to_application`, `.handshake_secret_of`, `.traffic_secret_of`, `.traffic_keys`, `.next_sequence` | no |
| `tlskey.finished_key`, `.finished_verify`, `.finished_matches`, `.resumption_secret`, `.ticket_psk`, `.update_traffic_secret`, `.exporter`, the `message` impl | no |
| `tlscert` — `TlsRootStore`, `TlsCertLimits`, `TlsVerified`, `TlsVerifyFault`, `TlsSigCheck[e]`, `TlsFnSigCheck` | types only |
| `tlscert.checker_of`, `.no_checker`, `.root_store`, `.empty_root_store`, `.root_count`, `.default_cert_limits`, `.embedded_cert_limits` | no |
| `tlscert.name_matches`, `.wildcard_matches`, `.dns_names_of`, `.valid_at`, `.is_ca`, `.path_len_of`, `.allows_server_auth`, `.allows_client_auth` | no |
| `tlscert.signed_data`, `.signature_scheme_of`, `.signature_of`, `.spki_of`, `.issued_by` | no |
| `tlscert.verify_chain`, `.verify_handshake_signature`, `.schemes_supported`, both impls | no |
| `tlshs` — `TlsClientConfig`, `TlsServerConfig`, `TlsHsState`, `TlsAlert`, `TlsAction`, `TlsStep`, `TlsSession`, `TlsHsFault` | types only |
| `tlshs.client_config`, `.with_alpn`, `.with_suites`, `.with_groups`, `.with_schemes`, `.with_limits` | no |
| `tlshs.server_config`, `.with_server_alpn`, `.with_client_auth`, `.with_server_schemes` | no |
| `tlshs.client`, `.server`, `.begin`, `.feed`, `.take_outgoing`, `.take_plaintext` | no |
| `tlshs.pending_chain`, `.pending_key_exchange`, `.pending_signature_input` | no |
| `tlshs.supply_shared_secret`, `.supply_certificate_verdict`, `.supply_signature` | no |
| `tlshs.send_application_data`, `.request_key_update`, `.close`, `.abort` | no |
| `tlshs.state_of`, `.alpn_of`, `.suite_of`, `.peer_chain_of`, `.is_handshake_done`, `.tickets_of` | no |
| `tlshs.alert_code`, `.alert_of_code`, `.is_fatal`, the `message` impl | no |
| `tlsconn` — `TlsBytePipe[e]`, `TlsTcpPipe`, `TlsConn`, `TlsConnFault` | types only |
| `tlsconn.dial`, `.accept`, `.handshake_over`, `.send`, `.recv`, `.close` | no |
| `tlsconn.now_civil`, `.system_roots`, `.roots_from_pem`, `.certificates_from_pem`, `.certificates_to_pem` | no |
| `tlsconn.alpn_of`, `.suite_of`, `.peer_chain_of`, `.session_of`, `.stream_of`, `.pipe_of` | no |
| `tlsconn` — the `TlsBytePipe` impl and the `message` impl | no |
