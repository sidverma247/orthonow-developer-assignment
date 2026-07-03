<h1 align="center">OrthoNow — Developer Assignment</h1>

<p align="center">
  <strong>Consultation-booking funnel: analytics schema, high-converting landing page & CRM integration design</strong>
</p>

<p align="center">
  <img alt="Task 1" src="https://img.shields.io/badge/Task_1-GTM_%2F_GA4_Schema-0b3d91" />
  <img alt="Task 2" src="https://img.shields.io/badge/Task_2-Landing_Page-1a73e8" />
  <img alt="Task 3" src="https://img.shields.io/badge/Task_3-Integration_Design-128a5b" />
  <img alt="Stack" src="https://img.shields.io/badge/stack-HTML5_%C2%B7_CSS3_%C2%B7_Vanilla_JS-555" />
  <img alt="Lighthouse" src="https://img.shields.io/badge/Lighthouse_Mobile-90%2B_target-brightgreen" />
  <img alt="License" src="https://img.shields.io/badge/license-MIT-black" />
</p>

---

> **🔗 Live demo:** https://sidverma247.github.io/orthonow-developer-assignment/ &nbsp;·&nbsp;
> **Repo:** https://github.com/sidverma247/orthonow-developer-assignment
> > **Loom Video:** https://www.loom.com/share/e23fcd286eb14a2c9f1815eccb343ccf

## Project overview

**OrthoNow** is a (fictional, for-assignment) multi-city orthopaedic clinic group running a
performance-marketing campaign — *"Get an expert orthopaedic opinion — book your consultation at
OrthoNow"* — targeting working professionals (28–50) in **Bengaluru, Hyderabad and Chennai** who
have knee, back, neck or sports-injury pain. The single business goal is **generating consultation
bookings**.

This repository is my end-to-end answer to that brief, split into the three deliverables the
assignment asks for:

| # | Deliverable | What it proves |
|---|-------------|----------------|
| **01** | **GTM / GA4 event schema** | I can design a measurable funnel — events, triggers, parameters, GA4 mapping, Google Ads conversion selection — that a marketing team can actually act on. |
| **02** | **Landing page** | I can ship a fast, accessible, high-converting page with real `dataLayer` tracking in plain HTML/CSS/JS — no framework crutches. |
| **03** | **Integration architecture** | I can design the server-side path from a lead to HubSpot CRM, WhatsApp (Karix) and Google Ads, including the failure modes that break real systems. |

Everything here runs as-is: open `task-02-landing-page/index.html` in a browser, submit the form,
and watch `window.dataLayer` populate in the console.

---

## Repository structure

```text
orthonow-developer-assignment/
│
├── README.md                         ← you are here
├── LICENSE                           ← MIT
├── .gitignore
│
├── docs/                             ← cross-cutting project docs
│   ├── assignment-overview.md        ← the brief, restated + how each task maps to it
│   ├── pagespeed-notes.md            ← Lighthouse strategy & expected scores
│   ├── loom-script.md                ← minute-by-minute walkthrough script (≤8 min)
│   └── testing-guide.md              ← how to test the page, validation, GTM & Lighthouse
│
├── task-01-gtm-schema/               ← TASK 1: measurement plan
│   └── GTM_Event_Schema.md           ← full event table (13 events) + 3-step funnel
│                                        drop-off (real JSON) + Google Ads conversion
│
├── task-02-landing-page/             ← TASK 2: the page
│   ├── index.html                    ← ONE self-contained file (HTML+CSS+JS)
│   ├── README.md                     ← design & tracking rationale
│   └── assets/                       ← desktop/mobile/pagespeed screenshots
│
├── task-03-integration-design/       ← TASK 3: server-side architecture
│   ├── README.md
│   ├── README_Task_03_Integration_Design.md  ← the integration design write-up
│   └── architecture-diagram.png      ← the data-flow diagram
│
├── assets/
│   └── screenshots/                  ← shared screenshots referenced by the README
│
└── .github/
    └── workflows/
        └── lint.yml                  ← HTML validation on push (CI)
```

**Why this layout?** Each task is a self-contained folder with its own `README`, so a reviewer can
open any one of them and understand it in isolation. Cross-cutting material (testing, Lighthouse,
the Loom script) lives in `docs/`. Binary assets are kept out of the task folders' prose so the
Markdown stays readable.

---

## Technology used

| Layer | Choice | Why |
|-------|--------|-----|
| Markup | **HTML5, semantic** | Landmarks (`header`/`main`/`section`/`footer`), native `<details>` FAQ, and a `<form>` with real labels → free accessibility + SEO. |
| Styling | **CSS3, custom properties** | Design tokens in `:root`, mobile-first, zero external CSS. |
| Behaviour | **Vanilla JavaScript** | ~90 lines, no dependencies. Validation + one `dataLayer.push()`. |
| Fonts | **System font stack** | No web-font download → faster first paint, no layout shift. |
| Analytics | **Google Tag Manager → GA4 → Google Ads** | Documented in Task 1; the page emits the `dataLayer` event GTM consumes. |
| CRM (design) | **HubSpot CRM API + Karix (WhatsApp)** | Documented in Task 3. |
| CI | **GitHub Actions** | Lints/validates `index.html` on every push. |

**No** frameworks, bundlers, CDNs, external fonts or images are used anywhere in the landing page.

---

## How to run locally

The landing page is a single static file — no build step, no server required.

```bash
# 1. Clone
git clone https://github.com/sidverma247/orthonow-developer-assignment.git
cd orthonow-developer-assignment

# 2a. Simplest: open the file directly
open task-02-landing-page/index.html        # macOS
# xdg-open task-02-landing-page/index.html   # Linux
# start task-02-landing-page/index.html      # Windows

# 2b. Or serve it (closer to production; needed for a clean Lighthouse run)
cd task-02-landing-page
python3 -m http.server 8080
# then visit http://localhost:8080
```

To see the analytics event fire: open DevTools → **Console**, fill the form with a valid
10-digit mobile, submit, and inspect the logged `window.dataLayer` array.

Full step-by-step (including a Lighthouse run) is in [`docs/testing-guide.md`](docs/testing-guide.md).

---

## Assignment overview

The campaign's job is to turn paid clicks into booked consultations and to **measure every step** so
marketing can optimise spend. My three deliverables cover the full loop:

1. **Measure it** — a GTM event schema that captures the booking funnel plus every secondary
   conversion (call, WhatsApp, patient guide, blog engagement), mapped to GA4 events, reports,
   audiences and the one Google Ads conversion worth importing.
2. **Convert it** — a landing page engineered for the 28–50 professional on a phone: one primary
   CTA, friction-light three-field form, trust signals, and honest CRO copy.
3. **Integrate it** — a resilient server-side path that gets each lead into HubSpot without
   creating duplicates, triggers a WhatsApp confirmation inside a 2-minute SLA, and feeds Google
   Ads a clean conversion signal.

See [`docs/assignment-overview.md`](docs/assignment-overview.md) for the brief restated in full.

---

## Screenshots

> Place exported images in the paths below and they will render here.

| Desktop | Mobile | Lighthouse (mobile) |
|---------|--------|---------------------|
| ![Desktop preview](task-02-landing-page/assets/desktop-preview.png) | ![Mobile preview](task-02-landing-page/assets/mobile-preview.png) | ![PageSpeed mobile](task-02-landing-page/assets/pagespeed-mobile.png) |

---

## Loom walkthrough

A ≤8-minute video walking through the repo, a live demo of the form + `dataLayer`, and the
integration design.

**▶ Loom link:** _paste your recorded Loom URL here_
**▶ Live page to demo in the video:** https://sidverma247.github.io/orthonow-developer-assignment/

The full minute-by-minute script (and likely interviewer questions with answers) lives in
[`docs/loom-script.md`](docs/loom-script.md).

---

## Testing instructions

Quick version — full guide in [`docs/testing-guide.md`](docs/testing-guide.md):

1. Open `task-02-landing-page/index.html`.
2. Submit the form empty → three inline errors, no `alert()`, no reload.
3. Enter a bad phone (e.g. `12345`) → phone error only.
4. Enter valid data → form is replaced by the thank-you panel.
5. Open Console → `window.dataLayer` contains one `consultation_form_submitted` object.
6. Run Lighthouse (mobile) → Performance / Accessibility / Best Practices / SEO all ≥ 90.

---

## Future improvements

- Wire the front-end `dataLayer` push to a real GTM container ID and publish the GA4 + Ads tags.
- Add the multi-step booking flow (Task 1 documents its events) as a progressive-enhancement layer.
- Server endpoint implementing the Task 3 architecture (HubSpot search-by-phone → upsert → queue → Karix).
- Add automated a11y tests (axe-core) and a Lighthouse-CI budget to the GitHub Action.
- Consent-mode v2 gating so tags respect the visitor's cookie choice.

---

## Author

**Siddharth** — Frontend / Analytics developer
GitHub: [@sidverma247](https://github.com/sidverma247)
Built for the OrthoNow / Namoza developer assignment.

---

## License

Released under the [MIT License](LICENSE).
