# PageSpeed / Lighthouse Notes

**Target:** ≥ 90 on all four Lighthouse categories, **mobile**, for `task-02-landing-page/index.html`.

## How to run

```bash
cd task-02-landing-page
python3 -m http.server 8080
# Chrome → DevTools → Lighthouse → Mobile → Analyze page load
# (serve over http:// rather than file:// so Best Practices scores accurately)
```

Or headless:

```bash
npx lighthouse http://localhost:8080 \
  --preset=perf --form-factor=mobile --screenElementTimeout=5000 \
  --output=html --output-path=./lighthouse-report.html
```

## Why the page scores well

### Performance
- **No external requests.** HTML, CSS and JS are inlined in one file; no fonts, images, or third-party scripts. The browser makes essentially one request → very fast **LCP** and **FCP**.
- **System font stack.** Zero font download, so no FOUT/FOIT and no font-driven **CLS**.
- **No layout shift.** No lazy images popping in, no injected banners; the sticky bars reserve their own space (mobile `body` has bottom padding).
- **Tiny main-thread cost.** ~90 lines of dependency-free JS, no hydration, no framework runtime → low **Total Blocking Time**.
- **CSS-drawn UI.** The logo mark, icon chips and success tick are CSS/emoji, not image files.

### Accessibility
- Semantic landmarks (`header`/`main`/`section`/`footer`), one `h1`, ordered headings.
- Every input has a `<label>`; errors use `role="alert"` + `aria-invalid`; hints via `aria-describedby`.
- Visible focus states, AA colour contrast, 48px tap targets, a skip link, and `prefers-reduced-motion` support.

### Best Practices
- HTTPS-ready, no console errors, no deprecated APIs, valid HTML5, `theme-color` set, no mixed content.

### SEO
- Descriptive `<title>` + meta description, `lang="en"`, canonical, Open Graph, mobile viewport, and JSON-LD `MedicalClinic` structured data. Content is crawlable text (no JS-rendered copy).

## Expected results (representative)

| Category | Expected mobile score |
|----------|-----------------------|
| Performance | 97–100 |
| Accessibility | 96–100 |
| Best Practices | 96–100 |
| SEO | 100 |

> Scores vary slightly by machine/throttling. Serving over `http://localhost` (not `file://`) and
> running in an incognito window with extensions disabled gives the most representative numbers.
> Save the exported report screenshot to `../task-02-landing-page/assets/pagespeed-mobile.png`.
