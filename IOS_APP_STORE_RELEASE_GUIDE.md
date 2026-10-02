# Publishing a Flutter App to the App Store — End-to-End Guide

**Team:** 101 SAS (1cero1 S.A.S.) · Mobile area
**Reference app:** 101 Fintech — first app published through this pipeline (Sept 2026), later reused for Parque Arví
**Owner of this doc:** Brayan Romero · **Last updated:** 2026-10-02
**Jira:** FSM — "Runbook de publicación iOS del área" (epic FSM-1)
**Spanish version:** [`GUIA_PUBLICACION_APP_STORE.md`](./GUIA_PUBLICACION_APP_STORE.md) (same content)

> **Why this exists.** The area has no Mac and, until now, only one person knew how to ship to the App Store. This guide lets anyone on the team take a Flutter app from repo to "Ready for Sale" using Codemagic, without depending on that person.

---

## Contents

0. [Quick reference](#0-quick-reference)
1. [The big picture](#1-the-big-picture)
2. [Is the app ready? Minimum requirements](#2-is-the-app-ready-minimum-requirements)
3. [Access: Apple account and the 101 SAS team](#3-access-apple-account-and-the-101-sas-team)
4. [Codemagic: API key, certificates and profiles](#4-codemagic-api-key-certificates-and-profiles)
5. [Per-app setup in Apple](#5-per-app-setup-in-apple)
6. [The reusable `codemagic.yaml`](#6-the-reusable-codemagicyaml)
7. [Build and TestFlight](#7-build-and-testflight)
8. [Screenshots with Figma (Platano Screenshots)](#8-screenshots-with-figma-platano-screenshots)
9. [Store listing and submission for review](#9-store-listing-and-submission-for-review)
10. [Errors we already hit (and how to fix them)](#10-errors-we-already-hit-and-how-to-fix-them)
11. [New-app checklist (copy into the Jira ticket)](#11-new-app-checklist-copy-into-the-jira-ticket)
12. [Secrets inventory and bus factor](#12-secrets-inventory-and-bus-factor)
13. [Glossary](#13-glossary)

---

## 0. Quick reference

| Item | Value |
|---|---|
| Apple Developer Program | **Organization**, legal entity "101 SAS" |
| Team ID | `43N22346WS` |
| Account Holder | Mauricio Villa Sierra — the only one who pays the membership and accepts agreements |
| Codemagic integration name | `101Software App Store` (referenced verbatim in every `codemagic.yaml`) |
| Codemagic machine | `mac_mini_m2`, manual trigger, **30 min cap** |
| Typical build time | ~15–25 min |
| Minimum Xcode for uploads | **Xcode 26 / iOS 26 SDK** (Apple requirement since 2026-04-28) → `xcode: latest` |
| Reference pipeline | `desarrollofasemovil/101Fintech` → `codemagic.yaml`, `docs/release/LANZAMIENTO_APP_STORE.md` |

**Apps already on this pipeline**

| App | Bundle ID | Apple ID (numeric) |
|---|---|---|
| 101 Fintech | `com.x101Fintech.fintech101` | `6815744048` |
| Parque Arví | ⟦fill in⟧ | `6817031302` |

> The iOS bundle ID **does not have to match** the Android package. 101 Fintech uses `com.x101Fintech.fintech101` on iOS and `com.x101Fintech.x101Fintech` on Android.

---

## 1. The big picture

```
 ONE TIME PER TEAM                      ONE TIME PER APP                       EVERY RELEASE
 ─────────────────                      ────────────────                       ─────────────
 Apple Developer membership  ──┐        App ID (Identifiers) + capabilities    Bump version in pubspec
 App Store Connect API key  ───┼──►     App record in App Store Connect   ──►  Start build in Codemagic
 Codemagic integration       ──┤        App Store provisioning profile         ↓
 Apple Distribution cert     ──┘        codemagic.yaml in the repo             TestFlight (internal test)
 APNs key (if push)                     Codemagic app + secrets group          ↓
                                        Store listing + screenshots            Submit for review → Release
```

Who touches what:

| Where | What happens there |
|---|---|
| **developer.apple.com** (Certificates, Identifiers & Profiles) | App IDs, certificates, profiles, APNs keys |
| **appstoreconnect.apple.com** | Users, API keys, app records, TestFlight, listing, review |
| **codemagic.io** | Builds the IPA on a Mac in the cloud, signs it, uploads it |
| **The repo** | `codemagic.yaml`, `Info.plist`, `project.pbxproj`, icons, `pubspec.yaml` version |

---

## 2. Is the app ready? Minimum requirements

Before spending macOS minutes, the Flutter app has to meet these basics. Every item here comes from a real rejection or a real wasted build.

### 2.1 Project configuration (`ios/`)

| Requirement | Where | Why |
|---|---|---|
| Unique bundle ID, set in all build configs | `ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER` | Must match the App ID and the provisioning profile exactly |
| `DEVELOPMENT_TEAM = 43N22346WS` | `project.pbxproj` | Signs under the 101 SAS team |
| **iPhone only:** `TARGETED_DEVICE_FAMILY = 1` in the **three** Runner configs (Debug/Release/Profile) | `project.pbxproj` | Avoids error 90474 and the iPad screenshot requirement — see §10.2 |
| Deployment target that your plugins accept | `IPHONEOS_DEPLOYMENT_TARGET` | 101 Fintech uses **15.0** because Firebase iOS SDK 12 requires it |
| `ITSAppUsesNonExemptEncryption = false` | `ios/Runner/Info.plist` | Otherwise every build is stuck in "Missing Compliance" — see §10.3 |
| A purpose string for **every** privacy API linked by a plugin | `Info.plist` (`NS…UsageDescription`) | Otherwise ITMS-90683 — see §10.4 |
| Icons with **no alpha channel** (all sizes, especially 1024×1024) | `ios/Runner/Assets.xcassets/AppIcon.appiconset/` | Otherwise ITMS-90717 — see §10.1 |
| Push capability in `Runner.entitlements` only if the app really uses push | `ios/Runner/Runner.entitlements` | The profile must include it or signing fails — see §10.6 |

> **No Mac? Edit the files as text.** `project.pbxproj` and `Info.plist` are plain text. Every fix in this guide was done without opening Xcode.

> **Dependencies:** with Flutter 3.44, native iOS dependencies go through **Swift Package Manager** by default, and 101 Fintech has **no Podfile**. If an older app still uses CocoaPods, `flutter build ipa` runs `pod install` on its own — nothing to add to the pipeline.

### 2.2 Product requirements (what the reviewer checks)

- **Version:** `pubspec.yaml` → `version: X.Y.Z+N`. `X.Y.Z` is what users see; `N` is the build number (the pipeline auto-raises it if needed).
- **Demo account** if there is any login. The reviewer will not get past the login screen without one, and that is an automatic rejection. 101 Fintech ships a demo mode in the same release binary (document `9000000101`, credentials in `docs/release/CUENTA_DEMO_REVISION.md` of that repo — **not** copied here).
- **Privacy policy URL** — mandatory for every app, even ones that collect nothing.
- **Support URL** — mandatory.
- **Account deletion** inside the app if the app lets users *create* accounts (App Review Guideline 5.1.1(v)).
- **Financial apps** must be published by the organization, not by an individual (Guideline 5.1.1(ix)) — 101 SAS already complies.
- **Use Apple's names for Apple features:** on iPhone say "Face ID", not "huella". 101 Fintech had to rename the biometric login copy for iOS (PR, 2026-09-25).
- **No dead ends:** buttons that do nothing, "coming soon" screens or placeholder content are a common rejection (Guideline 2.1, App Completeness).

---

## 3. Access: Apple account and the 101 SAS team

### 3.1 Roles that matter

| Role | Can do | Who should have it |
|---|---|---|
| **Account Holder** | Pay/renew membership, accept legal agreements, everything else | One person only (Mauricio Villa Sierra) |
| **Admin** | Invite users, create API keys, manage certificates, everything except agreements/payment | **At least two people in the mobile area** |
| **App Manager** | Manage apps, TestFlight, submit for review | Mobile developers who release |
| **Developer** | Upload builds, TestFlight | Developers who only build |

### 3.2 Getting a new person into the team

1. An **Admin** (or the Account Holder) goes to **App Store Connect → Users and Access → "+"**.
2. Enters the person's name and e-mail, picks the role (normally **App Manager**), and ticks **"Access to Certificates, Identifiers & Profiles"** if they will manage signing.
3. The person accepts the e-mail invitation, signs in with (or creates) an Apple Account, and **turns on two-factor authentication**.
4. They verify they see the team **"101 SAS"** in the top-right selector of both developer.apple.com and App Store Connect. If they belong to several teams, they must switch to 101 SAS before doing anything.

### 3.3 Lessons from recovering the account (Sept 2026)

The account was recovered with a phone call to Apple (2026-09-21) and the membership had been **expired since 2022-09-09**. To avoid repeating that:

- **Renewal date** in a shared calendar with a 30-day reminder. An expired membership blocks uploads and can remove apps from sale.
- **Pending agreements** (App Store Connect → Business) can only be accepted by the Account Holder. If an upload or a TestFlight build is blocked "for agreements", that is the cause.
- **Two Admins minimum**, each with their own Apple Account and their own trusted phone number. Never a shared login.
- Keep the Apple support case number and the membership details in the secrets vault (§12).

---

## 4. Codemagic: API key, certificates and profiles

These are **team-level** and done **once**. A new app reuses all of them except the provisioning profile.

### 4.1 App Store Connect API key (once per team)

1. **App Store Connect → Users and Access → Integrations → App Store Connect API → Team Keys → "+"**.
2. Name: `Codemagic`. Access: **App Manager**.
3. **Download the `.p8` immediately — Apple only lets you download it once.** Write down the **Issuer ID** (top of the page) and the **Key ID**.
4. Store the `.p8`, Issuer ID and Key ID in the secrets vault (§12).

### 4.2 Codemagic team integration (once per team)

1. **Codemagic → Team settings → Integrations → Developer Portal → Connect**.
2. Name: **`101Software App Store`** — exactly that, because every `codemagic.yaml` references it by name.
3. Paste Issuer ID, Key ID and upload the `.p8`.

> If the name differs by one character, the build fails at the start with "integration not found". Fix the name, not the yaml.

### 4.3 Apple Distribution certificate (once per team)

One distribution certificate signs **every** app of the team. Apple allows only a few at a time, so do not create one per app.

1. **Codemagic → Team settings → codemagic.yaml settings → Code signing identities → iOS certificates → Generate certificate**.
2. Type **Apple Distribution**, using the `101Software App Store` integration.
3. Codemagic shows a password for the `.p12`. **Save it in the vault** and download the `.p12` too: Codemagic keeps it, but that is the only other copy.

If it already exists (it does for 101 SAS), skip this step — just check it is listed and not expired (they last one year).

### 4.4 Provisioning profile (once per app)

1. **Code signing identities → iOS provisioning profiles → Fetch profiles**, pick the **App Store** profile for the app's bundle ID.
2. If it does not exist, create it in **developer.apple.com → Profiles → "+" → App Store Connect**, select the App ID and the distribution certificate above, then fetch again.

> **A profile is a snapshot of the App ID's capabilities.** If you add a capability later (e.g. Push Notifications), the old profile does not include it: regenerate the profile and fetch it again in Codemagic.

### 4.5 Add the app to Codemagic (once per app)

1. **Codemagic → Add application → GitHub → repo** (org `desarrollofasemovil`), project type **Flutter**.
2. In the app, **"Switch to YAML configuration"**. From then on Codemagic reads `codemagic.yaml` from the branch being built; the visual editor no longer applies.
3. If the app has secrets: **Environment variables → new group** (e.g. `api_myapp`), each variable marked **Secure**. The group name must match `environment.groups` in the yaml.

---

## 5. Per-app setup in Apple

Done once for each new app, before the first build.

1. **Register the App ID** — developer.apple.com → Identifiers → "+" → App IDs → App:
   - Description: app name. Bundle ID: **Explicit**, e.g. `com.x101Fintech.fintech101`.
   - Tick capabilities the app really uses (**Push Notifications** if it uses FCM, Sign in with Apple, Associated Domains…). Do this **before** creating the profile (§4.4).
2. **Create the app record** — App Store Connect → Apps → "+" → New App:
   - Platform iOS, name (≤30 chars, must be unique in the store), primary language (Spanish), bundle ID from step 1, SKU (internal, anything unique, e.g. `fintech101-ios`), User Access: Full.
3. **Write down the numeric Apple ID** — App Information → General → *Apple ID* (e.g. `6815744048`). It goes into `APP_STORE_APPLE_ID` in the yaml.
4. **Push only — APNs key.** developer.apple.com → Keys → "+" → *Apple Push Notifications service (APNs)*. Download the `.p8` (once only), note the Key ID, and upload it in **Firebase Console → Project settings → Cloud Messaging → Apple app configuration**. One APNs key works for **all** apps of the team, so create it once and reuse it. Without it, FCM push never reaches iPhones and **nothing shows an error**. *(Status: pending for 101 Fintech — tracked in FSM.)*

---

## 6. The reusable `codemagic.yaml`

The template is in this folder: **[`codemagic.template.yaml`](./codemagic.template.yaml)**. Copy it to the root of the app repo as `codemagic.yaml` and replace every `⟦CHANGE⟧`.

### 6.1 What to change per app

| Key | 101 Fintech value | Your app |
|---|---|---|
| `environment.flutter` | `3.44.0` | Same version you use locally/Android CI |
| `environment.groups` | `api_101fintech` | Your secrets group, or remove |
| `ios_signing.bundle_identifier` and `vars.BUNDLE_ID` | `com.x101Fintech.fintech101` | Your bundle ID |
| `vars.APP_STORE_APPLE_ID` | `6815744048` | Numeric Apple ID from §5.3 |
| `vars.BUILD_NUMBER_MINIMO` | `16` | Start at `1`; raise only after a burned build (§10.5) |
| "Compose env.json" step | 4 secrets via `jq` | Delete if the app has no `--dart-define` secrets |
| `--dart-define-from-file=env.json` | present | Only if you kept the step above |

**Do not change:** `integrations.app_store_connect`, `instance_type`, `max_build_duration`, `xcode: latest`, the signing step, the build-number step, `publishing`.

### 6.2 Why it is built this way

| Decision | Reasoning |
|---|---|
| **No `triggering` block** (manual start) | macOS minutes are shared by every app of the team. A routine push must not spend them; a release is a deliberate act. |
| **`max_build_duration: 30`** | A stuck build (signing prompt, hung upload) used to be able to eat 60 minutes. A healthy build takes 15–25. Lowered in PR #72. |
| **iOS only in Codemagic** | Android already builds on GitHub Actions (Linux, cheap). Duplicating it here wastes Mac time. |
| **`xcode-project use-profiles`** | Converts the Flutter project to manual signing with the fetched profile and writes `export_options.plist`. Without it, `flutter build ipa` tries automatic signing and fails on a CI Mac. |
| **Build number from TestFlight + floor** | Every upload of the same version needs a unique `CFBundleVersion`. Asking TestFlight for the latest and adding 1 lets you re-run a release without editing the pubspec. The floor covers the case TestFlight cannot see (§10.5). |
| **`--obfuscate --split-debug-info` + symbol artifacts** | Release binaries are obfuscated; without the Dart symbols and dSYMs, Crashlytics traces are unreadable **forever** — rebuilding the same commit does not regenerate them. |
| **`submit_to_testflight: false`** | The build lands for **internal** testers without Apple review. Moving to external testers or to App Review is a human decision in App Store Connect. |
| **Env check that fails fast** | A missing secret would compile fine and crash at the first login with an unrelated-looking error. Failing in the pipeline names the missing key (never its value). |
| **Pre-flight checks step** *(new in the template)* | Fails in a second if the plist encryption key is missing, the app declares iPad, or the 1024 icon has alpha — the three mistakes we already paid for. ⚠️ Added for the template, **not yet run on Codemagic**; if it misbehaves on the first app that uses it, fix or remove it. |

---

## 7. Build and TestFlight

### 7.1 Start a build

1. Bump `version:` in `pubspec.yaml` if it is a new version, commit and push.
2. **Codemagic → the app → Start new build → branch → workflow "iOS · TestFlight"**.
3. Watch the log. In order: Dependencies → (env.json) → Signing → Build number (prints `Version: X.Y.Z (N)`) → Pre-flight → Build IPA → Package symbols → Publishing.
4. When it finishes, **download and archive** `dart-symbols-ios-*.zip` and `dsyms-ios-*.zip` next to the release (team drive, folder per version). Not optional — see §6.2.

### 7.2 What happens after the upload

1. App Store Connect **processes** the build (usually 5–30 min). You get an e-mail when it is ready — or a rejection e-mail with an `ITMS-xxxxx` code (go to §10).
2. **TestFlight → iOS builds**: the build appears. Thanks to `ITSAppUsesNonExemptEncryption`, it does not stop at "Missing Compliance".
3. **Internal testing:** TestFlight → Internal Testing → group → add the build. Internal testers must be users of the team in App Store Connect (up to 100). **No Apple review.**
4. **External testing** (client, people outside the team): requires **Beta App Review** for the first build of each version. Usually fast, but it is a review.
5. Testers install the **TestFlight** app on their iPhone and accept the invitation.

### 7.3 Smoke test on a real iPhone (mandatory)

Without a Mac, **the first TestFlight build is the first time the app runs on iOS.** Before submitting, check at least:

- [ ] Cold start, splash, and the main flow end to end.
- [ ] Login with the **demo account** (exactly what the reviewer will use).
- [ ] Every permission prompt shows the right Spanish text.
- [ ] Face ID / biometrics (if any), external links, PDFs, payment redirects.
- [ ] Push (if any — needs the APNs key, §5.4).
- [ ] Safe areas (notch, Dynamic Island, home indicator) on every screen.

---

## 8. Screenshots with Figma (Platano Screenshots)

### 8.1 What Apple requires (iPhone-only app)

| Display | Accepted sizes (portrait, px) | Required? |
|---|---|---|
| **6.9"** | **1320 × 2868**, 1290 × 2796, 1260 × 2736 | **Yes** — Apple scales these down for smaller iPhones |
| 6.5" | 1284 × 2778, 1242 × 2688 | Only as a fallback if you have no 6.9" |
| iPad 13" | 2064 × 2752 / 2048 × 2732 | **Only if the app declares iPad** — another reason to ship iPhone-only |

- **1 to 10** screenshots per display size and language. Aim for **5–8**.
- **PNG or JPEG, no transparency.** A PNG can carry an alpha channel even if it looks opaque, and the upload is rejected. When in doubt, export **JPG**.
- Optional: up to 3 app previews (videos) — not used so far.

### 8.2 Workflow with Platano Screenshots

The **Platano Screenshots** Figma template already contains the iPhone device frames at the sizes Apple requires. Template link: ⟦paste Figma Community link⟧.

1. **Duplicate** the template into the 101 team's Figma space (one file per app, e.g. `AppStore Screenshots — <App>`).
2. **Get raw screens of the app**, in this order of preference:
   - **From a real iPhone via TestFlight** with the demo account. A screenshot from an iPhone Pro Max/Plus is already 1290 × 2796 or 1320 × 2868 — valid as-is.
   - **From the designer's Figma** (see §8.3).
   - From Android, only as a last resort: the status bar and navigation are Android-looking, and Apple can reject screenshots that do not show the app as it runs on iOS.
3. **Use deterministic data.** 101 Fintech's demo mode uses a fixed seed (`DEMO_DATOS_ALEATORIOS=false`) so every screenshot shows the same account. Never use real customer data.
4. Drop each screen into a **6.9" frame**, write the headline (short, benefit-oriented, in Spanish), keep text inside the safe area.
5. **Export** each frame at **1x**, **JPG** (or PNG without transparency), named in order: `01_home.jpg`, `02_movimientos.jpg`…
6. Store the exports in the repo or team drive under `store/ios/screenshots/<version>/` so the next release starts from them.

### 8.3 Reusing the designer's screenshots

If the designer already made store screenshots (e.g. for Google Play), reuse them — but **resize the frame, do not stretch the image**:

1. Check the original aspect ratio. Android/Play designs are often 9:16 (1080 × 1920); iPhone 6.9" is ~9:19.5. They are **not** the same shape.
2. In Figma, create a frame of exactly **1320 × 2868** (or 1290 × 2796), set the same background color/gradient as the design, and place the original composition inside **scaled proportionally** (hold Shift / use Scale tool `K`). Fill the extra vertical space with background, or move the headline up and give the device mockup more room.
3. If the design uses an Android device mockup, swap it for the iPhone frame from Platano Screenshots.
4. Export as in §8.2 step 5 and verify the pixel size of each file before uploading (Figma's export panel shows it; on Windows, file Properties → Details).

---

## 9. Store listing and submission for review

### 9.1 Fill in the listing (once per app, review each version)

| Section | Fields | Notes |
|---|---|---|
| **App Information** | Name (≤30), subtitle (≤30), category (primary + optional secondary), **Content Rights** | Content Rights: declare whether the app shows third-party content (photos, videos) and that you have the rights. It is a legal statement — confirm with the client. |
| **Pricing and Availability** | Price (free), countries | Ask the client: all countries or Colombia only. |
| **App Privacy** | Privacy policy URL, data-collection questionnaire | Answer from the **real** dependencies in `pubspec.yaml` (Firebase Analytics, Crashlytics, any SDK that collects data), not from memory. "Data Not Collected" is only valid if no SDK collects anything. Then **Publish** the privacy section. |
| **Age Rating** | Questionnaire | Most of our apps end at 4+. Answer "Unrestricted Web Access" = Yes only if the app lets the user browse any URL. |
| **Version page** | Screenshots, promotional text (≤170), description (≤4000), keywords (≤100, comma-separated), support URL, marketing URL (optional), copyright (`2026 101 SAS` or the client), "What's New" (not on the first version) | Prepare texts in a `.md` in the repo (Parque Arví used `docs/release/PROMPT_COWORK_APP_STORE.md`) and paste them; check character limits before pasting. |
| **App Review Information** | Contact name, phone, e-mail; **sign-in required → demo user and password**; notes | Notes: explain anything non-obvious (e.g. "payments open the bank's PSE gateway in the browser; the demo account simulates it"). |
| **Version Release** | Manual / automatic after approval / scheduled | Use **Manually release** so the client decides the launch moment. |

### 9.2 Submit

1. Version page → **Build** → "+" → pick the TestFlight build you smoke-tested.
2. Check the export-compliance answer was taken from the plist (no question shown).
3. **Add for Review → Submit to App Review**. Status goes *Waiting for Review → In Review → Pending Developer Release* (or *Rejected*).
4. Review usually takes from a few hours to a couple of days.

### 9.3 If Apple rejects

1. Read the message in **App Review → the submission** (Resolution Center). It cites a guideline number.
2. If it is **metadata only** (texts, screenshots, demo account), fix it in App Store Connect and reply/resubmit — no new build.
3. If it needs **code**, fix it, start a new Codemagic build (same version, the pipeline raises the build number), select the new build and resubmit.
4. If you think the reviewer is wrong, reply in the thread with specifics (steps, demo credentials, video). Appeals exist, but a clear reply usually solves it.
5. Log the rejection reason in the app's FSM ticket and, if it is new, **add it to §10 of this guide.**

### 9.4 After approval

- *Pending Developer Release* → **Release This Version** when the client agrees.
- Next updates: bump `version` in `pubspec.yaml` → build → TestFlight → new version in App Store Connect ("+ Version") → fill "What's New" → submit.

---

## 10. Errors we already hit (and how to fix them)

| # | Symptom | Code | Root cause | Fix |
|---|---|---|---|---|
| 10.1 | Upload rejected, "invalid large app icon… alpha channel" | **ITMS-90717** | Icons with transparency | Flatten icons on an opaque background |
| 10.2 | Upload rejected, iPad multitasking needs all orientations | **90474** | App declares iPad (`"1,2"`) but is portrait-only | `TARGETED_DEVICE_FAMILY = 1` |
| 10.3 | Every build stuck in "Missing Compliance" | — | No encryption declaration | `ITSAppUsesNonExemptEncryption = false` |
| 10.4 | Build rejected while processing, "missing purpose string" | **ITMS-90683** | A plugin links a privacy API without its `NS…UsageDescription` | Add the purpose string(s) |
| 10.5 | Next upload "build number already used" although TestFlight doesn't show it | — | A rejected build burned its number | Raise `BUILD_NUMBER_MINIMO` |
| 10.6 | Signing/upload fails on `aps-environment` | — | Profile created before enabling Push on the App ID | Enable Push, regenerate profile, fetch again |
| 10.7 | Build fails at start, integration not found | — | Integration name mismatch | Name must be `101Software App Store` |
| 10.8 | Upload rejected for SDK version | — | Built with Xcode < 26 | Keep `xcode: latest` |

### 10.1 Alpha channel in the icons — ITMS-90717

**What happened (101 Fintech, 2026-09-24):** all 21 PNGs in `AppIcon.appiconset` had transparent rounded corners. App Store Connect rejects the upload if the 1024×1024 icon has transparency.

**Fix used:** fill the transparent corners with the brand color (`#14387F`, the same background as the Android adaptive icon). Nothing changes visually because **iOS rounds the corners itself** — the icon must be a full square.

**How to prevent it in a new app:**

```yaml
# pubspec.yaml — flutter_launcher_icons
flutter_launcher_icons:
  ios: true
  image_path: "assets/images/logo.png"
  remove_alpha_ios: true
  background_color_ios: "#14387F"   # brand color behind the transparent parts
```

Then `dart run flutter_launcher_icons`.

**Check before building** (works on Windows/Linux, needs Python + Pillow):

```bash
python -c "import glob; from PIL import Image; [print(f, Image.open(f).mode) for f in glob.glob('ios/Runner/Assets.xcassets/AppIcon.appiconset/*.png')]"
```

Every line must say `RGB`. `RGBA` or `LA` = alpha channel → fix it. To flatten manually:

```bash
python -c "import glob; from PIL import Image; [Image.alpha_composite(Image.new('RGBA', (im:=Image.open(f).convert('RGBA')).size, '#14387F'), im).convert('RGB').save(f) for f in glob.glob('ios/Runner/Assets.xcassets/AppIcon.appiconset/*.png')]"
```

### 10.2 iPad / multitasking — error 90474

**What happened (Parque Arví and 101 Fintech, 2026-09-25):** the Flutter template ships with `TARGETED_DEVICE_FAMILY = "1,2"` (iPhone + iPad). An app that supports iPad with multitasking must support all four orientations, and our apps are locked to portrait (`Info.plist` + `SystemChrome.setPreferredOrientations`). App Store Connect rejected the upload.

**Fix used:** declare the app **iPhone-only**. In `ios/Runner.xcodeproj/project.pbxproj`, replace in the **three** Runner configurations (Debug, Release, Profile):

```diff
- TARGETED_DEVICE_FAMILY = "1,2";
+ TARGETED_DEVICE_FAMILY = 1;
```

Check: `grep -n TARGETED_DEVICE_FAMILY ios/Runner.xcodeproj/project.pbxproj` → only `= 1;` lines.

**Side effects (all positive for us):** no iPad screenshots required, and the app still installs on iPads in iPhone compatibility mode.

**Only if the client really wants iPad:** keep `"1,2"` and either support all four orientations, or add `UIRequiresFullScreen = true` to `Info.plist` (opts out of multitasking). Then you also need iPad screenshots and iPad testing.

### 10.3 `ITSAppUsesNonExemptEncryption`

**What happened (Parque Arví, detected during listing, 2026-09-29):** without this key, App Store Connect asks the export-compliance question for **every build**, and the build cannot be tested or submitted until someone answers it.

**Fix:** add to `ios/Runner/Info.plist`, inside the root `<dict>`:

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

**When `false` is correct:** the app only uses encryption provided by iOS or standard protocols — HTTPS/TLS, Keychain/secure storage, Face ID. That covers all our apps today, including 101 Fintech. **If an app ever ships its own or proprietary cryptography**, this must be `true` and the team must review export-compliance documentation. Do not copy the `false` blindly into such an app.

### 10.4 Permission texts — ITMS-90683

**What happened (101 Fintech build 2.0.2 (15), 2026-09-25):** Apple rejected the build while processing it because it lacked `NSPhotoLibraryUsageDescription` and `NSCameraUsageDescription`. **The app never asks for photos or camera** — it only attaches PDFs — but `file_picker` compiles its gallery/camera code (DKImagePickerController) and, under Swift Package Manager, there is no way to exclude it. Apple scans the binary, not the behavior.

**Rule:** every privacy API **linked** into the binary needs its purpose string, whether or not the app calls it.

**Fix used:**

```xml
<key>NSFaceIDUsageDescription</key>
<string>101 Fintech usa Face ID para autenticarte de forma segura.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>101 Fintech solo accede a tus fotos si eliges adjuntar un documento desde tu galería.</string>
<key>NSCameraUsageDescription</key>
<string>101 Fintech solo usa la cámara si eliges fotografiar un documento para adjuntarlo.</string>
```

**Common plugin → key map** (check each app's `pubspec.yaml`):

| Plugin | Keys usually required |
|---|---|
| `local_auth` | `NSFaceIDUsageDescription` |
| `file_picker`, `image_picker` | `NSPhotoLibraryUsageDescription`, `NSCameraUsageDescription` (image_picker may also need `NSMicrophoneUsageDescription` for video) |
| `geolocator`, `location` | `NSLocationWhenInUseUsageDescription` (+ `…AlwaysAndWhenInUse…` only for background) |
| `mobile_scanner`, `camera` | `NSCameraUsageDescription` |
| `contacts_service`, `flutter_contacts` | `NSContactsUsageDescription` |
| `speech_to_text` | `NSSpeechRecognitionUsageDescription`, `NSMicrophoneUsageDescription` |

**How to write the texts:**
- In **Spanish**, specific, naming the app and *why*: "X usa la cámara para fotografiar documentos que adjuntas". Generic texts ("Necesitamos acceso a la cámara") are a known rejection reason under Guideline 5.1.1.
- **Do not add keys "just in case".** Apple flagged `NSLocationWhenInUseUsageDescription` as not required for 101 Fintech, and `permission_handler` starts compiling location code when it sees that key. Add only what the rejection e-mail or the plugin docs ask for.

### 10.5 Burned build number

**What happened:** build 2.0.2 (**15**) was rejected while processing (ITMS-90683 above). Apple considers number 15 used, but **TestFlight does not list rejected builds**, so the pipeline's "latest build" lookup could not see it and would have reused 15.

**Fix:** `BUILD_NUMBER_MINIMO: 16` in `codemagic.yaml`. Whenever a build is rejected during processing, set the floor to *(that number + 1)*. Raise it; never lower it.

### 10.6 Push capability vs. provisioning profile

`Runner.entitlements` declares `aps-environment` (the value `development` is correct even for App Store; export replaces it with `production` from the profile). If the App ID did not have **Push Notifications** enabled when the profile was created, the profile does not include it and signing/upload fails. Enable the capability on the App ID → regenerate the App Store profile → **Fetch profiles** in Codemagic → rebuild.

If the app does **not** use push, remove the `aps-environment` key/entitlements file instead of enabling the capability.

### 10.7 Integration not found

The yaml says `app_store_connect: 101Software App Store`. The Codemagic team integration must have **exactly** that name (case and spaces included).

### 10.8 Old Xcode / SDK

Since **2026-04-28**, App Store Connect only accepts builds made with **Xcode 26+ / iOS 26 SDK**. `xcode: latest` keeps us compliant. If an app pins an older Xcode for some reason, the upload will be rejected.

### 10.9 Review-time rejections to anticipate

Not hit yet in our apps, but prepared for:

- **2.1 App Completeness** — reviewer can't log in → always provide a working demo account.
- **5.1.1** — vague permission texts, or account creation without in-app account deletion.
- **4.2 Minimum functionality** — apps that are just a website wrapper. Our municipal/WebView-heavy apps must show native value.
- **Using "huella" / Android wording on iOS** — say Face ID, use iOS-looking screenshots.

---

## 11. New-app checklist (copy into the Jira ticket)

**Repo (Flutter)**
- [ ] Bundle ID decided and set in `project.pbxproj` (all configs) — §2.1
- [ ] `DEVELOPMENT_TEAM = 43N22346WS`
- [ ] `TARGETED_DEVICE_FAMILY = 1` ×3 — §10.2
- [ ] Deployment target compatible with plugins
- [ ] `ITSAppUsesNonExemptEncryption = false` (if it qualifies) — §10.3
- [ ] Purpose strings for every privacy API linked by plugins — §10.4
- [ ] Icons RGB, no alpha (`remove_alpha_ios: true`) — §10.1
- [ ] Push entitlement only if push is used — §10.6
- [ ] Demo account / demo mode working in a release build — §2.2
- [ ] `codemagic.yaml` from the template, all `⟦CHANGE⟧` replaced — §6

**Apple**
- [ ] App ID registered with the right capabilities — §5.1
- [ ] App record created; numeric Apple ID copied into the yaml — §5.2–5.3
- [ ] APNs key uploaded to Firebase (if push) — §5.4

**Codemagic**
- [ ] App added from GitHub, switched to YAML — §4.5
- [ ] App Store provisioning profile fetched — §4.4
- [ ] Secrets group created (if needed) — §4.5

**Release**
- [ ] Build OK, symbols archived — §7.1
- [ ] Smoke test on a real iPhone via TestFlight — §7.3
- [ ] Screenshots 6.9" exported (JPG, no alpha) — §8
- [ ] Listing complete: texts, privacy (published), age rating, pricing, URLs, copyright, review info — §9.1
- [ ] Submitted with "Manually release" — §9.2
- [ ] Released and logged in FSM

---

## 12. Secrets inventory and bus factor

Everything below must live in the **team password vault** ⟦name/location⟧, never in a repo, chat or e-mail. At least two people must have access.

| Secret | Created in | Used by | Rotation / expiry |
|---|---|---|---|
| Apple Accounts of Admins (+2FA phones) | Apple | People | — |
| App Store Connect API key (`.p8`, Issuer ID, Key ID) | App Store Connect → Integrations | Codemagic integration | Revoke + recreate if leaked; `.p8` downloadable only once |
| Apple Distribution certificate (`.p12` + password) | Codemagic / developer.apple.com | All apps' signing | **Expires yearly** — calendar reminder |
| APNs key (`.p8`, Key ID) | developer.apple.com → Keys | Firebase (all apps) | No expiry; revoke if leaked |
| Per-app secrets (e.g. `api_101fintech` group) | The provider/client | Codemagic env group | Per app |
| Demo/review account credentials | The app's repo docs | App Review Information | Change if leaked |
| Membership renewal date | Apple | Account Holder | **Yearly** — calendar reminder |

**Escalation:** blocked by agreements or payment → Account Holder (Mauricio Villa Sierra). Blocked by account access → any Admin ⟦names⟧. Pipeline/signing → mobile area owner ⟦name⟧.

---

## 13. Glossary

| Term | Meaning |
|---|---|
| **Bundle ID** | Reverse-DNS identifier of the iOS app (`com.x101Fintech.fintech101`). Permanent once the app is published. |
| **App ID** | The bundle ID registered in developer.apple.com, with its capabilities (Push, etc.). |
| **Apple ID (numeric)** | App Store Connect's number for the app record (`6815744048`). Not a secret. |
| **Team ID** | Identifier of the 101 SAS developer team (`43N22346WS`). |
| **Distribution certificate** | Cryptographic identity that signs App Store builds for the whole team. |
| **Provisioning profile** | Ties one App ID + one certificate + capabilities. One per app. |
| **`.p8` key** | Private key file Apple lets you download once (API key, APNs key). |
| **IPA** | The iOS app package that gets uploaded. |
| **dSYM / Dart symbols** | Files that translate obfuscated crash traces back to readable code. |
| **TestFlight** | Apple's beta distribution. Internal testers: no review. External: Beta App Review. |
| **`CFBundleShortVersionString` / `CFBundleVersion`** | User-visible version (`2.0.3`) / build number (`16`). Flutter fills them from `pubspec.yaml`. |

---

*Sources for Apple requirements as of 2026-10-02: [Apple — SDK minimum requirements](https://developer.apple.com/news/upcoming-requirements/), [App Store Connect screenshot specifications (summary)](https://appdrift.co/guides/app-store-connect/screenshot-requirements). Everything team-specific comes from the 101 Fintech repo (`codemagic.yaml`, `docs/release/`, commits 7825598, 1b42424, 8850343, cf3c309, 8b4414d) and the Parque Arví release work.*
