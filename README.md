# App Store Guide — 101 SAS Mobile Area

How the 101 SAS mobile area publishes Flutter apps to the Apple App Store, from repo to "Ready for Sale", using Codemagic (no Mac needed).

Built from the first real release through this pipeline (101 Fintech, Sept 2026) and reused for Parque Arví.

## Start here

| If you are… | Read |
|---|---|
| New to the area, or not technical | **Summary** — [Español](./RESUMEN_PUBLICACION_APP_STORE.md) · [English](./APP_STORE_SUMMARY_FOR_NEWCOMERS.md) (10 min) |
| The developer doing the release | **Full guide** — [Español](./GUIA_PUBLICACION_APP_STORE.md) · [English](./IOS_APP_STORE_RELEASE_GUIDE.md) |
| Setting up Codemagic for a new app | [`codemagic.template.yaml`](./codemagic.template.yaml) — copy to the app repo as `codemagic.yaml` and replace every `⟦CHANGE⟧` |

## What the full guide covers

1. Access to the Apple Developer account and the 101 SAS team
2. Certificates, provisioning profiles and the App Store Connect API key in Codemagic
3. The reusable `codemagic.yaml`
4. TestFlight and submission for review
5. Screenshots with Figma (Platano Screenshots template)
6. Errors already seen: alpha channel in icons (ITMS-90717), iPad error 90474, `ITSAppUsesNonExemptEncryption`, permission texts (ITMS-90683)
7. New-app checklist, secrets inventory and glossary

## Keeping it current

- Every new App Store rejection goes into §10 of **both** full guides (ES and EN), with what happened and the fix.
- Update the summary only if the process itself changes.
- Never commit secrets here (`.p8`, `.p12`, passwords, demo credentials). This repo only says *where* they live.

**Owner:** Brayan Romero · Jira: FSM (epic FSM-1)
