# eMudhra CA compromise: updating your local dev environment

Last updated: 23 October 2026

Your browser is already protected. This guide covers the command-line tools and runtimes on your machine, which use their own root bundles and don't check revocation.

This guide covers machine-level tooling only. Project dependencies (lockfiles, base images in repos) go through the repo audit, so don't commit fixes to them independently.

Work through the sections that apply to you, then run the checks in step 10.

Background and current patch list: https://emudhra.we-have-a-problem.com/#developers

## 1. Operating system

### macOS

1. Install all pending macOS updates (System Settings → General → Software Update).
2. Update Homebrew's bundle:
   ```bash
   brew update && brew upgrade ca-certificates openssl
   ```
3. Patch the system bundle. Apple's built-in `curl`, `git`, `python3`, `perl`, `ruby` and `openssl` read `/etc/ssl/cert.pem`, which Apple hasn't updated. This command backs the file up first:
   ```bash
   sudo cp -p /etc/ssl/cert.pem /etc/ssl/cert.pem.bak &&
   sudo awk '/^(###|===) /{skip=0}
     /^=== .*\/CN=(AAA Certificate Services|emSign ECC Root CA - G3)$/{skip=1}
     !skip' /etc/ssl/cert.pem.bak | sudo tee /etc/ssl/cert.pem > /dev/null
   ```
4. Only if you use Apple's built-in `python3` and `pip3` (not Homebrew, pyenv or uv): point pip at the patched file.
   ```bash
   F="$(/usr/bin/python3 -c 'import sysconfig; print(sysconfig.get_path("stdlib"))')/sitecustomize.py" &&
   echo 'import os; os.environ.setdefault("PIP_CERT", "/etc/ssl/cert.pem")' | sudo tee -a "$F" > /dev/null &&
   sudo chmod 644 "$F"
   ```

### Linux, including WSL

Update the `ca-certificates` package, then check the installed version against the table.

| Distro | Command | Fixed version |
|---|---|---|
| Debian / Ubuntu | `sudo apt update && sudo apt install --only-upgrade ca-certificates` | Debian 12: `20250419~deb12u2`<br>Debian 13: `20250419+deb13u1`<br>Debian testing/unstable: `20260816+nmu1`<br>Ubuntu: `20260816ubuntu0.<release>.1` |
| Fedora | `sudo dnf upgrade ca-certificates` | `2026.2.92_v9.0.319` |
| Arch | `sudo pacman -Syu ca-certificates-mozilla` | `3.131.1-1` |
| Alpine | `sudo apk upgrade ca-certificates` | `20261020-r0` |

To check the version, run `dpkg -l ca-certificates` (Debian/Ubuntu) or `rpm -q ca-certificates` (Fedora). If your distro hasn't shipped a fixed version, report it in the thread rather than applying a manual workaround.

### Windows

1. Run Windows Update. Microsoft pushes the block automatically to machines that receive trust list updates.
2. Git for Windows ships its own CA bundle. Switch it to the Windows certificate store, which already has the block:
   ```
   git config --global http.sslBackend schannel
   ```
3. If your machine is university-managed, don't run local certificate-store scripts. IT manages that.
4. Update Docker Desktop to 4.96.2 or later.

## 2. Node.js and Bun

Install the patched Node release for each major version you use, and make it your default.

| Major | Fixed version |
|---|---|
| 22 | 22.23.4 |
| 24 | 24.22.1 |
| 26 | 26.12.1 |

For example, with nvm:

```bash
nvm install 22.23.4 && nvm alias default 22.23.4
```

fnm, Volta and mise work the same way. If you use mise, update mise itself to 2026.10.17 or later.

Open a new shell and confirm that `node --version` reports the fixed version.

If you use Bun, run `bun upgrade` to get 1.4.4 or later.

## 3. Rust

```bash
rustup self update   # needs 1.29.2 or later
rustup update
```

Tools installed with `cargo install` were built against whatever webpki-roots they locked. Reinstall any that make network calls:

```bash
cargo install --force <tool>
```

## 4. Python

- **Interpreter:** upgrade to 3.15.1, 3.14.9 or 3.13.17, depending on your minor version. With pyenv, run `pyenv install 3.13.17` (or the version you need). With uv, run `uv python install 3.13.17`. Older minor versions (3.12 and below) have no listed fix, so rely on certifi.
- **uv:** `uv self update` (0.13.4 or later)
- **pip / certifi** in global, user or pipx environments you use outside projects: `python -m pip install -U pip certifi` (pip 26.2.2 or later, certifi 2026.10.19 or later)
- **pipx:** `pipx upgrade-all`
- **Conda:** `conda update -n base conda` (26.9.4 or later)
- Project virtualenvs are updated through the audit, not by hand.

## 5. Go

Install Go 1.27.3 or 1.26.10. Then rebuild any tools you installed with `go install`, because existing binaries keep the old behaviour until they're recompiled.

## 6. Java

Patched JDKs are now available (Oracle-style version numbers):

| Line | Fixed version |
|---|---|
| 27 | 27.0.1.0.2 |
| 25 | 25.0.5.0.2 |
| 21 | 21.0.13.0.2 |
| 17 | 17.0.21.0.2 |
| 11 | 11.0.33.0.2 |
| 8 | 8u511 |

Advisories: [OpenJDK](https://openjdk.org/groups/vulnerability/advisories/2026-10-21), [Oracle JDK](https://ops.java/emudhra)

1. **Find every JDK on your machine.** Include the runtimes bundled with IntelliJ and Android Studio, because Gradle and Maven may run on those.
   - macOS: `/usr/libexec/java_home -V`
   - SDKMAN: `sdk list java`
   - Linux: `update-alternatives --list java`
2. **Upgrade each one** to the fixed release for its line through your usual channel: SDKMAN, Homebrew, apt, your vendor's installer, or IDE updates for bundled runtimes. Vendor builds (Temurin, Corretto, Zulu) may number their releases differently. Use the step 10 test to confirm.
3. **Debian/Ubuntu (including WSL) with apt-installed JDKs:** these JDKs use the system Java keystore, which can fall out of sync with the system store. After updating `ca-certificates` in step 1, resync it:
   ```bash
   sudo touch /var/lib/ca-certificates-java/fresh
   sudo dpkg-reconfigure ca-certificates-java
   ```

**If you can't upgrade a JDK yet**, replace its blocked-certificate list with Oracle's updated one:

1. Download the file:
   - Windows JDKs: https://ops.java/windows/blocked.certs
   - All other JDKs, including JDKs inside WSL: https://ops.java/unix/blocked.certs

   Try `curl -fLO https://ops.java/unix/blocked.certs` first. If the site is under load and curl fails, download the file in your browser instead.

   **WSL:** a JDK inside WSL is a Linux JDK, so download the **unix** file even though you're using a Windows browser. You can't drag a file into the JDK directory because of permissions, so put it in your WSL home directory first, either by dragging it in Explorer or by running:
   ```bash
   cp /mnt/c/Users/<windows-username>/Downloads/blocked.certs ~/
   ```
2. In `[JDK_HOME]/lib/security`, run:
   ```bash
   sudo mv blocked.certs blocked.certs.BACKUP   # back up the existing file
   sudo cp ~/blocked.certs blocked.certs        # copy in the downloaded file
   ```
   If you had added your own entries to the old file, re-add them from the backup. On Java 8, the file is named `blacklisted.certs`, in the same directory.

Don't use Oracle's other option, turning on revocation checking with JVM flags, for local dev. It makes every TLS connection depend on reaching the CA's CRL server.

## 7. PHP

- Run `composer self-update`. Composer bundles its own CA fallback.
- **composer/ca-bundle's status is unclear.** The disclosure page lists 1.4.16 and 1.6.0 as pending, and the upstream PR is still open, but both versions exist on Packagist. Don't trust the version number. Check the bundled file in a project directly:
  ```bash
  grep -c "emSign ECC Root CA - G3" vendor/composer/ca-bundle/res/cacert.pem
  ```
  `0` means the root has been removed. Any other number means it's still trusted, so report it in the thread.
- If you use Laravel Herd, Valet or a Homebrew PHP, the system or Homebrew bundle from step 1 covers it.

## 8. Docker

Update Docker Desktop to 4.96.2 or later.

Your local images contain their own `ca-certificates`. Rebuild them with fresh base images:

```bash
docker compose build --pull --no-cache
```

Pulling `:latest` isn't enough if the Dockerfile pins a digest. The audit handles pinned digests.

## 9. Apps and tools

- Update Claude Code to 2.1.303 or later: `claude update`.
- Update VS Code, Cursor, Slack and other Electron apps. The fix is in Electron 44.9.1, 43.8.1 and 42.11.16.

## 10. Verify

The test hosts serve certificates from the compromised intermediates. A **certificate error means you're fixed**. A successful response means you're still exposed.

The `-noct-` hosts serve leaf certificates that were never logged to Certificate Transparency. A client that connects to these both trusts the compromised intermediates and doesn't enforce CT, which is the case an attacker can exploit without being noticed. Most command-line tools don't enforce CT, so for them the `-noct-` and plain hosts give the same result.

```bash
# curl / system bundle
for h in dv-g3b dv-g3b-aaa dv-g3b-noct dv-g3b-aaa-noct; do
  curl -sS "https://$h-emudhra.we-have-a-problem.com/" -o /dev/null 2>/dev/null \
    && echo "$h: EXPOSED" || echo "$h: OK"
done

# Node
node -e "fetch('https://dv-g3b-noct-emudhra.we-have-a-problem.com/').then(()=>console.log('EXPOSED'),()=>console.log('OK'))"

# Bun
bun -e "fetch('https://dv-g3b-noct-emudhra.we-have-a-problem.com/').then(()=>console.log('EXPOSED'),()=>console.log('OK'))"

# Python (certifi, as used by requests/httpx)
python3 -c "import ssl,certifi,urllib.request as u; u.urlopen('https://dv-g3b-noct-emudhra.we-have-a-problem.com/', context=ssl.create_default_context(cafile=certifi.where())); print('EXPOSED')" 2>/dev/null || echo "OK"

# Python (interpreter default context)
python3 -c "import urllib.request as u; u.urlopen('https://dv-g3b-noct-emudhra.we-have-a-problem.com/'); print('EXPOSED')" 2>/dev/null || echo "OK"

# git
git ls-remote https://dv-g3b-noct-emudhra.we-have-a-problem.com/ 2>&1 | grep -qi certificate && echo "OK" || echo "EXPOSED"
```

The certifi check needs certifi installed in whichever `python3` comes first on your PATH. If certifi is missing, the check prints "OK" incorrectly.

For Java (11 or later), run this once per JDK, using that JDK's `java` binary:

```bash
cat > /tmp/EmudhraCheck.java <<'EOF'
public class EmudhraCheck {
  public static void main(String[] a) throws Exception {
    try {
      java.net.URI.create("https://dv-g3b-noct-emudhra.we-have-a-problem.com/").toURL().openStream().close();
      System.out.println("EXPOSED");
    } catch (javax.net.ssl.SSLException e) {
      System.out.println("OK");
    }
  }
}
EOF
java /tmp/EmudhraCheck.java
```

If anything prints EXPOSED, post it in the thread with your OS and the tool version.
