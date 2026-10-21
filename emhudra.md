```markdown
Audit this repository for exposure to the eMudhra intermediate CA key compromise (disclosed 20 October 2026). Do not modify anything. Report only.

## Background
- The private keys of 8 eMudhra intermediate CAs are compromised:
  - TLS: EM DV TLS CA - G3B/G3C, EM OV TLS CA - G3B/G3C, EM EV TLS CA - G3B/G3C
  - Code signing: EM OV CS CA - G3B/G3C
- Each intermediate exists twice with the same public key: once issued by eMudhra, once cross-signed by Sectigo "AAA Certificate Services". That gives 16 certificates and 8 distinct keys. Match on SPKI SHA-256, not certificate hash.
- Cause: the CA generated the keys incorrectly, so anyone can derive them from the public certificates. This has been exploitable since the intermediates were issued in April 2025.
- Browsers were protected throughout because they enforce Certificate Transparency and no certificates were logged. Non-browser TLS clients that don't enforce CT or check revocation remain exposed until their runtime or root bundle is updated.
- Certificate data (fetch both first):
  - https://emudhra.we-have-a-problem.com/impacted-intermediates.json (16 certificates; use the spki_sha256 field)
  - https://emudhra.we-have-a-problem.com/roots.json (4 roots: emSign Root TLS CA - G3, emSign Root CS CA - G3, AAA Certificate Services, emSign ECC Root CA - G3)

## Fixed versions
Source: https://emudhra.we-have-a-problem.com/#developers. Re-check it, because the table is still being updated.
- Go 1.27.3 / 1.26.10 (binaries must be rebuilt; Go <= 1.25 is unsupported)
- Node.js 22.23.4 / 24.22.1 / 26.12.1
- Rust webpki-roots 1.0.10; rustls-webpki 0.103.16
- Python certifi 2026.10.19; pip 26.2.2; uv 0.13.4
- composer/ca-bundle 1.4.16 (1.4 line) or 1.6.0. There is no 1.5.x fix; 1.5.x users need 1.6.0.
- Electron 44.9.1 / 43.8.1 / 42.11.16
- NSS 3.131.1
- Debian ca-certificates 20250419~deb12u2 (bookworm) / 20250419+deb13u1 (trixie)
- Ubuntu ca-certificates 20260816ubuntu0.<release>.1 (22.04, 24.04, 26.04, 26.10)
- Alpine ca-certificates 20261020-r0
- Homebrew ca-certificates 2026-09-25_1
- Java, Bun: pending

## Checks
1. Inventory every outbound TLS client: language, HTTP library, and where it gets its trust store (bundled package, OS store, custom PEM file, environment variable).
2. Rust:
   - Check the webpki-roots and rustls-webpki versions in Cargo.lock.
   - Determine whether rustls is configured with webpki-roots, rustls-native-certs, or rustls-platform-verifier, including via reqwest, hyper-rustls, or tonic feature flags.
3. PHP:
   - Check the composer/ca-bundle version in composer.lock.
   - Determine whether Guzzle and curl use the system bundle or a bundled cacert.pem.
   - Find any custom `verify` paths and any curl.cainfo or openssl.cafile settings.
   - Note that the system bundle usually makes the container's ca-certificates package the deciding factor.
4. Node/TypeScript:
   - Find the Node version in .nvmrc, .node-version, package.json engines, Dockerfiles and CI.
   - Note any use of NODE_EXTRA_CA_CERTS, --use-system-ca, --use-openssl-ca, or custom `ca` options.
   - Check for Electron and Bun.
5. Python:
   - Check the certifi version in all lockfiles, and the pip and uv versions used in CI and images.
   - Note any REQUESTS_CA_BUNDLE, SSL_CERT_FILE, or custom verify paths.
6. Go: check the `go` and `toolchain` directives in go.mod, builder image tags, and CI setup-go versions.
7. Java: identify any JVM workloads and their truststore source. No fix has been released yet, so flag these as exposed.
8. Containers:
   - List every base image in Dockerfiles, compose files, CI, and Helm/k8s manifests.
   - Identify the distro and release, and whether the installed ca-certificates version meets the fixed version above.
   - Note digest pins that freeze an older bundle, and whether a rebuild is needed.
9. Committed certificate material:
   - Find *.pem, *.crt, *.cer, *.der, *.p7b, *.p12, *.pfx, *.jks, cacert* and ca-bundle* files, plus inline "BEGIN CERTIFICATE" blocks.
   - Decode each with openssl and compute the SPKI SHA-256: `openssl x509 -in FILE -noout -pubkey | openssl pkey -pubin -outform der | openssl dgst -sha256`
   - Compare the results against both JSON files.
10. Disabled verification: flag verify=False, InsecureSkipVerify, CURLOPT_SSL_VERIFYPEER false, rejectUnauthorized: false, NODE_TLS_REJECT_UNAUTHORIZED=0, custom ServerCertVerifier implementations, and danger_accept_invalid_certs. These are exposed regardless of this incident.
11. Breakage risk:
    - List outbound hostnames configured in code and config.
    - Where network access allows, check their chains with `openssl s_client -connect HOST:443 -showcerts`.
    - Flag any that chain to emSign ECC Root CA - G3 or AAA Certificate Services, because they will break once those roots are removed.
12. Code signing: find any pipeline that signs with, or verifies against, emSign code-signing certificates (EM OV CS CA - G3B/G3C, emSign Root CS CA - G3).
13. Live test:
    - Where you can build and run the container or runtime, request these URLs using the application's own HTTP client and trust configuration (not system curl):
      - https://dv-g3b-emudhra.we-have-a-problem.com/
      - https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
    - A certificate error is the pass condition. A successful response means the runtime is exposed.
    - Report the result per runtime.
14. Historical exposure: list the outbound integrations that have existed since April 2025 and carry sensitive data (credentials, tokens, health or personal data). These are candidates for a risk-register entry even after remediation.

## Output
1. A table with columns: Area | Finding | File:line | Status (exposed / fixed / not applicable / unknown) | Action
2. Live test results per runtime.
3. Required actions in priority order.
4. Anything you could not determine, and why.
```
