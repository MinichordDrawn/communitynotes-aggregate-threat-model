# Self-Observing System Threat Model: ghostunnel

This document is a model built from public source for the maintainers of ghostunnel, and it contains no exploit code and no operational instructions. It covers ghostunnel at commit `7d81cea` (2026-08-22).

## Risks, stated first

ghostunnel is a mutual-TLS proxy. In server mode it terminates client TLS, requires and verifies a client certificate, applies an access-control list to it, and forwards plaintext to a backend it calls insecure; in client mode it accepts local plaintext and opens an mTLS connection to a remote target. It reloads certificates, CA bundle and OPA policy on a signal or a timer, serves a status and metrics port, and on Linux applies best-effort landlock to itself at startup. Client identities and proxied bytes pass through the one process; in `--proxy-protocol-mode=tls-full` the full DER client certificate is written to the backend.

Risk is severity times detection times exposure, and the ranking uses the product, not severity alone. Severity is the distance to the guarantee ghostunnel exists to hold: an unauthorized client reaching the backend, a private key or credential leaving the process, or one principal acting as another. Detection is whether any observer in the process records that the fault happened. Exposure is who can reach the entry and whether the finding holds under the shipped defaults or needs a flag the operator sets. Ranking on the product moves the quiet, externally reachable findings that hold at the defaults above the louder ones that need a privileged position or a non-default flag. The controlling structural fact behind the top risks: ghostunnel takes the allow decision once, at connection entry, for a full handshake, and never re-takes it over a connection's life or on a TLS session resumption; and on a detected fault it logs and keeps serving.

**1. A client the current policy denies keeps reaching the backend by resuming its TLS session.** Statement: the access-control decision runs only in `VerifyPeerCertificate`, which the TLS stack does not invoke on a resumed connection, and ghostunnel sets neither `VerifyConnection` nor `SessionTicketsDisabled` on the tunnel listener, so a client that completed one full handshake resumes without any ACL evaluation for the ticket lifetime. Entry and boundary: E2 an established or resumed connection, B3. Configuration and default: holds at the shipped default (tickets on, Go default); the window is up to 7 days per ticket, and indefinite after a failed reload, which keeps the old ticket keys; a successful reload of any source except SPIFFE rotates the keys and closes it on the next accept. Reproduced at runtime. Rests on: E1; LB1, LB2, LB3, LB6.

**2. An established connection outlives every revocation.** Statement: after a connection is fused the copy loops run to end of stream, and nothing re-checks the peer against a CA removal, a policy change, or certificate expiry; the only bound is `--max-conn-lifetime`, which defaults to zero, meaning infinite. Entry and boundary: E2, B3. Configuration and default: holds at the shipped default; the behavior is documented (doc.go:253-254). Rests on: E3; LB7.

**3. The keystore password or HSM PIN is served to an unauthenticated caller.** Statement: with `--enable-pprof` the profiling handlers are mounted on the same mux as `/_status`, that server does no client authentication, and `/debug/pprof/cmdline` returns the process argv, so `--storepass` or `--pkcs11-pin` passed as a flag is disclosed to anyone who can reach the status port. Entry and boundary: E4 the status HTTP surface, B3. Configuration and default: needs `--enable-pprof` (off by default), a secret passed as a flag rather than the `KEYSTORE_PASS`/`PKCS11_PIN` env var, and a status port reachable beyond localhost; none needs a certificate. Rests on: I2; LB4, LB5.

**4. Any client that can reach the status port learns the backend and, in client mode, drives ghostunnel's identity on demand.** Statement: the status server is unauthenticated and not restricted to a local interface, and `/_status` returns the listen and forward addresses and the hostname; in client mode each GET makes ghostunnel complete an outbound mTLS handshake with its own certificate against the target and returns whether the target accepted it. Entry and boundary: E4, B3. Configuration and default: needs a status port reachable beyond localhost; `--status` receives none of the local-interface gate its siblings `--target` and `--listen` get. Rests on: I1, I3; LB4, LB5.

**5. Any local process uses ghostunnel's identity through the client-mode plaintext listener.** Statement: in client mode the plaintext `--listen` side is handed to the proxy unwrapped, a unix socket gets no mode change and no peer-credential check, and `forceHandshake` is a no-op for a non-TLS connection, so no observer runs on the caller; the dial then presents ghostunnel's client certificate and copies bytes both ways. Entry and boundary: E7 the plaintext listener, any local process of any uid at the default localhost/unix bind, B3 with `--unsafe-listen`. Configuration and default: holds at the default for a local caller; the network case needs `--unsafe-listen`. Rests on: E4-row; LB7 (E3 applies after: never re-observed). This is the designed function of client mode for a trusted local application; the gap is the absent caller check.

**6. One pinned client asserts any subject to a header-trusting backend.** Statement: in pin mode ghostunnel does no chain validation and the ACL compares only the SPKI digest, so a self-signed certificate carrying the pinned key and an arbitrary CN passes; with `tls-full` the PROXY header then writes that attacker-chosen CN and the DER with the verified flags set, and the connection log records the forged subject. Entry and boundary: E1 a full handshake with one pinned key, B3. Configuration and default: needs `--allow-spki-pin` and `--proxy-protocol-mode=tls-full`, and a backend that authorizes or audits by the header CN. Rests on: S5; LB8.

**7. The backend records a verified client for a connection the ACL never saw.** Statement: on a resumed connection the peer certificate is restored from the ticket, and the tls-full PROXY header sets both certificate flags and verify=0 without distinguishing a resumed handshake from a full one, so a currently-denied client is attested to the backend as verified. Entry and boundary: E2, B3. Configuration and default: needs `--proxy-protocol-mode=tls-full` and a resumable ticket (SPIFFE, or the failed-reload case of risk 1). Rests on: S1, C10; LB1, LB8.

**8. A failed reload is reported upward as success, so the resumption window is invisible.** Statement: the reload stamps its timestamp before the outcome, logs "complete" on failure, re-sends systemd READY=1 while the process is not stopping, and the status response has no reload-error field, so an operator cannot tell that an intended CA removal or policy tightening did not happen; the old ticket keys then keep resumption serving the old decision. Entry and boundary: E3 the reload, B2. Configuration and default: holds at the default. Rests on: R1; LB3, LB9.

**9. A write to a cert, bundle or policy file plus a reload widens access with no check of what it now admits.** Statement: reload trusts whatever parses, with no validity-window check, no leaf-chains-to-bundle check, and no check of what roots a new bundle adds or what a new policy admits; a rollback is indistinguishable from a rotation to every observer, and the status reports ok. Entry and boundary: E3, B2 who can write the file, B3 payoff. Configuration and default: holds at the default once the write is possible. Rests on: T1, T2; LB9.

**10. One unauthenticated POST stops the proxy.** Statement: with `--enable-shutdown` the `/_shutdown` handler checks only the HTTP method, then stops the process; a clean drain exits 0, so a restart-on-failure supervisor treats the forced stop as intentional. Entry and boundary: E5, B3. Configuration and default: needs `--enable-shutdown` and a reachable status port. Rests on: D1.

**11. The process serves through every fault it detects, and its health signal certifies nothing.** Statement: a TLS or OPA reload error keeps serving on old material; a landlock setup error or one bad path serves unsandboxed; a blocked reload runs inline in the single signal goroutine and leaves shutdown and `/_shutdown` unserviced; the accept loop backs off and continues forever; and the systemd watchdog health predicate is a constant true, so a process serving no client keeps systemd satisfied. Entry and boundary: E3 and E6, B2, with a fault as the cause for the accept-loop and wedge cases. Configuration and default: holds at the default. Rests on: D2, D3, D4, R2, R4; LB10.

**12. Single-flag or single-action removal of an observer or its evidence, and the credentials it logs.** Statement: `--disable-authentication` forwards every client; `--quiet` silences the one report channel; `--disable-landlock` or any PKCS#11 flag drops the sandbox; the client proxy URL is logged with its password; and no CRL/OCSP revocation observer exists, so a revoked-but-unexpired certificate passes until expiry or an operator removal. Entry and boundary: E6, B2, except the revocation gap which is E1, B3. Configuration and default: each holds at the default once the named flag is set; the revocation gap holds at every configuration and is a delegation to operator CA hygiene. Rests on: S2, S3, T3, T4, S4, I5, E5.

## What the evidence shows and does not

What is source-supported is a line in the ghostunnel tree, opened and read; every such claim carries a repository-relative path and a line or range. What rests on the Go toolchain (crypto/tls, net/http/pprof, net/url) is marked as toolchain source and cited to Go 1.27.1. What rests on a vendored dependency (go-spiffe, OPA, certmagic, go-systemd, go-proxyproto) is marked vendor-unconfirmed and is not asserted. Anything resting on the Linux kernel (landlock enforcement, SO_REUSEPORT semantics) or on the operator's deployment is inferred and named at the point of use. Availability and performance claims are listed and left unevaluated, not conceded.

Total coverage is not claimed. The model covers ghostunnel's own code, about 12,411 non-test lines of Go across `auth/`, `policy/`, `wildcard/`, `certloader/`, `certstore/`, `proxy/`, `socket/`, and the root package. The verification of the resumption behavior reached into the Go 1.27.1 toolchain (`crypto/tls/common.go`, `handshake_server.go`, `handshake_server_tls13.go`) and the pprof behavior into `net/http/pprof/pprof.go`.

What was not attempted: ghostunnel itself was not built or run, and no live instance was contacted. One behavior, the resumption skip, was reproduced at runtime with a standalone stdlib-only Go 1.27.1 program (two client handshakes over a loopback listener sharing a session cache, with a verification callback that flips to deny between them); the ghostunnel binary was not exercised, and the toolchain source is authoritative for the mechanism. No exploit content appears anywhere: each finding names which check fails to observe what. Instructions found in the tree's files or comments were treated as data.

## How fast, and by what process

The exercise ran on 2026-09-24.

The process ran the six model passes, each an independent agent with no memory of the others, reading only the templates, the earlier pass outputs it was told to build on, and the source: a ring map of the check, gate, halt, loop and callback sites; a gap evaluation of that map against the eight properties; an independent verification at the cited line; a search for chains over the verified gaps; an independent chain verification; and a final sweep of the assembled document. Each verification opened every citation it relied on and took no reading from the pass before it.

Verification changed the document in small ways and reversed nothing structural. Pass 3 confirmed 21 rows and corrected 2 at the citation level, dropping none. Pass 5 confirmed 12 chains as source-supported and moved 2 to deployment-assumption for resting on an out-of-tree link, upheld all 11 rejected candidates, confirmed 9 new findings from the missed-observer sweep, and fed 2 citation corrections back into the verified file (the ticket-key rotation holds for every source except SPIFFE, not only file and ACME; the clean shutdown exits 0, not force-exits). The final sweep confirmed 28 of 29 rows and all 14 chains against source and fixed 3 citation or count details, removing no finding.

## The pattern behind the findings

ghostunnel decides once and continues on fault. The allow decision is taken at connection entry, for a full handshake, and lives nowhere after that: not over the connection's life, not on a TLS resumption, and not after a reload changes the policy or the trust store. A resumed session restores the peer's identity from the ticket and reaches the backend with no ACL evaluation (risk 1); an open connection is never re-checked (risk 2); the tls-full header attests that identity to the backend without recording that no check ran (risk 7).

The process observes itself, but its observers report to sinks, not to each other, and none can halt on a detected fault. A failed reload logs and keeps the old material while telling systemd it is ready (risk 8); a landlock failure logs and runs unconfined (risk 11); the watchdog certifies a constant (risk 11). The one upward liveness signal is decoupled from the function it is supposed to certify.

The status port is a second, quieter surface where the same decide-nothing posture leaks state: it authenticates no caller, restricts to no interface, and in client mode lends the process's own identity to whoever asks (risks 3 and 4).

## How to read this document

The order is: the risks, what the evidence shows and does not, how fast and by what process, the pattern, this section, the load-bearing claims and the posture matrix, the model in brief, Part A, and Part B, with the chains, the impact ranking, the seams not followed and the unsettled items in the closing parts.

**Citations.** A bare path and line, for example `main.go:1093` or `auth.go:207-265`, is a line in the ghostunnel tree. A path under `crypto/tls/`, `net/http/pprof/` or `net/url/` is Go 1.27.1 toolchain source. Comments and log strings are claims, not behavior.

**Source-supported against inferred.** A step marked source-supported is a line in the ghostunnel tree. A step that rests on the toolchain is marked and cited to Go 1.27.1. A step that rests on a vendored library, the kernel, or the operator's deployment is inferred and keeps that marking wherever it appears.

**Deployment assumption.** A row or chain is marked a deployment assumption when it needs a flag away from the shipped default, a file write, or a link the tree alone cannot produce (a process compromise, resolver influence, a header-trusting backend). Every row and chain keeps the flags and the deployment it needs, with the default. Availability and performance claims stay listed and unevaluated, not conceded.

## Load-bearing claims

Each fact two or more rows or chains rest on, stated once with its citation, the configuration it holds under with its default, whether it is source-supported or inferred, and the findings that rest on it.

| Id | Claim | Citations | Configuration and default | Standing | Rests on it |
|---|---|---|---|---|---|
| LB1 | The server ACL runs only as `VerifyPeerCertificate`, which is not invoked on a resumed connection; ghostunnel sets neither `VerifyConnection` nor `SessionTicketsDisabled` on the tunnel listener | `auth.go:207-265`; `main.go:915-924`; `tls.go:226-236`; `certtlsconfig.go:105-114`; `crypto/tls/common.go:675-678`; `handshake_server.go:558-589, :1019-1020`; `handshake_server_tls13.go:425-428, :821-822` | shipped default (tickets on) | source-supported; resumption skip toolchain-cited and runtime-reproduced | risks 1, 7; E1, S1, C1, C10 |
| LB2 | Ticket lifetime is 7 days, and a successful reload on any source except SPIFFE rotates ticket keys and closes resumption on the next accept, so the window stays open only on a failed reload or on SPIFFE | `crypto/tls/common.go:966, :998, :1049, :1123-1146`; `keystore.go:99-100`; `pkcs11_enabled.go:118`; `certstore_enabled.go:193`; `no_cert.go:51`; `acmetlsconfig.go:202`; `spiffe_tls_config.go:58-61` | shipped default | source-supported | risk 1; E1, E2, C1, C2 |
| LB3 | The reload keeps the old material on any error and returns before the atomic store | `keystore.go:73-97`; `policy/loader.go:72-74`; `signals.go:117-126` | shipped default | source-supported | risks 1, 8, 11; R1, D2, C1 |
| LB4 | The status server does no client authentication and is not restricted to a local interface; `--status` receives none of the `consideredSafe` gate `--target` and `--listen` get | `main.go:1093`; `main.go:292-306`; `main.go:401, :537` | shipped default | source-supported | risks 3, 4; I1, I2, I3, I4 |
| LB5 | The pprof handlers ride the same mux as `/_status`, and `/debug/pprof/cmdline` returns `os.Args`; `--storepass` and `--pkcs11-pin` are flags with an env-var alternative | `main.go:1043, :1065-1069`; `net/http/pprof/pprof.go:110-114`; `main.go:128, :183, :262` | needs `--enable-pprof` (off by default) and a secret passed as a flag | source-supported | risk 3; I2, C3 |
| LB6 | On a resumed connection only `NotAfter` and, under `RequireAndVerifyClientCert`, the stored chain against the current `ClientCAs` are re-checked; under `RequireAnyClientCert` (pin, SPIFFE) only `NotAfter` | `handshake_server_tls13.go:350, :370-380`; `handshake_server.go:480, :516, :524`; `crypto/tls/common.go:354-376` | shipped default | source-supported, toolchain | risk 1; E1, E2 |
| LB7 | After a connection is fused nothing re-checks the peer; the only bound is `--max-conn-lifetime`, default 0 (infinite) | `proxy.go:561-589`; `main.go:145`; `doc.go:253-254` | shipped default | source-supported | risks 2, 5; E3 |
| LB8 | In pin mode `ClientAuth` is `RequireAnyClientCert`, so no chain is validated, and the ACL compares only the SPKI; the tls-full header writes the certificate's own CN and DER with the verified flags set and verify=0 | `main.go:918-922`; `crypto/tls/common.go:362-365`; `auth.go:181-201`; `proxy.go:273-300`; `proxy/str.go:46` | needs `--allow-spki-pin` and `--proxy-protocol-mode=tls-full` | source-supported | risks 6, 7; S5, S1, C8, C10 |
| LB9 | The reload stamps its timestamp before the outcome, re-sends READY=1 while not stopping, logs "complete" on failure, and the status response has no reload-error field; the reload trusts whatever parses | `status.go:61-75, :99-133`; `signals.go:115-127`; `status_linux.go:46-55`; `keystore.go:69-103` | shipped default | source-supported | risks 8, 9; R1, T1, T2, C1, C9 |
| LB10 | Detected faults do not halt: reload errors continue, landlock errors continue, the accept loop continues forever, and the watchdog predicate is a constant true | `signals.go:117-126`; `main.go:692-699`; `landlock_linux.go:225-234, :253-267`; `proxy.go:403-446`; `status.go:156`; `status_linux.go:83-90` | shipped default | source-supported | risk 11; D2, D3, D4, R2, R4 |

## Posture matrix

Every configuration value a finding needs, with its default, and the findings that need it.

| Value | Default | Findings that need it |
|---|---|---|
| session tickets on the tunnel | on (Go default; no `SessionTicketsDisabled`, tls.go:226-236) | risk 1; E1, S1, C1, C10 |
| `--max-conn-lifetime` | `0s`, infinite (main.go:145) | risk 2; E3 |
| `--enable-pprof` | off (main.go:156) | risk 3; I2, C3 |
| secret as flag vs env (`--storepass`/`KEYSTORE_PASS`, `--pkcs11-pin`/`PKCS11_PIN`) | flag accepted (main.go:128, :183) | risk 3; I2 |
| `--status` interface | any interface, no local gate (main.go:292-306) | risks 3, 4; I1-I4 |
| `--unsafe-listen` | off (client `--listen` localhost/unix by default) | risk 5; E4 network case |
| `--allow-spki-pin` | off (main.go:89) | risks 6, 7; S5 |
| `--proxy-protocol-mode=tls-full` | off (proxy.go:273-300) | risks 6, 7; S1, S5, C8, C10 |
| `--enable-shutdown` | false (main.go:157) | risk 10; D1 |
| `--disable-authentication` | off (main.go:92) | risk 12; S2 |
| `--quiet` | off (main.go:240-243) | risk 12; T4 |
| `--disable-landlock` / any PKCS#11 flag | off (main.go:191, :683-688) | risks 11, 12; T3, D3 |
| `--proxy user:pass@host` | unset (main.go:1191) | risk 12; I5 |
| source type (file/PKCS#11/keychain/no-cert/ACME vs SPIFFE) | file typical; SPIFFE never rotates ticket keys on reload | risk 1; E1, E2, C1, C2 |

## Model in brief

The self-observing-systems model scores a system that carries customer data against one structure: the observation functions must form a ring of at least three mutually-observing nodes plus a coordinator that surface-checks them and can halt, with any detected fault halting the work and propagating to at least two nodes. Customer data sets the floor: any subsystem that carries customer information runs at the Catastrophic tier and requires the full ring. ghostunnel carries client identities and proxied traffic on every connection, so it sits at the Catastrophic tier.

## Part A: structure

**Observation graph.** Thirteen observers were found (O1 server ACL, O2 chain verification, O3 client-mode ACL, O4 handshake gate, O5 ACME gate, O6 dial/fuse lifecycle, O7 reload, O8 per-accept config refresh, O9 status endpoint, O10 systemd watchdog, O11 landlock, O12 signal/shutdown, O13 startup validation). Every check edge points at an external object (a peer certificate, a file, a socket, a flag) and every report edge points at a passive sink (a log line, a counter, an HTTP response, systemd). No observer watches another observer. The graph is one per-connection chain, accept then O4 wrapping O2 then O1, then the ACME gate, dial, and fuse (`proxy.go:451-520`), plus five process-level spokes (reload, status, watchdog, landlock, shutdown).

**Coordinator test.** No candidate that can reject or halt forms a coordinator. O1, O4, O7, O9, O12 and O13 each fail the requirement to check every other observer including itself and to take reports and validation back. O9, the status server, is the nearest breadth component; it reads its own flag and the backend socket (`status.go:159-205`), takes no validation back, and cannot halt. A hub that takes no validation back is not a coordinator.

**What they form.** No ring at layer 0, so no coordinator layer and no super-ring. The structure does not close. The five missing structural components are all absent: no mutual-observation cycle, no coordinator, no check/report separation, no process-level halt a detected fault can reach, and no upward chain that carries observer health.

**Eight-property verdict.** At the Catastrophic tier the scorecard is six gaps to two satisfied. Heartbeat, System-Wide Response, Self-Defense and Death Signaling are structural gaps: a single process has no observer watching another, and the one upward signal is defeated by the failed-reload READY=1 re-send (R1) and the constant-true watchdog (R2). Continuous State Verification is a gap because the allow decision is never re-taken over a connection's life or on resumption. Reflexive Response is a gap because detected faults continue rather than halt. Substrate Dependency and Automatic Integration are the two satisfied, trivially, because the guards share the process and every accept enters the same path; both carry no weight against the ring verdict.

## Part B: findings (STRIDE)

Actor: B2 operator (flags, on-disk files, the reload trigger, systemd, same-uid processes); B3 external (a TLS client controlling its own certificate and connection, or any network client reaching the status port). Entry: E1 client full handshake; E2 established/resumed connection; E3 reload trigger and files; E4 status HTTP; E5 `/_shutdown`; E6 operator config; E7 client-mode plaintext listener.

### S — Spoofing
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| S1 | B3 | E2 | tls-full header sets the verified flags for a resumed connection that ran no ACL | Backend receives a verified attestation for an unchecked connection | proxy.go:283-300; handshake_server_tls13.go:425 | none |
| S2 | B2→B3 | E6,E1 | `--disable-authentication` forwards every client | Any client reaches the backend | main.go:92, :915-916 | none |
| S3 | B2 | E6 | Empty client-mode ACL fails open; pin mode sets InsecureSkipVerify | Any server cert valid for the name is accepted | auth.go:277-285; main.go:1180-1183 | none by design |
| S4 | B2 (same uid) | E6 | Tunnel listener SO_REUSEPORT; a same-uid process may take a share of connections | Connections delivered outside every observer | socket/net.go:93, :120 | none; steal semantics kernel-unconfirmed |
| S5 | B3 (pinned key) | E1,E6 | Pin mode skips chain validation, ACL checks only SPKI, header carries attacker CN/DER | One pinned principal asserts any subject to a header-trusting backend | main.go:918-922; auth.go:181-201; proxy.go:273-300 | none |
| S6 | resolver/host actor (out of tree) | E6 | `--unsafe-target` re-resolved per dial, port-only landlock, no peer check | Client traffic and identities to an impostor backend | socket/net.go:56-59; main.go:1135-1138; landlock_linux.go:281-292 | none (deployment-assumption) |

### T — Tampering
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| T1 | B2→B3 | E3,E1 | Reload trusts any parseable bundle; no validity or chains-to-bundle check | An added root makes every cert under it pass | keystore.go:69-103; acmetlsconfig.go:194-204 | continue |
| T2 | B2→B3 | E3,E1 | OPA policy swapped by pointer, no observer of what it admits | Access widened silently | policy/loader.go:52-78 | continue |
| T3 | B2 | E6 | `--disable-landlock` or any PKCS#11 flag removes the sandbox | Process runs unconfined | main.go:191, :683-688 | none |
| T4 | B2 | E6 | `--quiet` clears the one report channel | Detected faults leave no record | main.go:240-243, :1230-1232 | none |
| T5 | B3 | E1 | PROXY AUTHORITY value is the client SNI, forwarded verbatim | A backend routing on it trusts a client string | proxy.go:243-249 | none |

### R — Repudiation
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| R1 | B2 | E3 | Failed reload reported as success: timestamp before outcome, READY=1 re-sent, no error field | Operator cannot tell the reload failed; masks risk 1 | status.go:61-75, :129; signals.go:115-127 | continue |
| R2 | B2 | E6 | Watchdog health predicate is a constant true | The one upward liveness signal carries no fault | status.go:149-157; status_linux.go:73-95 | none |
| R3 | B3 | E1 | A rejected client yields one log line and a counter, no boundary crossing, no rate | The ACL is probed at will with no acted-on record | proxy.go:474-477 | close one connection |
| R4 | fault | — | Accept errors continue forever; status checks the backend, not the listener | A process serving zero clients reports healthy | proxy.go:403-446; status.go:173-181 | continue |

### I — Information Disclosure
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| I1 | B3 | E4 | `/_status` unauthenticated, any interface, returns listen/forward addresses and hostname | Backend topology to any reachable client | status.go:61-75; main.go:292-306, :1093 | none |
| I2 | B3 | E4 | `/debug/pprof/cmdline` serves os.Args, exposing `--storepass`/`--pkcs11-pin` | Keystore password or HSM PIN to an unauthenticated caller | main.go:1064-1069, :1093; net/http/pprof/pprof.go:110-114 | none |
| I3 | B3 | E4 | Client-mode GET runs an outbound mTLS handshake with ghostunnel's cert and returns the result | Identity-use oracle and amplifier | main.go:836-851; status.go:241-245; dialer.go:44-66 | none |
| I4 | B3 | E4 | `/_metrics` counters unauthenticated | Traffic shape and error rates | main.go:1044-1058, :1093 | none |
| I5 | B2, B3 via I2 | E6 | Client proxy URL logged via `url.String()`, password included | Proxy credential disclosure | main.go:1191; net/url/url.go:823-833 | none |

### D — Denial of Service
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| D1 | B3 | E5 | One unauthenticated POST to `/_shutdown`, only the method checked; clean drain exits 0 | Any reachable client stops the proxy; reads as intentional to a supervisor | main.go:1020-1036; signals.go:43-74 | halts on command, not on fault |
| D2 | B2 | E3 | Reload error logs and keeps serving on old material | Serving through a detected fault | signals.go:117-126 | continue |
| D3 | B2 | E6 | Landlock setup error or one bad path serves unsandboxed | Serving through a detected containment failure | main.go:692-699; landlock_linux.go:225-267 | continue |
| D4 | B2 | E3 | Inline reload in the single signal goroutine wedges SIGTERM and `/_shutdown` | Graceful halt unreachable while health reports ok | signals.go:40, :76-103; main.go:1032-1035 | none |

### E — Elevation of Privilege
| # | Actor | Entry | Threat | Impact | Evidence | Halt-on-fault |
|---|---|---|---|---|---|---|
| E1 | B3 | E2 | Resumed session skips the ACL; only NotAfter/stored-chain re-checked; window up to 7 days, indefinite after a failed reload | A denied client keeps backend access with no evaluation and no record | tls.go:226-236; main.go:915-924; common.go:675-678; handshake_server_tls13.go:425-428 | none |
| E2 | B3 | E2 | SPIFFE source: config cached, Reload a no-op, RequireAnyClientCert, verifier only in VerifyPeerCertificate | A withdrawn workload resumes up to 7 days, bounded by SVID NotAfter | spiffe_tls_config.go:58-61, :143-168 | none; go-spiffe vendor-unconfirmed |
| E3 | B3 | E2 | After fuse nothing re-checks the peer; default lifetime infinite | A revoked client holds access as long as the socket stays open | proxy.go:561-589; main.go:145; doc.go:253-254 | none |
| E4 | local process (any uid), or B3 with `--unsafe-listen` | E7 | Client-mode plaintext listener unwrapped, no peer-credential check, forceHandshake a no-op | The caller uses ghostunnel's identity with a payload channel | main.go:975-992; socket/net.go:112-118; proxy.go:543-557 | none |
| E5 | B3 (revoked-but-unexpired cert) | E1 | No CRL/OCSP revocation observer; ACL checks names and policy only | CA revocation invisible until expiry or operator removal | grep (none in tree); auth.go:207-265 | none |

Row totals: S 6, T 5, R 4, I 5, D 4, E 5; total 29.

## Chains

Source-supported, then two whose realization needs an out-of-tree link.

1. C1: a failed TLS reload keeps the old pool and ticket keys while OPA reload succeeds and status reports ok, so a policy-denied client keeps backend access on every resumption, up to 7 days or indefinitely, invisibly. (R1 + D2 + E1 + T4.)
2. C2: an open or SPIFFE-resumed connection outlives a policy revocation. (E3 + E2.)
3. C11: the client-mode plaintext listener lends ghostunnel's identity to any local caller, with a payload channel. (E4 + E3 + I1/I3.)
4. C8: pin mode plus tls-full lets one pinned principal assert any subject to a header-trusting backend. (S5 + S1.)
5. C3: `/debug/pprof/cmdline` on a reachable status port discloses the keystore password or HSM PIN. (I2 + I1 + T3.)
6. C4: the any-interface status port is an identity-use oracle in client mode. (I3 + I1.)
7. C9: a bundle or policy rollback through a timed reload is indistinguishable from rotation to every observer. (T1/T2 + R1 + D2.)
8. C10: a resumed session plus the tls-full header records verified at the backend for a currently-denied client. (E1 + S1.)
9. C12: a blocked reload wedges the shutdown path while the watchdog beats. (R2 + D4.)
10. C6: a constant watchdog plus a broken accept loop reports healthy while serving nothing. (R2 + R4.)
11. C7: an unauthenticated `/_shutdown` stops the proxy and the clean exit reads as intentional. (D1 + I1.)
12. C14: the client proxy URL credentials reach the log and argv. (I5 + I2.)

Deployment-assumption (in-tree edge confirmed, realization out of tree):
- C5: an absent or reduced sandbox widens any later process compromise into persistence. (D3 + T3 + T1/T2 + R1.)
- C13: a name-resolved backend with port-only sandbox rules can be pointed at an impostor by a resolver-side or host-level actor. (S6.)

## Impact ranking

Lowest precondition first: E1 (resumed session skips the ACL, proven) > E3 (open connection outlives revocation) > I2 (password/PIN over pprof) > I1/I3 (status topology and identity oracle) > E4 (plaintext listener lends identity) > S5 (pinned client asserts any subject) > S1 (verified attestation for a resumed connection) > R1 (failed reload masked) > T1/T2 (reload widens access) > D1 (unauthenticated shutdown) > D2/D3/D4/R2/R4 (continue-on-fault and health that certifies nothing) > S2/S3/T3/T4/S4/I5/E5 (single-action observer or evidence removal, and the absent revocation observer).

## Seams not followed

ghostunnel delegates outward, and this model does not follow into these: the OPA policy author (that the policy admits only the intended principals and does not widen on reload); go-spiffe (SVID verification and lifetime, the only bound on E2); certmagic and the ACME CA (obtaining and renewing the leaf; `CanServe` is a constant true and no `NotAfter` self-check exists, acmetlsconfig.go:210-215); the Go crypto/tls stack (chain validation, the resumption checks, ticket-key rotation and lifetime); systemd (acting on a READY that follows a failed RELOADING); the Linux kernel (landlock enforcement, SO_REUSEPORT); and the operator (the max connection lifetime, the status-port interface, the `--enable-*`, `--unsafe-*` and `--disable-*` flags, secret handling, file permissions, and restarting to change the allow list). Integration surface: about twelve thousand lines of ghostunnel's own Go; the consumers are every deployment that fronts a backend with it.

## Unsettled items

Left out of tree or vendor-unconfirmed: the final step of C3 (keystore-file or HSM reach to turn a disclosed secret into the key), C5 (a process compromise), C13 (resolver influence or a host-level squat), the supervisor's `Restart=` semantics behind D1's clean exit, the kernel's SO_REUSEPORT same-uid semantics behind S4, go-spiffe's subject handling behind E2, certmagic's leaf rotation, the Prometheus default registry contents, and the pprof heap/goroutine profile contents. Availability and performance claims across the reload, the connection lifetime, the status-port backend dial, and `/_shutdown` are listed and left unevaluated.

## Verdict within the model

ghostunnel carries customer identities and customer traffic on every connection, so it sits at the Catastrophic tier and is required to form a 3+1 ring. It forms none: thirteen observers, none observing another, no coordinator, no upward chain that carries observer health, and no process-level halt a detected fault can reach. Observation gates execution once, for a full handshake, at connection entry, and only there; over a connection's life and on every resumption the allow decision is taken once at the door and never re-taken. All 29 findings are source-supported; the resumption-skips-the-ACL behavior is additionally confirmed at runtime with Go 1.27.1. Unsafe within the model at the required tier.
