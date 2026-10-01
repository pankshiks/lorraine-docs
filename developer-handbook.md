# Developer handbook (main)

> Part 1 · Understand the project. Read this before anything else — it's the whole story in plain words.

## 01 · The project in five minutes

### The client
- DeYoe Wellness Acupuncture, Georgia, USA. Three offices: Clarkesville, Blairsville, Decatur.
- Dr. James DeYoe, LAc — the acupuncturist. Signs the templates. Does **not** use the tool. *(Changed 29 Sep, user decision awaiting Poonam: he signs each visit on S9, and the RFS (boxes 21 and 29) on S11, in signature boxes — the only things he does in the tool. *Client instruction 1 Oct: box 21 only — RFS page 2 is faxed completely blank.*)*
- Lorraine Fordham, Practice Manager — the **only user** at go-live. Prepares, approves and sends everything. Non-technical, busy, careful.
- Office is 7 hours behind India (US Eastern).

### What they need
- They treat military veterans paid by the **VA** (US Department of Veterans Affairs) under the **Community Care** programme.
- For every VA **authorization** (an approval for N visits), they must send the VA a clinical note for every visit, plus a fax cover sheet and sometimes an RFS (Request for Service) form.
- Today this is typed by hand onto scanned paper templates and faxed. A 12-visit packet takes an afternoon and mistakes creep in.

### What we're building
A small web app on the office Mac. Lorraine picks a patient and diagnosis, records each visit (date, pain level, the patient's answers), and the tool produces the three fax files — cover sheet, clinical notes, RFS — printed on the practice's own scanned forms *(changed 25 Sep: generated as new PDFs from the practice's forms kept as HTML templates — awaiting Poonam's confirmation, see §09 rule 2)*. She reviews them and the tool faxes them through SRFax, then records whether they arrived. Grok (xAI) can read Dr. DeYoe's note and suggest the diagnosis and first-visit answers. **Patient identifiers are never sent to Grok.**

### Glossary
| Term | Meaning |
|---|---|
| VA | US Department of Veterans Affairs. Pays for the veterans' acupuncture. |
| Community Care | VA programme that sends veterans to outside providers such as DeYoe. |
| Authorization | The VA's approval: one diagnosis, a number of visits, a date period and a VA number like `VA0012345678`. A packet is built per authorization. |
| VAMC / VA facility | The VA medical centre that receives the fax: Atlanta VAMC, Charlie Norwood VAMC (Augusta), Blairsville VA. |
| Progress note / clinical note | One page per visit, on the diagnosis's scanned template, signed by Dr. DeYoe. |
| Diagnosis template | The scanned blank progress note for one diagnosis, with that diagnosis's usual ticks already on it. There are 20. |
| Cover sheet | The fax cover page. One per destination. Five versions. |
| RFS | Request for Service — VA Form 10-10172, two pages, asks the VA to continue care. Three versions, one per office address. |
| ICD-10 | The standard diagnosis code printed on the RFS. Received 22 Sep for all 20, stored with the dot (M54.50). Hip, knee and shoulder use one code for both sides, as on the practice's list (user decision 28 Sep; practice to confirm). |
| HSRM | The VA's online portal. The practice uploads there by hand. **Out of scope** — we only let them download the files. |
| SRFax | The practice's fax provider (HIPAA, API). Practice fax number 404-738-1714. |
| BAA | Business Associate Agreement (HIPAA). Signed with us and with SRFax. Real patient data is allowed only in production on the Mac. |
| PHI | Protected health information. Anything that identifies a patient. **Never in the repo, designs, prompts, tests, or screenshots.** |

## 02 · Today's process, and what changes

| # | Today (by hand) | With the tool |
|---|---|---|
| 1 | VA approves an authorization (diagnosis, visits, VA number). | Lorraine enters it once (S6). |
| 2 | Dr. DeYoe writes a short note: diagnosis, symptoms, which template to use. | She can paste it; Grok suggests diagnosis and first-visit answers (S10). |
| 3 | Patient visits happen over weeks. | After each visit she records date, pain level, answers (S9) — mostly pre-filled. |
| 4 | She picks the diagnosis template and types a page per visit: name, dates (top and bottom), pain, VA number, last four, labels, notes. | The tool prints every page onto the scan. Date entered once, printed twice. |
| 5 | She fills a cover sheet and maybe an RFS, counts pages by hand. | Built automatically, page count calculated (S11). |
| 6 | Faxes three separate files through SRFax. Blairsville sends are also faxed to Atlanta. | Reviewed (S12, S13), sent, both destinations automatic, delivery tracked (S14, S15). |
| 7 | Uploads to HSRM. | Still manual — she downloads the files from S12. |

**The mistakes this prevents** (seen in the practice's own samples): the last four differs between notes and cover sheet (1234 vs 1111; 0001 vs 1234); the top and bottom dates on a page differ (04/03 vs 4/01); page counts typed wrong; identical patient answers on every page even when the doctor's note says otherwise.

## 03 · Scope, deliverables and acceptance

### In scope
- 20 diagnosis templates with first / middle / last page rules
- Per-visit entry with pre-filled answers and visit limits
- 5 cover sheets, 3 RFS versions (always both pages), 3 separate files per fax
- Consistency checks across files
- Grok assist with identifier redaction
- SRFax sending: review, confirm, status polling, log, test fax *(Built 30 Sep: background sender `backend/app/fax/sender.py` — hands approved faxes to SRFax, polls, logs; Retry = new attempt; S14/S15 from the fax records; SRFax details entered on S19, password encrypted. `FAX_MODE=fake` until the practice's SRFax details arrive; outside production only `FAX_ALLOWED_NUMBERS` are dialled. Needs a real test fax to confirm SRFax's field names and status words.)*
- Full activity log
- Install on the office Mac, backups, handoff document, one walkthrough call, 14-day bug-fix window

### Not in scope
- HSRM or any VA portal upload; claims or billing
- EHR, scheduling or billing integrations; inbound faxes
- Redesigning their forms or writing clinical wording
- An on-screen form editor for the practice (we maintain diagnosis settings)
- The 22 additional diagnoses (separate add-on, post go-live)
- Clinic sign-up / multi-clinic platform features
- Mobile apps; support after the 14-day window

### Acceptance criteria (what the client signs off)
| Area | Must be true |
|---|---|
| Notes | A 12-visit and an 8-visit packet generate correctly: page 1 has the initial-visit label and note; the last page has the last-visit label, closing note and "Excellent"; middle pages change only date, pain and the patient's answers. Top and bottom dates match on every page. An invalid or mismatched value blocks finalizing with a clear message. A single page can be edited and the change appears in the output. |
| Cover + RFS | A package of 1 cover + RFS + 8 notes produces three files in the right order, and the cover sheet shows the correct total page count. RFS page 1 shows the correct Reason for Request wording for the patient; page 2 is included. A mismatch between cover/RFS and notes is flagged. |
| Staff screens | A non-technical person can create, preview, edit and approve a packet using only the handoff document. |
| Grok | A sample doctor's note becomes the correct fields. A test proves no identifiers are in requests. Drafts are editable and need approval. |
| Fax | A test fax to the client's chosen number arrives with the right page count and order. A failed send shows clearly and can be retried. |
| Handoff | Lorraine runs one complete flow (packet, cover, RFS, approve, fax) unaided. Handoff document delivered and reviewed. |

### Timeline
| Week of | Client sees | Team works on |
|---|---|---|
| 2026-09-21 | Main screens (by Thu 2026-09-24) | M0, M2, M3 · Claude Design passes 1–2 |
| 2026-09-28 | Sample packets on all 20 templates — **Milestone 1** | M4 · design pass 3 · M1 |
| 2026-10-05 | Everyday screens, cover, RFS, three files — **Milestone 2** | M5, M6, M7 |
| 2026-10-12 | Grok, fax, test faxes — **Milestone 3** | M8, M9, M10, M11 tests |
| 2026-10-19 | Install on the Mac + walkthrough call | M11 install, handoff; 14-day bug-fix window starts |

The client reviews each milestone in 1–2 business days. The budget is tight: keep low-traffic screens (patient history, activity log, correct-and-resend) simple and spend effort on S9, S11–S14 and the PDF engine.

## 04 · What the client has decided

Confirmed in writing by Lorraine (2026-09-19). **These override anything older, including the original Scope of Work.**

| Topic | Decision |
|---|---|
| Left / right | Never print the side in the diagnosis title. "Left/Right" is only in template names so staff pick the right one. The side is shown on the page in the Post Treatment "Location of pain" area, which is already on each template. |
| Post-treatment severity | Always 1/10 on every diagnosis (bottom-left, Post Treatment), whatever the scan shows. |
| Last page | "Excellent" ticked under today's response on the final visit, for every diagnosis (covers the pre-printed "Good"). |
| Who uses it | Lorraine only prepares and approves. Dr. DeYoe takes no part. Roles exist for future staff. |
| Blairsville | Every send to Blairsville VA also goes to Atlanta VAMC — automatically. |
| Hosting | Office Mac desktop. **SQLite**, FileVault, nightly encrypted backups. *(Build 29 Sep, QA H5: `ops/backup.sh` backs up the database, the PDFs as sent (`backend/data/documents`) and `backend/config` to the external encrypted drive (`BACKUP_DIR`, refused on the Mac's own disk), checks each copy, keeps 30 days.)* |
| Fax files | Three separate PDFs per fax: cover sheet, clinical notes, RFS. **Never merged** — the VA rejects bundled files. (This replaces "one merged PDF" in the original Scope of Work.) |
| Notes file name | `{First} {Last} {last4} Acupuncture Records {VA number}.pdf` — e.g. `John Doe 1234 Acupuncture Records VA0012345678.pdf` |
| Fax provider | SRFax. Sending number **404-738-1714** (the supplied forms still show the old 706-664-0412 — always print the setting, not the form's printed number). *(Build 30 Sep: one SRFax fax per destination with the three files attached separately, in order — cover, notes, RFS. Whether the VA needs three separate transmissions instead is for Poonam to confirm.)* |
| Diagnoses | 20 total. PTSD and Brain Stem Stroke removed; Headache and Cervicalgia – Left added; two replacements still to come. *(Update: the replacements are Vertebrogenic Low Back Pain – Right and Fibromyalgia (22 Sep); all 20 forms in hand and set up, 28 Sep.)* |
| Repository | Private GitHub repo under Lorraine Fordham's account; team as collaborators. *(Update 22 Sep: the practice account DeYoeWellness714, with 2FA.)* |

### Decided by us (internal — build it this way)

| # | Decision |
|---|---|
| A | **Forms are configuration.** One master form definition + one settings file per diagnosis (defaults, hidden options, print positions, cover-ups). Our team adds and maintains these files. No on-screen editor. |
| B | **Patient answers are recorded per visit** — type of pain, painful activities, daily-life interference — with automatic pre-fill (rules in §05). |
| C | **Multi-clinic ready, single-clinic deployed.** tenant = clinic, office = branch. DeYoe is the only clinic. |
| D | **Last four only.** No field for a full SSN, anywhere. |
| E | **Everything is logged** — views, changes, sign-ins, downloads, settings, system events. |

## 05 · Business rules

Every rule the code must enforce. Tests should cover each row.

### Progress-note pages

| Rule | Detail |
|---|---|
| One page per visit | On the diagnosis's current template version. |
| First page (visit 1) | Label `INITIAL VISIT IN AUTHORIZATION` + the first-visit physician note. |
| Last page (visit marked final) | Label `Last Visit in authorization` + closing note (default "Pt responded well to tx. Pt had an excellent outcome w/ acupuncture") + "Excellent" ticked. |
| Middle pages | Only date, pain level and the patient's answers change. |
| Date | Stored once per visit; printed at the top and bottom. They can never differ. |
| Pain level | 0–10, entered every visit. The tool shows last visit's value as a hint but never pre-selects or generates it. |
| Post-treatment | Always 1/10. |
| Printed on every page | Patient name, VA number, last four, date (×2), pain level, the patient's answers (ticks), any cover-ups from the template settings. |
| Never changed | Clinical sections (objective, pulse/tongue, assessment, prognosis, plan, treatments, point prescription, therapies) and Dr. DeYoe's signature stay exactly as scanned. *(Build now, awaiting Poonam/Shikha: S9 shows these sections pre-filled from the form and editable per visit. Since 29 Sep Dr. DeYoe signs every visit on S9 — required to save — and the signature prints beside the visit date on that page.)* |
| Pre-printed first-visit wording | On Anosmia and Vertebrogenic, cover it on every page except the first (default; awaiting confirmation). |
| Missing top pain value | Shoulder (both), Dorsalgia – Right, Cervicalgia – Right, Low Back Pain – Right: print it in the standard position used by the other templates (default). |
| Title wording | Print the diagnosis title as on its template, without the side. Where typed text overlaps form lines (Dorsalgia – Left, Vertebrogenic), cover and re-type it cleanly. |

### Patient answers — pre-fill

1. *(cut off by a page break in the source screenshot — not captured; ask for the exact wording of rule 1 if it matters for implementation)*
2. Same patient and same diagnosis as an earlier authorization: copy that patient's last recorded visit under the earlier authorization. Drops answers for options hidden in the current template version. *(exact wording partly obscured by the screenshot's page break — confirm before relying on it precisely)*
3. Visit 1 otherwise: the diagnosis's default ticks from its settings file.
4. Dr. DeYoe's note (Grok) can replace any of the above after Lorraine accepts it.
5. Every answer is editable. An edit carries forward to later visits that haven't been sent. `answer_source` records `default` · `previous_visit` · `previous_authorization` · `doctor_note` · `changed`. *(Build 29 Sep, QA H3: a saved visit is corrected on S9 in edit mode — Edit → on every S7 visit row, `PUT /api/authorizations/{id}/visits/{visitId}` — with the same rules re-checked and a new signature; answers later visits carried forward unchanged follow the edit. Locked once a packet for the authorization is approved.)*
6. "Same as last visit" copies everything in one click.

### Authorization and visit limits

| Rule | Behaviour |
|---|---|
| Visit count ≤ visits approved | Hard block in the service, the API (409), and a SQLite trigger. "Add visit" disabled with the reason. |
| Visit date inside the authorization period | Hard block. Period start and end are required. |
| Dates | Required, unique within the authorization, after the previous visit. |
| Nearing the limit | Warning at 2 visits left, on S7 and the dashboard. |
| Reducing approved visits or the period | Refused if it would exclude a recorded visit (name the visit). |
| Extension | Edit approved visits or end date on S8 with a short reason, logged. Default assumption: same VA number. |
| Final visit | `is_final` set automatically at the last approved visit; can be set earlier if treatment ends; at most one; no visits after it. Drives the last-page rules. *(Build 29 Sep, QA H2: a packet can't be built or approved until a visit is final.)* |
| Diagnosis | Locked once visits exist. A different diagnosis = a new authorization. |
| Patients | Possible duplicate = same last name + DOB + last four. Suggest, never merge automatically. |

## 08 · The 21 screens at a glance

Full fields and states are in the screen spec (not yet captured in this repo — ask for it when detailed screen work starts). This is the map.

| ID | Screen | Purpose | Module |
|---|---|---|---|
| S1 | Sign in | Email + password; lock-out; "Forgot password?" emails a one-time reset link *(added 28 Sep; also "Change password" in the top bar)*; idle sign-out notice *(idle sign-out switched off 28 Sep at the user's request — `IDLE_TIMEOUT_MINUTES=0`; to confirm with Poonam before go-live)* | M1 |
| S2 | Dashboard | Ready to send · failed/waiting faxes · nearing last visit · missing ICD-10 · in progress · recently sent | M10 |
| S3 | Patients | Search by name or last four | M5 |
| S4 | Add / edit patient | Identity; duplicate suggestion | M5 |
| S5 | Patient detail | All authorizations, pain trend (keep simple) | M5 |
| S6 | New authorization | VA number, diagnosis picker, facility, office, visits, period | M5 |
| S7 | Authorization detail | Visits, "7 of 12 approved", packets, faxes | M5 |
| S8 | Edit authorization | Fixes and extensions (with reason) | M5 |
| S9 | Visit entry | **Most used.** Date, pain, answers via FormRenderer, final-visit switch | M6 |
| S10 | AI assist | Paste doctor's note, accept proposals | M9 |
| S11 | Build packet | Kind, cover sheet, auto Atlanta, RFS toggle, page counts | M7 |
| S12 | Preview | Tab per file, per-page edit, consistency panel, download | M7 |
| S13 | Review and send | The one irreversible step; explicit confirmation | M8 |
| S14 | Send result | Per destination: queued / sent / failed / waiting too long | M8 |
| S15 | Fax log | Every transmission, retry, open files as sent | M8 |
| S16 | Diagnoses | Read-only list, versions, sample-page previews | M10 |
| S17 | — | Not built. Diagnoses are added by developers via config. | — |
| S18 | Cover sheets, RFS, destinations | Cover sheets, RFS versions, Blairsville → Atlanta rule | M10 |
| S19 | Practice and sending | Provider, NPI, phone, fax, SRFax (write-only), retention, backup status, test fax *(Build 1 Oct: VA facility fax numbers are editable here too — used from the next packet built)* | M10 |
| S20 | Users and activity log | Users/roles; log filtered by type | M10 |
| S21 | Correct and re-send | New packet linked to the sent one (keep simple) | M8 |

`FormRenderer` (S9) is the shared component that renders a diagnosis's master form definition + settings — this is `frontend/src/forms/`. SRFax credentials on S19 are write-only (entered, never displayed back).

## Part 4 · Build

*(Build 1 Oct: for manual testing, `backend/scripts/seed_test_data.py` loads 12 fake test scenarios through the API and `reset_test_data.py` removes exactly those — see README, "Test scenarios".)*

## 09 · Golden rules

Per the source doc: *"Put these in the repo's `CLAUDE.md` (section 15) so every Claude Code session follows them."* They're in [CLAUDE.md](../../CLAUDE.md) in this repo — reproduced here for reference:

1. **No real patient data, anywhere, until go-live.** Designs, prompts, fixtures, tests, the repo and review links use fake patients only — Jane Doe, John Doe. Never paste a real name, date of birth, SSN or VA number into Claude Design, Claude Code or Grok.
2. **We print on the practice's scans; we never redraw their forms.** Every output page is their scanned template with values stamped on top (reportlab + pypdf).
   *Changed 25 Sep (user decision, awaiting Poonam's confirmation under rule 8):* no clean scans, so the clinical notes, RFS and cover sheets are new PDFs generated from the practice's forms as HTML templates, filled from the saved records and printed with headless Chromium (`backend/app/pdf/`). All 20 diagnosis forms are set up (28 Sep). Signature areas stay blank.
3. **Forms come from configuration.** The visit form is rendered from the master form definition and the diagnosis settings. Never hard-code a diagnosis's checkboxes in a component.
4. **Every query is scoped to a clinic.** All clinic data goes through the tenant-scoped data layer. No raw query without `tenant_id`.
5. **Three files, never merged.** Cover sheet, clinical notes and RFS are always separate PDFs.
6. **Nothing is faxed without an explicit human confirmation**, and nothing sent is ever edited — corrections create a new packet.
7. **No full SSN.** Last four only. No identifiers in any Grok request; the exact prompt sent is stored.
8. **This handbook wins.** If a design, a document or a generated change disagrees with it, stop and raise it with Poonam — don't let a tool quietly change a rule.

---
*Annotated 28 Sep with build changes awaiting Poonam's confirmation (marked "changed" / "update"); see `changelog.html` v2.4–v2.5.*

*Transcribed from the project doc's "Developer handbook (main)" tab on 2026-09-22 (§§01–03 from screenshot 1, §§04–05 from screenshot 2, §§08–09 from screenshot 3). §06–07 (patient answers detail continuation, data validation?) and the Screen spec, Data model, Current process, SOW tabs still need to be captured when shared. §05 "Patient answers — pre-fill" rule 1 and part of rule 2 were cut off by a screenshot page break — confirm exact wording before treating as final.*
