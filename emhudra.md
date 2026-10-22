Audit this repository for exposure to the eMudhra intermediate CA key compromise (disclosed 20 October 2026; prompt updated 23 October 2026). Do not modify anything. Report only.

## Background
- The private keys of 8 eMudhra intermediate CAs are compromised:
  - TLS: EM DV TLS CA - G3B/G3C, EM OV TLS CA - G3B/G3C, EM EV TLS CA - G3B/G3C
  - Code signing: EM OV CS CA - G3B/G3C
- Each intermediate exists twice with the same public key: once issued by eMudhra and once cross-signed by Sectigo "AAA Certificate Services". That makes 16 certificates and 8 distinct keys. Match on SPKI SHA-256, not on the certificate hash.
- Cause: the CA generated the keys incorrectly, so anyone can derive them from the public certificates. This has been exploitable since the intermediates were issued in April 2025.
- Browsers were protected throughout because they enforce Certificate Transparency and no certificates were logged. Non-browser TLS clients that don't enforce CT or check revocation stay exposed until their runtime or root bundle is updated.
- Certificate data (fetch both first):
  - https://emudhra.we-have-a-problem.com/impacted-intermediates.json (16 certificates; use the spki_sha256 field)
  - https://emudhra.we-have-a-problem.com/roots.json (4 roots: emSign Root TLS CA - G3, emSign Root CS CA - G3, AAA Certificate Services, emSign ECC Root CA - G3)

## Fixed versions
Source: https://emudhra.we-have-a-problem.com/#developers. Re-check it, because the table is still being updated.
- Go: 1.27.3 / 1.26.10. Binaries must be rebuilt. Go <= 1.25 is unsupported.
- Node.js: 22.23.4 / 24.22.1 / 26.12.1
- Bun: 1.4.4
- Deno: 2.9.8
- Java: 27.0.1.0.2 / 25.0.5.0.2 / 21.0.13.0.2 / 17.0.21.0.2 / 11.0.33.0.2 / 8u511. These use Oracle numbering, and vendor builds (Temurin, Corretto, Zulu) may number differently.
- Python: 3.15.1 / 3.14.9 / 3.13.17. Also certifi 2026.10.19, pip 26.2.2, uv 0.13.4, conda 26.9.4, pixi 0.82.1.
- Rust: webpki-roots / webpki-root-certs 1.0.10; rustls-webpki 0.103.16; rustup 1.29.2
- composer/ca-bundle: UNCONFIRMED. Versions 1.4.16 and 1.6.0 exist on Packagist, but the disclosure page lists them as pending and the upstream PR (composer/ca-bundle#154) is unmerged. Don't trust the version number. Check the bundled cacert.pem directly (check 3).
- Electron: 44.9.1 / 43.8.1 / 42.11.16
- NSS: 3.131.1; curl CA bundle 2026-10-20
- Debian ca-certificates: 20250419~deb12u2 (bookworm) / 20250419+deb13u1 (trixie) / 20260816+nmu1 (testing/unstable)
- Ubuntu ca-certificates: 20260816ubuntu0.<release>.1 (22.04, 24.04, 26.04, 26.10)
- Alpine ca-certificates: 20261020-r0
- Fedora ca-certificates: 2026.2.92_v9.0.319
- Chainguard ca-certificates: 20260909-r3
- Docker Desktop: 4.96.2
- ArgoCD: 3.3.18
- pypa/gh-action-pypi-publish: 1.14.3
- Pending: Bottlerocket, Erlang certifi

## Checks
1. Inventory every outbound TLS client. For each, record the language, the HTTP library, and where it gets its trust store (bundled package, OS store, custom PEM file, or environment variable).
2. Rust:
   - Check the webpki-roots and rustls-webpki versions in Cargo.lock.
   - Determine whether rustls is configured with webpki-roots, rustls-native-certs, or rustls-platform-verifier, including through reqwest, hyper-rustls, or tonic feature flags.
3. PHP:
   - Record the composer/ca-bundle version in composer.lock.
   - If vendor/ is present, run `grep -c "emSign ECC Root CA - G3" vendor/composer/ca-bundle/res/cacert.pem`. A result of 0 means fixed; anything else means exposed.
   - If vendor/ is absent, report the status as unknown.
   - Determine whether Guzzle and curl use the system bundle or a bundled cacert.pem.
   - Find any custom `verify` paths and any curl.cainfo or openssl.cafile settings.
   - Note that the system bundle usually makes the container's ca-certificates package the deciding factor.
4. Node/TypeScript/Bun/Deno:
   - Find runtime versions in .nvmrc, .node-version, .bun-version, package.json engines and packageManager, Dockerfiles and CI.
   - Note any use of NODE_EXTRA_CA_CERTS, --use-system-ca, --use-openssl-ca, or custom `ca` options.
   - Check for Electron.
5. Python:
   - Find the interpreter version in .python-version, pyproject.toml requires-python, Dockerfiles and CI. Interpreter minor versions 3.12 and below have no listed fix, so for those, rely on certifi.
   - Find the certifi version in all lockfiles, and the pip, uv and conda versions used in CI and images.
   - Note any REQUESTS_CA_BUNDLE, SSL_CERT_FILE, or custom verify paths.
6. Go: check the `go` and `toolchain` directives in go.mod, the builder image tags, and the CI setup-go versions.
7. Java:
   - Identify any JVM workloads, their JDK version and vendor (from Dockerfiles, CI, .sdkmanrc, .java-version, toolchain configs in Gradle or Maven), and their truststore source.
   - Compare the version against the fixed versions above.
   - For Debian/Ubuntu-based images that use ca-certificates-java, check whether the image regenerates the Java keystore after updating ca-certificates.
   - Note any custom truststores (-Djavax.net.ssl.trustStore, bundled .jks files).
8. Containers:
   - List every base image in Dockerfiles, compose files, CI, and Helm/k8s manifests.
   - Identify the distro and release, and whether the installed ca-certificates version meets the fixed version above.
   - Note digest pins that freeze an older bundle, and whether a rebuild is needed.
9. Committed certificate material:
   - Find *.pem, *.crt, *.cer, *.der, *.p7b, *.p12, *.pfx, *.jks, cacert* and ca-bundle* files, and inline "BEGIN CERTIFICATE" blocks.
   - Decode each with openssl and compute the SPKI SHA-256: `openssl x509 -in FILE -noout -pubkey | openssl pkey -pubin -outform der | openssl dgst -sha256`
   - Compare the results against both JSON files.
10. Disabled verification: flag any of the following, which are exposed regardless of this incident:
    - verify=False
    - InsecureSkipVerify
    - CURLOPT_SSL_VERIFYPEER false
    - rejectUnauthorized: false
    - NODE_TLS_REJECT_UNAUTHORIZED=0
    - custom ServerCertVerifier implementations
    - danger_accept_invalid_certs
    - Java TrustManagers that accept all certificates
11. Breakage risk:
    - List the outbound hostnames configured in code and config.
    - Where network access allows, check their chains with `openssl s_client -connect HOST:443 -showcerts`.
    - Flag any that chain to emSign ECC Root CA - G3 or AAA Certificate Services, because they will break once those roots are removed.
12. Code signing: find any pipeline that signs with, or verifies against, emSign code-signing certificates (EM OV CS CA - G3B/G3C, emSign Root CS CA - G3).
13. CI and deployment tooling: check versions of ArgoCD, pypa/gh-action-pypi-publish, and any other tool from the fixed-versions list used in workflows or deployment config.
14. Live test:
    - Where you can build and run the container or runtime, request each of these URLs using the application's own HTTP client and trust configuration, not system curl:
      - https://dv-g3b-emudhra.we-have-a-problem.com/
      - https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
      - https://dv-g3b-noct-emudhra.we-have-a-problem.com/
      - https://dv-g3b-aaa-noct-emudhra.we-have-a-problem.com/
    - A certificate error is the pass condition. A successful response means the runtime is exposed.
    - The -noct- hosts serve leaf certificates that were never logged to CT. Success there means the client both trusts the intermediates and doesn't enforce CT, which is the case that can be exploited silently.
    - Report results per runtime and per host.
15. Historical exposure: list outbound integrations that have existed since April 2025 and carry sensitive data (credentials, tokens, health or personal data). These are candidates for a risk-register entry even after remediation.

## Output
1. A table with columns: Area | Finding | File:line | Status (exposed / fixed / not applicable / unknown) | Action
2. Live test results, per runtime and per host.
3. Required actions in priority order.
4. Anything you could not determine, and why.
