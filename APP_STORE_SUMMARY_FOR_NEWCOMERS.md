# How We Publish an App to the App Store — Summary for Newcomers

**Who it's for:** anyone joining the 101 SAS mobile area, whether or not they know how to code.
**Reading time:** 10 minutes.
**Need the technical detail?** It's in the [full guide](./IOS_APP_STORE_RELEASE_GUIDE.md). Each topic here points to its section (§).

---

## The idea in one sentence

To get an app into the App Store, the company **builds** it, **signs** it, **tests** it and **hands it to Apple for review**. Since we have no Mac computers, the building is done by a cloud service called **Codemagic**.

---

## The pieces, explained with a comparison

Think of publishing an app as **getting a product onto the shelves of a very strict supermarket**.

| Piece | What it is | In the comparison |
|---|---|---|
| **Apple Developer account** | The company's yearly membership with Apple | The company's contract with the supermarket. If it expires, nothing can be sold |
| **"101 SAS" team** | The people in the company with access to that account | The list of employees allowed into the warehouse |
| **App Store Connect** | Apple's website where apps are managed | The supermarket's office: you register the product, put on the label and ask for approval |
| **Codemagic** | A Mac on the internet that turns the code into an installable app | The factory that packages the product |
| **Certificates and profiles** | Files that prove the app is ours | The company seal on the package. Without it, Apple won't accept it |
| **TestFlight** | Apple's app for testing before publishing | Free samples for the team before it goes on the shelf |
| **Apple review** | A person at Apple tries the app and approves or rejects it | The supermarket's quality control |

---

## The path, in 6 steps

```
 1. Prepare  →  2. Build    →  3. Test       →  4. Dress     →  5. Apple    →  6. Publish
    the app      in Codemagic    in TestFlight     the listing     review
```

1. **Prepare the app.** The developer checks that the app meets Apple's basics (see "What Apple always checks" below). → §2
2. **Build it in Codemagic.** One button, and in about 20 minutes Codemagic builds the app, signs it and sends it to Apple. → §7.1
3. **Test it in TestFlight.** The team installs it on a real iPhone and uses it like a customer would. **Mandatory:** with no Mac, this is the first time the app runs on an iPhone. → §7.3
4. **Fill in the store listing.** Name, description, screenshots, privacy policy and a **test account** for Apple's reviewer. Screenshots are made in Figma with the *Platano Screenshots* template, which already has the right iPhone sizes. → §8 and §9.1
5. **Submit for review.** Apple reviews it, usually within a few hours to a couple of days. If rejected, Apple says why; we fix it and resubmit. → §9.2 and §9.3
6. **Publish.** Once approved, we release it on the day the client chooses. → §9.4

**For updates**, repeat steps 2 to 6. The big setup (step 1 and the account configuration) is done once per app.

---

## What Apple always checks (and has cost us rejections)

You don't need to know how to fix these. You do need to know they exist, so you can ask the developer to check them **before** submitting:

| What happened | In plain words | How to avoid it |
|---|---|---|
| **Icon rejected** | The icon had transparent corners; Apple requires a solid square | Icon without transparency (the iPhone rounds the corners itself) |
| **"The app says it works on iPad"** | Our apps are made for iPhone in portrait; Apple demands more if you claim iPad support | Declare the app **iPhone only** |
| **Encryption question on every upload** | Apple asks whether the app uses special encryption. If not answered in advance, every version waits | Write the answer inside the app |
| **Missing permission texts** | If the app *could* use the camera, photos or Face ID, Apple requires a text explaining why, even if the app never asks | Write those texts in Spanish, clear and specific |
| **Reviewer can't log in** | If the app has a login, the reviewer needs a test user | Always provide a working **demo account** |

The technical explanation of each is in §10 of the full guide.

---

## Who does what

| I need… | Who does it |
|---|---|
| Pay/renew the membership, accept contracts with Apple | **Account Holder:** Mauricio Villa Sierra (only he can) |
| Access to the 101 SAS team | A team **Admin** ⟦names⟧ |
| Build the app and upload it | A mobile developer with Codemagic access |
| Listing texts, screenshots and details | Developer + designer + client |
| Decide the release day | The client |

**When you join:** ask to be invited to App Store Connect with the right role and **turn on two-factor authentication** on your Apple account. Never use someone else's account. → §3.2

---

## How long each thing takes (approx.)

| Step | Time |
|---|---|
| Build in Codemagic | 15–25 minutes |
| Apple processes the build into TestFlight | 5–30 minutes |
| Apple review | A few hours to a couple of days |
| First release of a new app (whole process) | Plan **1 to 2 weeks**, for listing, screenshots and a possible rejection |

---

## Three rules we never break

1. **Keys and secret files only live in the team password vault** ⟦name⟧. Never in chat, e-mail or the code.
2. **Never submit a version nobody has tested on a real iPhone.**
3. **Every new rejection gets written down** in the app's Jira ticket and in §10 of the guide, so it doesn't happen again.

---

## Words you'll hear

| Word | Meaning |
|---|---|
| **Build** | A packaged version of the app, ready to install |
| **Bundle ID** | The app's "ID number" with Apple. It never changes |
| **Pipeline** | The automatic recipe Codemagic follows to build the app |
| **TestFlight** | Where we test the app before publishing |
| **Rejection** | Apple didn't approve the app and explains why |
| **Listing** | What the public sees in the store: name, pictures, description |

---

*Based on the 101 Fintech and Parque Arví releases (Sept 2026). Last updated: 2026-10-02. Owner: Brayan Romero.*
