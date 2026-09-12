# tls-nv

TLS is the protocol that encrypts and authenticates most traffic on the
internet. Version 1.3 is specified in
[RFC 8446](https://www.rfc-editor.org/rfc/rfc8446). This package implements it
in novo-lang: the record layer, the handshake message codec, the key schedule,
certificate chain validation against a root store the caller supplies, and a
client and server handshake state machine that performs no input or output at
all.

The connection half sits over the standard library's networking and answers the
shape `std.tls` already has, so a client that declared a TLS hook can pass this
package where it expects one.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

This is TLS 1.3 and nothing else. There is no TLS 1.2, no renegotiation, no
compression, no session cache and no 0-RTT. "What is not included" says why for
each.

## What TLS 1.3 is

A TLS connection has two parts.

**The record layer** frames everything. A record is a five-byte header — a
content type, a legacy version number and a two-byte length — followed by at
most 16384 bytes of plaintext or 16640 bytes of ciphertext. After the handshake
begins, each record's payload is sealed with authenticated encryption, and the
real content type is the last non-zero byte inside the sealed payload.

**The handshake** agrees on keys. The client sends a `ClientHello` carrying its
supported cipher suites, the named groups it can do Diffie-Hellman over, and a
*key share* for the group it expects the server to pick. The server answers a
`ServerHello` with its own key share, and from that point both sides can compute
the same secrets. Everything after that is encrypted: the server's certificate,
its signature over the handshake so far, and a `Finished` message from each side
that proves both saw the same messages.

Between those two sits **the key schedule** (RFC 8446 section 7.1): a chain of
HKDF extract and expand steps that turns the Diffie-Hellman output and a running
hash of every handshake message into the traffic keys.

| Value | Size |
| --- | --- |
| Record header | 5 bytes |
| Largest plaintext record | 16384 bytes |
| Largest ciphertext record | 16640 bytes |

## Install

```
novo pkg add tls-nv
```

## Example

```novo
use tlscert
use tlshs
use tlsconn

// Fetch one page over TLS 1.3, verifying the chain against the
// platform's trust store.
//
// `checker` is not optional and there is no default: see "What is
// missing" — no ECDSA or RSA signature verification is published on
// the registry yet, so the caller supplies one.
fn fetch(host: Str, checker: TlsFnSigCheck) -> Result<Bytes, TlsConnFault> [net, time, fs]
    let roots = tlsconn.system_roots()!
    let cfg = tlshs.with_alpn(tlshs.client_config(host, roots), ["http/1.1"])

    let conn = tlsconn.dial(host, 443, cfg, checker)!
    let _ = tlsconn.send(conn, bytes.from_str("GET / HTTP/1.1\r\nHost: ${host}\r\n\r\n"))!
    let body = tlsconn.recv(conn, 16384)!
    let _ = tlsconn.close(conn)
    Ok(body)
```

Build and test with:

```
novo pkg build               # type- and effect-check the package
novo test --isolate tests/tlswire_tests.nv
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: tls-nv.<module>.<fn>`. The tests are the specification the
implementation will have to satisfy.

## What the package contains

Six modules perform no input or output. One does.

| Module | Contents | Effects |
| --- | --- | --- |
| `tlsrec` | The record layer: the header, the two ceilings, and a reader that is fed bytes and hands back whole records. | none |
| `tlsmsg` | The handshake messages and their extensions, as values, with a parser and a writer for each. | none |
| `tlssuite` | The cipher suites: their code points, their key, nonce and tag sizes, and the sealing and opening of a record. | none |
| `tlskey` | The transcript hash and RFC 8446 section 7.1's key schedule. | none |
| `tlscert` | Chain validation: name matching, validity windows, basic constraints, key usage, and the signature checker interface. | none, except two functions that cost whatever the caller's signature checker costs |
| `tlshs` | The handshake state machine, client and server, as one type with a role. | none |
| `tlsconn` | The connection: dialling, accepting, sending, receiving and closing. | `[net]`, plus `[time]` for the one clock read a certificate's validity window needs, and `[fs]` for the one function that reads the platform's trust store off disk |

`tlsconn` depends on the other six, and nothing depends on `tlsconn`. It is
about two hundred lines of pumping over the interface the other six define.

## How to choose an entry point

**A program that wants an ordinary TLS connection calls `tlsconn`.**
`tlsconn.dial` for a client, `tlsconn.accept` for a server, then `send`, `recv`
and `close`.

**A program with its own transport drives `tlshs` directly.** Feed it bytes,
act on the `TlsAction` it answers, and put the bytes wherever they go. A TLS
terminator in front of an existing event loop, a QUIC stack, or a device
speaking over a radio all work this way.

**A test or an analysis tool calls `tlskey`, `tlsmsg` and `tlsrec` on their
own.** RFC 8448's recorded traces are the oracle for every derivation in
`tlskey`, and running them needs the arithmetic and no connection.

## Driving the handshake yourself

A `TlsSession` is a handshake in progress. It is fed bytes and it answers one of
eight actions.

| Action | What the caller does |
| --- | --- |
| `TlsWantMoreBytes(least)` | Read from the peer and call `feed`. |
| `TlsSendBytes(length)` | Call `take_outgoing` and write to the peer. |
| `TlsDeliverPlaintext(length)` | Call `take_plaintext` and hand it to the application. |
| `TlsCheckCertificate(count)` | Run `tlscert.verify_chain`, answer with `supply_certificate_verdict`. |
| `TlsSignTranscript(scheme)` | Sign `pending_signature_input`, answer with `supply_signature`. |
| `TlsPerformKeyExchange(group)` | Run the Diffie-Hellman, answer with `supply_shared_secret`. |
| `TlsHandshakeDone` | Ask `alpn_of` and `suite_of` what was agreed. |
| `TlsPeerClosed` | Stop reading. |

No action reports something this package did. There is no "I sent it" and no "I
checked it", because nothing in here holds a socket, a private key, a random
number generator or a clock. All four are on the caller's side of that boundary.

Three consequences follow.

- A whole handshake against RFC 8448's recorded traces is a test with no network
  and no timing in it, producing the same bytes every run.
- A secure element that will not export a private key is an ordinary case: the
  signature arrives as an argument.
- The client and the server share one state machine rather than two that drift
  apart.

## The rules a user needs

1. **A signature checker is required and there is no default.**
   `tlscert.no_checker()` supports nothing and refuses everything, which is what
   ships today. See "What is missing".
2. **`tlssuite.suite_is_available` and `tlsmsg.group_is_available` answer
   honestly**, so a client offers only what it can actually complete. Do not
   assume a suite is usable because it has a code point.
3. **The validity window is checked against an instant you supply.** `tlscert`
   reads no clock, so `tlscert.valid_at` takes the instant as an argument.
   `tlsconn.now_civil` is the one function in the package that produces one.
4. **A function can be generic over one caller-supplied effect, not two.**
   `tlsconn.handshake_over` is generic over the caller's byte pipe, so its
   signature checker is concrete: `tlscert.TlsFnSigCheck`, a pair of named
   functions. A caller whose verification itself costs something — a secure
   element, a remote signer — drives `tlshs` and `tlscert.verify_chain` directly
   instead, where the choice is free.
5. **Resumption tickets are yours to store.** `tlshs.tickets_of` hands them
   over. This package keeps no cache.
6. **Nothing here prints.** Every failure is a value with a `message()`.

## What is missing, by name

This package cannot complete a handshake today. Four primitives are not on the
registry.

- **ECDSA signing and verification over P-256.**
  [p256-nv](https://novo-lang.org/packages/p256-nv) 0.0.2 carries ECDH and
  nothing else: `secret_key_from_bytes`, `public_key_of`, `ecdh`, and no `sign`
  or `verify`. Every certificate chain on the modern web needs verification and
  so does every `CertificateVerify` message.
- **RSA, at all.** jwt-nv declares an RSA signing key type and no package
  implements RSA. A large share of the public web is still RSA-signed, so a
  client with ECDSA and no RSA fails against real servers. PKCS#1 v1.5 and
  RSASSA-PSS verification are the minimum.
- **An AES block cipher, and AES-GCM over it.**
  [crypto-nv](https://novo-lang.org/packages/crypto-nv) 0.1.2 carries SHA-1,
  SHA-256, SHA-512, MD5 and HMAC, and no block cipher of any kind.
  `TLS_AES_128_GCM_SHA256` is the one suite RFC 8446 section 9.1 says every
  implementation MUST offer, and `tlssuite.suite_is_available` answers false for
  it.
- **SHA-384.** [hkdf-nv](https://novo-lang.org/packages/hkdf-nv) carries
  SHA-256, SHA-512 and SHA-1, and crypto-nv under it the same three plus MD5.
  `TLS_AES_256_GCM_SHA384` has no hash to run its key schedule on and
  `TlsEcdsaP384Sha384` has nothing to digest with, so `tlssuite.suite_hash`
  answers `None` for the SHA-384 suite rather than substituting one. SHA-384 is
  a truncated SHA-512 with different initial values, and is the smallest of the
  four gaps.

When those arrive, a signature checker built on p256-nv will ship with this
package and `no_checker` will stop being the sensible default.

## What is not included

- **TLS 1.2, and any fallback to it.** The `supported_versions` extension this
  package writes names 1.3 and nothing else. Offering 1.2 beside it is what lets
  a middlebox force a downgrade, and there would be nothing to fall back to:
  1.2's key schedule, record layer and handshake are a different protocol that
  happens to share a name.
- **0-RTT early data.** It is not replay protected, which RFC 8446 sections 2.3
  and 8 say plainly. Adding it would need a new action, an anti-replay window
  the server has to keep, and a client configuration field that cannot have a
  safe default.
- **Renegotiation.** TLS 1.3 removed it.
- **A session cache.** See rule 5.
- **Revocation checking.** OCSP and certificate revocation lists are a network
  fetch plus a policy about what to do when it fails, and the policy is the hard
  part. `tlscert` exposes what a caller needs to do it — `spki_of`,
  `signed_data`, the chain — and does none of it.
- **Name resolution.** `tlsconn.dial` takes a host and hands it to the standard
  library.
- **A compiled-in list of root certificates.** `TlsRootStore` is a value.
  `tlsconn.system_roots` reads the platform's bundle, and
  `tlsconn.roots_from_pem` takes one you supply. A copy of a certificate
  authority list is a data package with its own update schedule.
- **A build for a microcontroller with no heap allocator.** The six modules that
  perform nothing are shaped so they could be split into a package that makes
  such a claim; this one, which reaches a socket, cannot.

## Related packages

| Package | What it supplies |
| --- | --- |
| [hkdf-nv](https://novo-lang.org/packages/hkdf-nv) | HKDF extract and expand, which the whole of RFC 8446 section 7.1 is built from. `tlskey.hkdf_expand_label` is the structured wrapper the specification puts over expand, and getting that structure wrong is what this package's key-schedule tests exist to catch. |
| [crypto-nv](https://novo-lang.org/packages/crypto-nv) | SHA-256 for the transcript hash, HMAC through hkdf-nv, and `digest.ct_eq` for the `Finished` comparison — the one comparison in the handshake an attacker gets to iterate against. |
| asn1-nv | X.509 and PKCS#8: the certificate type, the subject alternative name extension, basic constraints, key usage, the extended key usage identifiers, validity against a civil date, and the fixed-width signature form TLS 1.3 uses. `tlscert` is a policy over that parser and parses no DER of its own. |
| [x25519-nv](https://novo-lang.org/packages/x25519-nv) | The key exchange every TLS 1.3 deployment actually uses, and the only group this package's default offer names first. |
| [p256-nv](https://novo-lang.org/packages/p256-nv) | The secp256r1 key exchange, for deployments that require a NIST curve. |
| calendar-nv | Civil dates, for the certificate validity window. |
| base64-nv | PEM, which is base64 with a header and a footer. `tlsconn.roots_from_pem` is how a caller's trust store gets from a file into a root store. |
| [chacha20-nv](https://novo-lang.org/packages/chacha20-nv) | ChaCha20-Poly1305, for `TLS_CHACHA20_POLY1305_SHA256`. |

## The reference implementation

`rustls` is the reference for the shape, and its split into a connection core
that performs nothing with an input/output adapter over it is the design this
package ports. RFC 8446 is the specification, RFC 8448 supplies recorded traces
for every derivation in `tlskey`, and RFC 5280 is what `tlscert` is a policy
over.

Four things differ from `rustls`.

It pumps its connection through read and write methods against the standard
library's I/O trait. Here the pump answers `TlsAction` values and the caller
does the reading, which is what makes "this function performs nothing" something
the compiler checks.

It owns its cryptographic providers behind a trait with two implementations
underneath. Here the signature checker is the caller's and the key exchange is
an action, because those providers do not exist yet and a design that assumed
one would have to be unpicked when they arrive.

Its root certificate store can be loaded from a compiled-in copy of Mozilla's
list. Here that list is not this package's to ship.

And it splits client and server into separate types over a shared core. Here
there is one session type with a role, because the two halves of a TLS 1.3
handshake are far more alike than they were in 1.2, and two types would mean two
state machines to keep in agreement.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tlsrec` — `TlsContentKind`, `TlsRecordHead`, `TlsRecordStep`, `TlsRecordReader`, `TlsRecordFault` | types only |
| `tlsrec.TLS_RECORD_HEADER_BYTES`, `.TLS_PLAINTEXT_MAX`, `.TLS_CIPHERTEXT_MAX`, `.TLS_LEGACY_RECORD_VERSION` | yes (they are constants) |
| `tlsrec.kind_code`, `.kind_of_code`, `.reader`, `.reader_with`, `.feed`, `.take`, `.pending_len`, `.fragment_of` | no |
| `tlsrec.head_at`, `.write_head_into`, `.inner_kind`, `.inner_len`, `.pad_into`, `.ciphertext_len`, `.is_legacy_version`, `.limit_for`, the `message` implementation | no |
| `tlssuite` — `TlsCipherSuite`, `TlsAeadFault` | types only |
| `tlssuite.suite_code`, `.suite_of_code`, `.suite_name`, `.suite_hash`, `.suite_key_bytes`, `.suite_iv_bytes`, `.suite_tag_bytes` | no |
| `tlssuite.suite_is_available`, `.default_suites`, `.known_suites`, `.sequence_limit` | no |
| `tlssuite.record_nonce`, `.record_aad`, `.seal_into`, `.open_into`, the `message` implementation | no |
| `tlsmsg` — `TlsRole`, `TlsMsgKind`, `TlsNamedGroup`, `TlsSigScheme`, `TlsExtKind`, `TlsExtension`, `TlsKeyShareEntry`, `TlsClientHello`, `TlsServerHello`, `TlsCertEntry`, `TlsCertificateMsg`, `TlsCertVerifyMsg`, `TlsTicket`, `TlsMsgLimits`, `TlsMsgFault` | types only |
| `tlsmsg.msg_kind_code`, `.msg_kind_of_code`, `.group_code`, `.group_of_code`, `.group_key_bytes`, `.group_is_available`, `.default_groups` | no |
| `tlsmsg.scheme_code`, `.scheme_of_code`, `.default_schemes`, `.ext_kind_code`, `.ext_kind_of_code`, `.extension_by_kind` | no |
| `tlsmsg.default_limits`, `.embedded_limits`, `.msg_head_at` | no |
| `tlsmsg.parse_client_hello`, `.parse_server_hello`, `.parse_encrypted_extensions`, `.parse_certificate`, `.parse_certificate_verify`, `.parse_finished`, `.parse_ticket`, `.parse_key_update` | no |
| `tlsmsg.write_client_hello`, `.write_server_hello`, `.write_encrypted_extensions`, `.write_certificate`, `.write_certificate_verify`, `.write_finished`, `.write_key_update` | no |
| `tlsmsg.is_hello_retry_request`, `.hello_retry_random`, `.selected_version`, `.server_name_of`, `.alpn_pick`, `.verify_context` | no |
| `tlsmsg.server_name_extension`, `.alpn_extension`, `.key_share_extension`, `.supported_versions_extension`, the `message` implementation | no |
| `tlskey` — `TlsTranscript`, `TlsSecrets`, `TlsTrafficKeys`, `TlsKeyFault` | types only |
| `tlskey.TLS_LABEL_PREFIX`, `.TLS_MESSAGE_HASH_TYPE` | yes (they are constants) |
| `tlskey.transcript`, `.transcript_update`, `.transcript_hash`, `.transcript_len`, `.transcript_replace`, `.empty_transcript_hash` | no |
| `tlskey.hkdf_expand_label`, `.derive_secret`, `.schedule`, `.with_psk`, `.without_psk`, `.with_shared_secret` | no |
| `tlskey.advance_to_handshake`, `.advance_to_application`, `.handshake_secret_of`, `.traffic_secret_of`, `.traffic_keys`, `.next_sequence` | no |
| `tlskey.finished_key`, `.finished_verify`, `.finished_matches`, `.resumption_secret`, `.ticket_psk`, `.update_traffic_secret`, `.exporter`, the `message` implementation | no |
| `tlscert` — `TlsRootStore`, `TlsCertLimits`, `TlsVerified`, `TlsVerifyFault`, `TlsSigCheck[e]`, `TlsFnSigCheck` | types only |
| `tlscert.checker_of`, `.no_checker`, `.root_store`, `.empty_root_store`, `.root_count`, `.default_cert_limits`, `.embedded_cert_limits` | no |
| `tlscert.name_matches`, `.wildcard_matches`, `.dns_names_of`, `.valid_at`, `.is_ca`, `.path_len_of`, `.allows_server_auth`, `.allows_client_auth` | no |
| `tlscert.signed_data`, `.signature_scheme_of`, `.signature_of`, `.spki_of`, `.issued_by` | no |
| `tlscert.verify_chain`, `.verify_handshake_signature`, `.schemes_supported`, both implementations | no |
| `tlshs` — `TlsClientConfig`, `TlsServerConfig`, `TlsHsState`, `TlsAlert`, `TlsAction`, `TlsStep`, `TlsSession`, `TlsHsFault` | types only |
| `tlshs.client_config`, `.with_alpn`, `.with_suites`, `.with_groups`, `.with_schemes`, `.with_limits` | no |
| `tlshs.server_config`, `.with_server_alpn`, `.with_client_auth`, `.with_server_schemes` | no |
| `tlshs.client`, `.server`, `.begin`, `.feed`, `.take_outgoing`, `.take_plaintext` | no |
| `tlshs.pending_chain`, `.pending_key_exchange`, `.pending_signature_input` | no |
| `tlshs.supply_shared_secret`, `.supply_certificate_verdict`, `.supply_signature` | no |
| `tlshs.send_application_data`, `.request_key_update`, `.close`, `.abort` | no |
| `tlshs.state_of`, `.alpn_of`, `.suite_of`, `.peer_chain_of`, `.is_handshake_done`, `.tickets_of` | no |
| `tlshs.alert_code`, `.alert_of_code`, `.is_fatal`, the `message` implementation | no |
| `tlsconn` — `TlsBytePipe[e]`, `TlsTcpPipe`, `TlsConn`, `TlsConnFault` | types only |
| `tlsconn.dial`, `.accept`, `.handshake_over`, `.send`, `.recv`, `.close` | no |
| `tlsconn.now_civil`, `.system_roots`, `.roots_from_pem`, `.certificates_from_pem`, `.certificates_to_pem` | no |
| `tlsconn.alpn_of`, `.suite_of`, `.peer_chain_of`, `.session_of`, `.stream_of`, `.pipe_of` | no |
| `tlsconn` — the byte pipe implementation and the `message` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.
