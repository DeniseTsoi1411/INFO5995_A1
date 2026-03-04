# AI Usage Log — INFO5995 Assignment 1

## Tools / References Used
- JADX decompiler (local binary downloaded from official GitHub release).
- USENIX LaTeX template + style file (downloaded from usenix.org author resources page).

## Step-by-step AI Interaction (Decompilation Setup)
1. **Prompt:** "Suggest a decompilation workflow for the provided APK in Windows."
   **Response summary:** Use JADX to decompile APK into Java sources; verify manifest and activities; search for auth and token logic with `rg`.
2. **Prompt:** "How can I locate randomness-related code in a decompiled Android project?"
   **Response summary:** Search for `Random`, `SecureRandom`, `UUID`, `Math.random`, and token-related strings. Build a table of findings and map each to its security role.

## Analysis Notes (Key AI-Assisted Decisions)
- Chose JADX to decompile `a1_case1.apk`, then searched Java sources for randomness and session token usage.
- Focused on `Login.generateSessionToken()` because `java.util.Random` is predictable and used for a session token (security-sensitive).

## Code Parts That Raised Main Red Flags
1. **Security-critical token generated with weak PRNG (primary vulnerability)**
   - File: `a1_case1_jadx/sources/com/example/mastg_test0016/Login.java`
   - Key lines:
     - `createSession()` writes token into session prefs (around lines 175-177).
     - `generateSessionToken()` uses `Random random = new Random();` (around line 185).
     - Token characters derived from `random.nextInt(62)` (around line 188).
   - Why flagged: `java.util.Random` is predictable and not suitable for authentication/session tokens.

2. **Session token persistence and reuse path increases impact (supporting flag)**
   - Files:
     - `a1_case1_jadx/sources/com/example/mastg_test0016/Login.java`
     - `a1_case1_jadx/sources/com/example/mastg_test0016/Profile.java`
   - Key lines:
     - `sessionToken` key usage in `Login.java` (around lines 26, 176, 181).
     - `SessionPrefs` / `sessionToken` constants and prefs load in `Profile.java` (around lines 16-17, 32-34).
   - Why flagged: predictable token is stored/read through app session state, making token prediction/storage tampering a practical attack path under local attacker assumptions.

3. **Use of randomness in non-security context (triage flag, not reported as vuln)**
   - File: `a1_case1_jadx/sources/com/example/mastg_test0016/MainActivity.java`
   - Key lines:
     - `randomNumberGenerator()` uses `new Random().nextInt(100)` (around lines 17-19).
   - Why flagged: all randomness sites were audited; this one appears UI/debug oriented and not authentication-sensitive, so it was documented but not treated as the main vulnerability.

## Rubric-Driven Mock Q&A (with Evidence)
### Rubric (copied)
**System & Threat Model (4 marks)**
- 4 marks: clear system model, key assets/data flows, and realistic attacker goals/capabilities explicitly linked to the chosen vulnerability.
- 3 marks: main components/assets and at least one plausible attacker are described; model is mostly consistent with the rest of the work.
- 1–2 marks: partial model (e.g., components/assets only) with vague/unrealistic attacker assumptions or weak vulnerability linkage.
- 0 marks: system/threat model largely missing or factually incorrect.

**Vulnerability Discovery & Explanation (6 marks)**
- 5–6 marks: correctly identifies a security-relevant weakness in “random-looking” value generation/use; explains discovery method (tooling and/or AI), risk, and a concrete step-by-step attack.
- 3–4 marks: identifies the intended (or closely related) vulnerability; explains risk and a plausible attack with some discovery evidence.
- 1–2 marks: issue is partially correct or focuses on a minor problem; attack scenario is vague, incomplete, or only loosely related to the real risk.
- 0 marks: no valid vulnerability identified, or explanation is mostly incorrect.

**Fix / Mitigation (2 marks)**
- 2 marks: proposes a concrete, technically sound fix and explains why it works.
- 1.5 marks: proposes a reasonable fix addressing the main risk, but with minor missing technical detail/justification.
- 0.5–1 mark: fix is vague, incomplete, or only partially addresses the real problem.
- 0 marks: no fix or mitigation, or suggestion does not address the vulnerability.

**Recorded Presentation & Q&A (3 marks)**
- 3 marks: recorded talk (≤5 minutes) clearly covers model, vulnerability, and fix; delivery is clear, and Q&A gives strong evidence-based grade/contribution justification.
- 2 marks: recorded talk (≤5 minutes) covers main points with generally clear delivery; Q&A is adequate but with minor grade/contribution justification gaps.
- 1 mark: recorded talk misses key elements or is hard to follow; Q&A shows limited understanding or weak grade/contribution justification.
- 0 marks: no presentation submitted, or recording is unusable.

### Mock Q&A
**Q1:** Where exactly is the weak randomness used, and why is it security relevant?
**A1:** In `Login.generateSessionToken()` the app uses `java.util.Random` to build a 16-character session token stored in `SharedPreferences` under `SessionPrefs`/`sessionToken`. Because the token represents authentication state, predictability enables impersonation.

**Q2:** What is the concrete attacker model and how does it exploit the weakness?
**A2:** A local attacker with root/ADB access can observe the approximate login time and read/modify app storage. They brute-force the `Random` seed over a time window to reproduce the token, then overwrite `SharedPreferences` to assume the victim’s session.

**Q3:** What is the fix and why does it work?
**A3:** Use `java.security.SecureRandom` to generate 128 bits of entropy and encode the token. SecureRandom is designed to be unpredictable, preventing feasible token prediction.

### Key Improvements Noted
- Make sure the attacker capabilities are explicitly linked to token prediction and storage access.
- Include a concrete attack flow and explicitly state where the token is stored.

## Second-Pass Audit (Additional Review)
### Scope
- Reviewed app-owned logic classes:
  - `a1_case1_jadx/sources/com/example/mastg_test0016/MainActivity.java`
  - `a1_case1_jadx/sources/com/example/mastg_test0016/Login.java`
  - `a1_case1_jadx/sources/com/example/mastg_test0016/Profile.java`
- Reviewed:
  - `a1_case1_jadx/resources/AndroidManifest.xml`
  - `a1_case1_jadx/resources/res/layout/activity_main.xml`
  - `a1_case1_jadx/resources/res/layout/activity_login.xml`
  - `a1_case1_jadx/resources/res/layout/activity_profile.xml`
  - `a1_case1_jadx/resources/res/xml/backup_rules.xml`
  - `a1_case1_jadx/resources/res/xml/data_extraction_rules.xml`

### Findings
1. **High**: Predictable session token generation (`java.util.Random`) in `Login.generateSessionToken()` (already reported).
2. **Medium**: Plaintext credential storage in internal file.
   - Evidence:
     - `MainActivity.java` writes `Username: <u> Password: <p>` to `credentials.txt` (around lines 59-60).
     - `Login.java` reads and compares plaintext credentials from `credentials.txt` (around lines 77-93).
   - Risk:
     - Credentials are not hashed/salted and remain recoverable to local attackers with elevated access (e.g., rooted device, backup/extraction paths).
3. **Low / informational**: Logging of non-security random number and login result (`Log.d(...)`) exists but does not by itself expose secrets in this code.

### Confidence Statement
- Confidence is **high** that no additional **critical** vulnerabilities exist in the app-owned code reviewed above.
- Confidence is **not absolute** because this is static decompiled analysis and does not cover:
  - runtime-only behavior,
  - device/environment-specific misconfigurations,
  - vulnerabilities in third-party AndroidX/library internals beyond app usage.

## Open Items
- Record and include the ≤5 minute presentation file.
- Provide PoC video if required by submission package.
