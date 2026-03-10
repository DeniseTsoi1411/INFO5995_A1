# AI Usage Log - INFO5995 Assignment 1 (Alice Combined Version)

## Tools and References Used
- Android SDK `apkanalyzer` (manifest, package tree, smali-like method extraction)
- Android SDK build tools (`aapt`, `dexdump` available)
- USENIX paper template (`usenix-2020-09`)

## Step-by-Step AI Interaction Summary
1. Prompt: suggest APK auditing workflow for assignment constraints.
   - Response used: decompile, identify app model, enumerate randomness, map to security roles.
2. Prompt: how to find security-sensitive randomness in Android code.
   - Response used: inspect `Random`, `SecureRandom`, token/session identifiers, and auth flows.
3. Prompt: rubric-driven self-check.
   - Response used: verify system model, attacker capability realism, exploit path clarity, and fix justification.

## Analysis Workflow Performed
1. Confirmed assignment requirements from rubric/spec.
2. Extracted APK metadata:
   - package id: `com.example.mastg_test0016`
   - launcher activity: `MainActivity`
   - app activities: `MainActivity`, `Login`, `Profile`
3. Enumerated app-defined dex classes/methods with:
   - `apkanalyzer dex packages --defined-only`
4. Decompiled key classes to smali-like output:
   - `Login`, `Login$1`, `MainActivity`, `Profile`
5. Triaged randomness and session usage to pick primary vulnerability.

## Key Evidence Logged
### Primary vulnerability: weak PRNG for session token
- File: `decompiled/Login.smali`
- Evidence:
  - `generateSessionToken()` starts at about line 337
  - `new-instance v0, Ljava/util/Random;` around line 341
  - `Random->nextInt(62)` around line 360

### Session state persistence path
- File: `decompiled/Login.smali`
- Evidence:
  - `createSession()` around line 414
  - writes key `sessionToken` to `SharedPreferences` around line 425

### Login success path to profile
- File: `decompiled/Login$1.smali`
- Evidence:
  - calls `createSession()` around line 123
  - starts `Profile` activity around lines 126-137

### Additional randomness site (non-primary)
- File: `decompiled/MainActivity.smali`
- Evidence:
  - `randomNumberGenerator()` uses `java.util.Random` around lines 234-267
  - treated as non-auth debug/UI randomness

## Supporting Observation (Secondary)
- Credentials are written/read as plaintext in local file flow (`credentials.txt`), which increases local compromise impact.
- Not selected as core vulnerability because rubric hint targets randomness/crypto misuse.

## Rubric-Driven Mock Q and A
Q1: Where is the intended weakness?
- A1: In `Login.generateSessionToken()` using `java.util.Random` for auth session tokens.

Q2: Why is it security-relevant?
- A2: Tokens represent authenticated state; predictable generation weakens impersonation resistance.

Q3: What attacker is realistic?
- A3: Local adversary with ability to inspect or modify app-local state (for example rooted/debug-capable environment).

Q4: What is the concrete fix?
- A4: Replace with `java.security.SecureRandom` and enforce token validation at protected entry points.

## Improvements Applied After Review
- Added exact code-location evidence from extracted smali files.
- Tightened attacker model to avoid over-claiming remote impact.
- Kept one clear primary vulnerability to align with rubric scoring.

## Reproducibility Commands
```bash
~/Library/Android/sdk/cmdline-tools/latest/bin/apkanalyzer manifest print a1_case1.apk
~/Library/Android/sdk/cmdline-tools/latest/bin/apkanalyzer dex packages --defined-only a1_case1.apk
~/Library/Android/sdk/cmdline-tools/latest/bin/apkanalyzer dex code --class com.example.mastg_test0016.Login a1_case1.apk
~/Library/Android/sdk/cmdline-tools/latest/bin/apkanalyzer dex code --class 'com.example.mastg_test0016.Login$1' a1_case1.apk
~/Library/Android/sdk/cmdline-tools/latest/bin/apkanalyzer dex code --class com.example.mastg_test0016.Profile a1_case1.apk
```

## Open Items Before Submission
- Replace placeholder author names in `report.tex`.
- Compile to `report.pdf` and verify max 2-page limit.
- Record presentation (`<= 5 min`) and check each member speaks >= 40 seconds.
- Include contribution summary (`activity-log.pdf`).

## Exceeds-Expectation Coverage Map
### System and Threat Model (target: 4/4)
- Clear system model and assets are documented in `report.tex` Section 2:
  - components: `MainActivity`, `Login`, `Profile`
  - assets: credentials, session token, authenticated profile access
  - data flow: registration -> local file, login -> session token -> profile
- Realistic attacker capabilities are explicitly linked to vulnerability:
  - local attacker can inspect/tamper local app state
  - vulnerability impact: session impersonation via predictable token + trusted local session storage

### Vulnerability Discovery and Explanation (target: 5-6/6)
- Discovery method is documented:
  - tooling: `apkanalyzer` class/method extraction
  - AI-assisted workflow + triage process
- Security-relevant weakness correctly identified:
  - `java.util.Random` used in auth session token generation
- Concrete attack path included:
  - capability -> observation window -> candidate prediction -> token replay/overwrite -> profile access
- Evidence locations are specific:
  - `decompiled/Login.smali` token generation and storage path
  - `decompiled/Login$1.smali` login-success -> session creation -> profile transition

### Fix/Mitigation (target: 2/2)
- Concrete fix included in `report.tex` Section 6:
  - replace `java.util.Random` with `java.security.SecureRandom`
  - enforce token validation at protected entry points (`Profile.onCreate`)
- Why it works is explicitly justified:
  - cryptographic unpredictability + elimination of blind trust in local session state

### Recorded Presentation and Q&A (target: 3/3)
- Prepared artifacts:
  - `presentation-plan.md` (5-minute script and evidence points)
  - `qa-bank.md` (evidence-first Q&A responses)
  - `activity-log-template.md` (contribution fairness support)
- Compliance checks:
  - total recording <= 5 minutes
  - each member >= 40 seconds speaking time
  - each claim in Q&A tied to code evidence and file location
