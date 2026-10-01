# Echo Music — Security & Engineering Audit Notes

[![Security Review](https://img.shields.io/badge/security-review-in%20progress-orange)](#-severity-legend)
[![Kotlin](https://img.shields.io/badge/Kotlin-Android-purple?logo=kotlin)](https://kotlinlang.org/)
[![Android](https://img.shields.io/badge/Platform-Android-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![GitHub](https://img.shields.io/badge/GitHub-Alitan999-181717?logo=github)](https://github.com/Alitan999)
[![Discord](https://img.shields.io/badge/Discord-alitan999-5865F2?logo=discord&logoColor=white)](https://discord.com/)

> **Static audit notes for Echo Music.**
>
> This document records potentially concerning findings and engineering improvement areas identified during a source-code review. It is **not** a claim that Echo Music is malware, nor is it a complete penetration test, SAST report, or independent security certification.

**Repository:** https://github.com/EchoMusicApp/Echo-Music  
**Auditor:** `Alitan999`  
**Discord:** `alitan999`  
**Contact:** `helloalitan.dev@gmail.com`

---

## 🚦 Severity legend

| Level | Meaning |
|---|---|
| 🔴 **Critical** | Immediate security concern, potential credential compromise, or issue requiring urgent remediation. |
| 🟠 **High** | Significant attack surface, privacy risk, security weakness, or architecture issue that should be investigated/remediated. |
| 🟡 **Medium** | Worth reviewing and improving; impact depends on implementation/context. |

> **Scope:** Only 🔴 🟠 🟡 findings are listed here. Informational/positive findings are intentionally omitted.

---

# 🔴 Critical Findings

## 🔴 1. Hardcoded Last.fm credentials

**Location:** `app/build.gradle.kts`

The build configuration contains Last.fm credential values that are exposed directly in source/build configuration and then passed into `BuildConfig`.

### Why this matters

If these values are valid credentials/secrets, anyone with access to the repository can potentially recover them. Once a secret has been committed to source control, simply deleting it from the latest revision is not sufficient: it may remain in Git history, forks, caches, CI logs, or previously published artifacts.

There is an additional concern if a secret is compiled into the Android application: values inside an APK should be treated as recoverable by an attacker.

### Recommended improvements

- Revoke/rotate the exposed Last.fm secret immediately.
- Remove secrets from source control and Git history where appropriate.
- Store CI-only secrets in GitHub Actions Secrets.
- Avoid putting true secrets in `BuildConfig`.
- If the provider requires a client-side API key, use provider-supported restrictions and assume it can be extracted from the APK.
- Audit old commits/releases for the same credentials.

### Verification checklist

- [ ] Rotate exposed credentials.
- [ ] Search complete Git history.
- [ ] Search published APK/AAB artifacts.
- [ ] Search CI logs/artifacts for accidental exposure.
- [ ] Confirm replacement credentials are not embedded as secrets.

---

# 🟠 High Findings

## 🟠 2. `REQUEST_INSTALL_PACKAGES` permission

**Location:** `app/src/main/AndroidManifest.xml`

The application declares:

```xml
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
```

### Why this matters

This permission gives an application access to Android's APK installation flow.

It does **not** prove malicious behavior by itself. However, for a music application, the exact reason for this permission should be documented and the complete call chain should be audited.

The project documentation also indicates that in-app OTA updates were removed, making the permission particularly worth investigating.

### Recommended improvements

- Identify every reference to `REQUEST_INSTALL_PACKAGES`.
- Identify every code path that downloads an APK.
- Verify the source and authenticity of downloaded packages.
- Ensure installation requires an explicit user action.
- Remove the permission if it is no longer required.
- Add a security test preventing arbitrary APK URLs from reaching the installer.

### Questions to answer

```text
Who supplies the APK URL?
Can the URL be controlled remotely?
Is HTTPS enforced?
Is the downloaded APK cryptographically verified?
Can an external Intent trigger the installation flow?
Can an untrusted server influence the package being installed?
```

---

## 🟠 3. Global cleartext HTTP allowed

**Location:** `network_security_config.xml`

The network security configuration allows cleartext traffic globally.

### Why this matters

The project explains that cleartext is needed for local Listen Together servers, while production uses WSS.

The problem is that a global cleartext allowance increases the chance that a future or accidental HTTP request can transmit data without transport encryption.

### Recommended improvements

Prefer a host-specific exception instead of globally enabling cleartext traffic.

For example, conceptually:

```xml
<domain-config cleartextTrafficPermitted="true">
    <domain includeSubdomains="true">allowed-local-host.example</domain>
</domain-config>
```

Everything else should remain HTTPS/WSS.

### Verification checklist

- [ ] Enumerate all HTTP endpoints.
- [ ] Enumerate all WebSocket endpoints.
- [ ] Confirm production endpoints use TLS.
- [ ] Confirm credentials/cookies never travel over HTTP.
- [ ] Restrict cleartext to the smallest possible scope.

---

## 🟠 4. Media playback authorization / exported service attack surface

**Location:** `AndroidManifest.xml` / `MusicService.kt`

The playback service is externally accessible as part of Android's media-session architecture.

The project's release history also documents a previous authorization-bypass issue involving playback controls.

### Why this matters

Media services legitimately need to communicate with Android system components and other media controllers. However, every externally reachable playback command becomes an authorization boundary.

### Recommended improvements

Audit every externally callable operation:

```text
play
pause
seek
skip
queue manipulation
like/favorite
download
custom commands
media-session callbacks
```

For each operation determine:

```text
Who can call it?
What authentication/authorization is performed?
Can another installed application invoke it?
Are package names trusted without cryptographic verification?
Are custom commands validated?
```

Add regression tests for unauthorized media-controller calls.

---

## 🟠 5. Exported widget receivers and custom Intents

**Location:** `AndroidManifest.xml`

Several widget/receiver components are exported to integrate with Android.

### Why this matters

Exported receivers can become an attack surface if custom actions perform privileged operations without validating their origin.

Potentially sensitive actions include:

```text
PLAY_PAUSE
LIKE
widget playback commands
recognition actions
```

### Recommended improvements

- Enumerate all exported receivers.
- Enumerate all accepted Intent actions.
- Check whether each action can be triggered by another application.
- Avoid trusting extras from untrusted callers.
- Use Android permissions where appropriate.
- Make receivers non-exported when external access is unnecessary.
- Add Intent-fuzzing/security tests.

---

## 🟠 6. OAuth callback surface

**Location:** `DiscordOAuthCallbackActivity`

The application uses an exported OAuth callback activity/custom URI scheme for Discord authentication.

### Why this matters

Custom URI schemes can be susceptible to interception by another application if the OAuth flow is not designed defensively.

### Recommended improvements

- Validate OAuth `state`.
- Use PKCE where supported.
- Validate redirect parameters.
- Never trust arbitrary callback extras.
- Minimize token lifetime.
- Avoid logging OAuth tokens/codes.
- Prefer Android App Links/verified HTTPS redirects when the provider supports them.

---

## 🟠 7. YouTube authentication/session material

**Location:** `innertube` / `InnerTube.kt`

The application handles YouTube Music authentication-related material including cookies and values used to construct authenticated requests.

### Why this matters

Session cookies and authentication material are highly sensitive. A bug in storage, logging, proxying, or request construction could expose a user's session.

This is **not evidence that Echo Music steals credentials**. It is a high-value area that requires careful review because of what the code legitimately has access to.

### Recommended improvements

- Never log authentication cookies.
- Never send them to non-YouTube hosts.
- Minimize persistence.
- Encrypt sensitive local storage where appropriate.
- Audit proxy configuration.
- Audit every network interceptor.
- Add host allowlists for authenticated requests.
- Test that redirects cannot leak authenticated headers/cookies.

---

## 🟠 8. Backup and sensitive local data

**Location:** Android backup configuration / Room / DataStore

The application enables Android backup behavior while explicitly excluding some playback/download data.

### Why this matters

A backup configuration should be reviewed against everything the application stores locally.

Potentially sensitive data may include:

```text
accounts
tokens
preferences
cookies
session identifiers
playlists
listening history
user configuration
```

### Recommended improvements

- Inventory every Room database.
- Inventory every DataStore.
- Inventory SharedPreferences and file storage.
- Identify tokens/session identifiers.
- Ensure sensitive authentication material is excluded from backup or securely stored.
- Test backup/restore on a clean device.

---

# 🟡 Medium Findings

## 🟡 9. Location permissions

**Location:** `AndroidManifest.xml`

The application requests coarse and fine location permissions.

### Why this matters

Location is sensitive personal data even when it is used for a legitimate feature.

The application should request it only when needed and clearly explain why.

### Recommended improvements

- Request location at the moment of use rather than startup.
- Prefer approximate location when precise location is unnecessary.
- Document exactly which feature uses location.
- Document whether location leaves the device.
- Avoid retaining historical location unless required.
- Add a privacy test covering location collection.

---

## 🟡 10. Microphone permission

**Location:** `AndroidManifest.xml`

The application requests:

```text
RECORD_AUDIO
FOREGROUND_SERVICE_MICROPHONE
```

This appears related to music recognition functionality.

### Recommended improvements

- Request microphone access only when recognition is explicitly activated.
- Stop microphone capture immediately after recognition.
- Provide clear UI indicating microphone use.
- Verify no audio is retained unnecessarily.
- Verify audio is not transmitted to unexpected endpoints.

---

## 🟡 11. Large `MusicService.kt`

**Location:** `MusicService.kt`

The playback service is very large and contains many responsibilities.

### Why this matters

Large services increase the likelihood of:

- lifecycle bugs;
- concurrency bugs;
- state-management bugs;
- authorization mistakes;
- difficult-to-test code paths.

### Recommended improvement

Split responsibilities into components such as:

```text
PlaybackController
QueueManager
MediaSessionManager
AudioEffectsManager
DownloadManager
ScrobblingManager
DiscordManager
CacheManager
```

Keep the Android service primarily responsible for lifecycle and service orchestration.

---

## 🟡 12. Large dependency/supply-chain surface

The project uses a substantial number of third-party libraries and additional Maven repositories.

### Why this matters

Every external dependency expands the supply-chain attack surface.

Particular attention should be given to:

```text
native libraries
FFmpeg-related components
YouTube extractors
network libraries
JitPack artifacts
custom Git dependencies
```

### Recommended improvements

- Pin dependency versions.
- Review dependency provenance.
- Prefer official Maven Central/Google repositories where possible.
- Minimize custom repositories.
- Generate and review an SBOM.
- Enable dependency vulnerability scanning.
- Use Dependabot/Renovate.
- Consider Gradle dependency verification.

---

## 🟡 13. Limited automated test coverage

The project documentation indicates that automated tests are limited.

### Why this matters

Security-sensitive changes in:

```text
MediaSession
MusicService
OAuth
networking
Room
downloads
WebSockets
```

can regress without being caught.

### Recommended improvements

Prioritize tests for:

```text
unauthorized media commands
OAuth callback validation
download URL validation
HTTP/HTTPS enforcement
token storage
backup behavior
WebSocket authentication
Intent handling
```

---

# 🔎 Recommended Security Audit Roadmap

The following sequence would provide the highest value for a deeper audit:

```text
1. REQUEST_INSTALL_PACKAGES
        ↓
2. APK download/install code
        ↓
3. Last.fm credential exposure/history
        ↓
4. Network endpoints + interceptors
        ↓
5. Listen Together WebSocket
        ↓
6. MusicService authorization
        ↓
7. Exported receivers/widgets
        ↓
8. Discord OAuth
        ↓
9. YouTube session/cookie handling
        ↓
10. Room/DataStore/backup
        ↓
11. FFmpeg/native/extractor dependencies
        ↓
12. GitHub Actions and build scripts
        ↓
13. Dependency/SBOM audit
```

---

# 🧪 Suggested Verification Commands

For a local clone, useful first-pass searches include:

```bash
# APK installation
rg -n "REQUEST_INSTALL_PACKAGES|PackageInstaller|ACTION_INSTALL_PACKAGE|ACTION_VIEW" .

# Hardcoded secrets
rg -n -i "api[_-]?key|secret|client[_-]?secret|token|password" .

# HTTP
rg -n "http://|cleartextTrafficPermitted" .

# Exported Android components
rg -n "android:exported=\"true\"" .

# WebSockets
rg -n "ws://|wss://|WebSocket" .

# OAuth
rg -n -i "oauth|pkce|authorization|redirect|callback|state" .

# APK/download operations
rg -n -i "download.*apk|\\.apk|install.*package|PackageInstaller" .

# YouTube authentication material
rg -n -i "SAPISID|cookie|Authorization|visitorData|dataSyncId" .
```

For credentials, use a dedicated secret scanner as well:

```bash
gitleaks detect --source . --redact
```

And review the Git history, not only the current working tree.

---

# 📌 Audit Status

| Area | Status |
|---|---|
| Hardcoded credentials | 🔴 **Investigate immediately** |
| APK installation capability | 🟠 **Investigate** |
| Cleartext networking | 🟠 **Harden** |
| MediaService authorization | 🟠 **Audit deeply** |
| Exported receivers | 🟠 **Audit deeply** |
| OAuth | 🟠 **Review** |
| YouTube session handling | 🟠 **Review deeply** |
| Backup/storage | 🟠 **Review** |
| Location | 🟡 **Review privacy** |
| Microphone | 🟡 **Review privacy** |
| Service architecture | 🟡 **Refactor candidate** |
| Dependency supply chain | 🟡 **Review** |
| Automated testing | 🟡 **Improve** |

---

## 👤 Auditor / Contact

**GitHub:** [@Alitan999](https://github.com/Alitan999)  
**Discord:** `alitan999`  
**Email:** `helloalitan.dev@gmail.com`

If you find a security issue while reproducing these findings, please document the affected version/commit, reproduction steps, expected behavior, actual behavior, and potential impact before reporting it publicly.

---

## ⚠️ Disclaimer

This document is an independent technical review of publicly available source code. Findings are based on static inspection and should be verified against the exact commit/build being evaluated.

A finding marked 🔴, 🟠, or 🟡 does **not** automatically mean the project is malicious or exploitable. Severity describes the priority for investigation or remediation based on the observed attack surface.

**Last reviewed:** 2026-10-01
