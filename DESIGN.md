# pantry: design

Find the subscriptions quietly draining your account. Drop in a bank statement, get the list, the yearly cost and a way to cancel. Runs in the browser. Nothing is uploaded.

Status: design only. Nothing is built yet.
Last updated: 2026-10-29.
Owner: tuftle (tuftle@proton.me). Part of the Tuftle family (wren, nook, burrow).
Working name: **pantry** (you go and look at what has gone stale at the back). Alternatives if the name is taken: larder, cellar. The owner decides before anything is built (section 16).

---

## 0. Start here (for the session that builds this)

Read this whole file before writing code. Then:

1. Work through the milestones in section 13, in order. Stop at the end of each one, run the tests, and report using the format in section 17. Wait for the owner to say "go" before the next milestone.
2. Never deploy, publish, post, buy anything or touch any repository except this one. The owner deploys.
3. Never ask for, accept or handle real bank data. Use only the synthetic fixtures described in section 11. The owner tests real statements on their own computer.
4. Commit as `tuftle <tuftle@proton.me>`. No co-author lines. Work on a branch called `build` and open a pull request to `main`. The owner merges.
5. House rules for all copy, code comments, commit messages and docs: lowercase and calm in the product, no em dashes, no buzzwords, no emojis. Plain words.
6. No runtime dependencies. Vanilla ES modules, no framework, no bundler, no CDN. Dev tools only if they are worth it, and pinned to exact versions.
7. Budget: the owner has limited cloud credit. Keep milestones small, keep tests fast, do not run long idle jobs, and only start a browser for the runtime privacy test in milestone 5.

If this file and the owner disagree, the owner wins. If something here is wrong or unclear, say so in your report instead of guessing.

---

## 1. What it is

A single web page. The visitor drops one or more bank or card statement files (CSV). In a few seconds, pantry shows:

- every recurring charge it found, grouped by how sure it is
- what each costs per month and per year, and the total
- price rises ("went from 13.99 to 17.99 in March")
- trials that quietly turned into paid plans
- what renews in the next 30 days
- where to cancel each one
- a share card with totals only

All of it is computed in the browser. There are no accounts, no bank connections, no uploads, no analytics.

## 2. The problem

Almost everyone pays for something they forgot: a trial that converted, a service they stopped using, a plan that crept up in price. Existing tools either need a bank connection and an account (and sell the data or a subscription), or are spreadsheets. People do not trust a website with a statement. "It never leaves your computer, and you can check" is the whole pitch, so the design treats verifiable privacy as a feature, not a footnote.

## 3. Goals, non-goals, anti-goals

Goals
- A useful answer within 10 seconds of dropping a file, on messy real statements.
- Verifiably local: the page cannot send data anywhere, and a visitor can confirm that.
- Honest about uncertainty: say "probably" and show why.
- Calm, finished design in the Tuftle style.
- Real actions: cancel links, a renewals calendar file, a list to take away.
- A share card that reveals nothing private unless the visitor chooses.

Non-goals (not in version 1)
- Bank connections (Plaid and similar), accounts, sync, a mobile app.
- A budgeting tool. pantry only looks at recurring charges.
- PDF statements. (Asked for often. Possible later with pdf.js. Out of scope now.)
- Currency conversion. No network means no rates. Amounts stay in the statement's currency.
- Advice. pantry says "worth a look", never "you should cancel".

Anti-goals (would make it a failure even if it works)
- Any network request that carries user data, including by accident.
- Claiming certainty it does not have ("this is a scam", "you are wasting money").
- Upsells, tracking, dark patterns, newsletters, sign-up walls.
- A catalogue of cancel links that quietly rots.

## 4. The visitor's journey

One page, five states. Copy below is a starting point.

1. **landing.** Headline: "find the subscriptions quietly draining your account." Sub: "drop in a bank statement. pantry finds what repeats, what it costs a year, and where to cancel. it all happens in this page. nothing is uploaded." A big drop zone: "drop a statement here, or choose a file". Under it: "try it with example data" (loads `sample/example-statement.csv`). A small line: "works offline. try it with wi-fi off." and a link "how do i know?" (section 10).
2. **check the columns.** After parsing, a compact preview of the first 8 rows with the guessed columns highlighted: date, description, amount. Three dropdowns to correct a wrong guess. One question only when truly ambiguous: "is 03/04/2026 the 3rd of april or the 4th of march?" and "are charges shown as negative numbers?" with a toggle. Button: "looks right".
3. **results.**
   - A summary band: "14 subscriptions, 1,326 a year" in large serif, with "(about 110 a month)" and the statement's date span.
   - Tabs or sections: **sure** (clear pattern), **probably**, **maybe**, **ended** (stopped recurring), **regular bills** (rent, phone, utilities, collapsed by default).
   - Each row: name, how often, amount, yearly cost, last charge, a small sparkline of past amounts. Badges: "price rose", "was a trial", "billed by apple" (or google, paypal).
   - Expanding a row shows the charge history, the reasons ("charged 9 times, about every 30 days, same amount"), the cancel help, and three buttons: **keep**, **worth a look**, **cancel**. Plus "not a subscription".
   - A running line at the bottom: "if you cancel the ones marked cancel: 312 a year."
4. **coming up.** The next 30 days of predicted renewals, with a button "add these to my calendar" (downloads an .ics file).
5. **share.** "make a card": a 4:5 image with the headline numbers only. Options: show names of the top three (off by default). Buttons: download image, copy text. Also "export my list" as CSV or JSON.

Always present: "forget everything" (clears memory and anything stored) and a footer with the privacy explanation.

## 5. Inputs

Version 1 reads CSV bank and card exports, one or many files at once (several months, several accounts). It merges them and removes duplicates when the same transaction appears in overlapping files.

Version 1.1 (after launch): OFX and QFX (a structured format many US banks offer), and QIF.

### 5.1 Reading the file

- **Decoding.** Check for a byte order mark (UTF-8, UTF-16 LE and BE). Otherwise try strict UTF-8. If that fails, decode as Windows-1252. Use `TextDecoder`.
- **Delimiter.** Try comma, semicolon, tab, pipe on the first 50 lines. Pick the one that gives the most consistent field count greater than 1.
- **Parser.** Write a small RFC 4180 state machine: quoted fields, doubled quotes, newlines inside quotes, CRLF and LF. No library.
- **Header row.** Many banks put account details above the table. Find the first row within the first 30 lines that has at least 3 text cells and is followed by rows where one column looks like a date. Everything above is preamble and is ignored. Rows with no valid date (totals, notes) are skipped and counted, and the count is shown ("skipped 3 rows without a date").

### 5.2 Columns

Score every column, then pick the best. Never trust a header name alone.

- **Date.** At least 90% of values parse as dates. Header hints: date, booked, posted, transaction date, datum, buchungstag, fecha, data, date de l'opération. If two date columns exist (transaction and posting), prefer the transaction date.
- **Amount.** At least 90% numeric. Three shapes must work:
  1. one signed column
  2. two columns, debit and credit, one empty per row (headers like debit, credit, money out, money in, paid out, paid in, withdrawal, deposit, soll, haben)
  3. an amount column plus a type column (DR/CR, debit/credit, D/C)
- **Description.** The text column or columns with the most variety and length. Header hints: description, details, memo, payee, narrative, merchant, name, reference, text, verwendungszweck, beschreibung, libelle. If several text columns look useful (payee plus memo), join them with a space.
- **Ignore.** Balance, running balance, category columns supplied by the bank (keep them as optional hints).

### 5.3 Dates, numbers, signs

- **Date formats.** ISO (2026-03-04), DD/MM/YYYY, MM/DD/YYYY, DD.MM.YYYY, D MMM YYYY, MMM D, YYYY, two digit years, and a time part after the date. Resolve DD/MM against MM/DD by scanning every value: if any first part is greater than 12 it is DMY; if any second part is greater than 12 it is MDY. If both stay ambiguous, use a hint (pound or euro symbol suggests DMY, dollar suggests MDY). If still unsure, ask once (section 4, step 2).
- **Numbers.** Work out the decimal separator by scanning: `1,234.56` against `1.234,56` against `1 234,56`. Accept currency symbols and codes, parentheses for negatives `(12.34)`, trailing minus `12.34-`, and a unicode minus.
- **Sign.** For one signed column, spending is whichever sign matches most rows, with a check: if almost every row is positive and the file has no large credits, positive means spending (some card issuers export that way). Always show the toggle.
- **Currency.** Take it from a column or the first symbol. If a file mixes currencies, group by currency and never add them together.

## 6. The pipeline

Every stage is a pure function with no access to the page, so it runs in Node tests and in a Web Worker. Data types are written in JSDoc in `core/types.js`.

### 6.1 Normalise the merchant

Goal: turn `NETFLIX.COM 866-579-7172 CA` and `Netflix Intl 3 Mar` into one key.

1. Fold to uppercase, strip accents, replace `* # / _ -` with spaces (keep `.` inside domains).
2. If the text contains `*`, the merchant is the part after it, and the part before is recorded as the **processor** (`SQ`, `TST`, `PP`, `PAYPAL`, `GOOGLE`, `APPLE.COM/BILL`, `AMZN MKTP`, `STRIPE`, `PADDLE`, `FS`). Processors matter: a charge billed by Apple or Google is cancelled in Apple or Google settings, not on the merchant's site.
3. Remove noise: runs of 3 or more digits, dates like `3/12`, masked card numbers (`XXXX1234`), reference tokens (six or more characters containing digits), phone numbers, url paths, trailing country or state codes.
4. Collapse spaces. The result is the **key**.
5. Match against the catalogue (section 7). A match gives a canonical `merchantId` and display name.
6. Unmatched keys are clustered: join keys with Jaro-Winkler similarity of 0.92 or more, or where one is a prefix of the other (at least 6 characters), but only when their amounts or cadences are compatible. The display name is the most common cleaned form, title-cased.

### 6.2 Find recurring charges

For each merchant cluster, sort by date and split into **amount bands** (a band holds charges within 3% or one currency unit of each other), because one merchant can have two plans.

For each band with at least 2 charges, compute the day gaps between consecutive charges and score them against these cadences:

| cadence | target days | tolerance |
|---|---|---|
| weekly | 7 | 2 |
| every 2 weeks | 14 | 3 |
| every 4 weeks | 28 | 3 |
| monthly | 30.4 | 4 (also check the day of month stays within 3 days, or sticks to month end) |
| every 2 months | 61 | 6 |
| quarterly | 91 | 8 |
| twice a year | 182 | 12 |
| yearly | 365 | 14 |

A gap of about twice the cadence counts as one skipped period (a penalty, not a failure). Weekend and bank holiday shifts are covered by the tolerance. A cadence is accepted when at least 75% of gaps fit.

Minimum evidence: weekly 4 charges; every 2 and 4 weeks 3; monthly 3 (or 2 if the catalogue matches); quarterly and twice a year 2 with a catalogue match; yearly 2 charges when the data spans more than 13 months, otherwise a single charge counts only as "possibly yearly" when the catalogue matches or the text says annual, yearly, renewal.

Not subscriptions (filter out): merchants with 8 or more charges in any 30 days (coffee, groceries) unless the catalogue says otherwise; amounts that vary more than 25% (coefficient of variation) without a catalogue match; transfers and cash withdrawals; known non-subscription categories by keyword.

**Regular bills** (rent, phone, utilities, insurance): recurring with 3% to 25% variation or flagged by keyword. Listed separately, collapsed.

**Ended:** the last charge is more than 1.5 cadences before the end of the data. Shown in its own section, with the date it stopped. ("you stopped paying hulu in june.")

**Same-day duplicates:** two equal charges on one day from one merchant are shown as "2 at once" (two seats, or a double charge) and never silently merged.

### 6.3 Price changes

For a recurring series, find a change point where the amount differs by 3% or more and the new amount holds for at least 2 charges (or is the latest charge, marked "just changed"). Report old amount, new amount, percent, date, and the yearly effect. Ignore small random variation (foreign exchange noise, tax rounding) under 3%.

### 6.4 Trials that converted

A first charge of at most 1.50 (or at most 15% of the next charge) followed within 7 to 35 days by a higher charge from the same merchant: "started as a trial on <date>, became <amount> a <cadence>". If only the small charge exists in the last 35 days and the catalogue knows the normal price: "looks like a trial. usually becomes <price> on about <date>."

### 6.5 Overlaps

If two or more active subscriptions share a catalogue category (music, video, cloud storage, news, vpn, antivirus, meditation, ebooks), add a quiet note: "you pay for 3 video services." No recommendation.

### 6.6 Confidence

Score from 0 to 1 using: number of charges, how regular the gaps are, how stable the amount is, a catalogue match, and subscription words in the text (subscription, membership, premium, plus, pro, monthly). Tiers: **sure** at 0.85 or more, **probably** at 0.6 to 0.85, **maybe** at 0.4 to 0.6. Below 0.4 is hidden. Each item stores its reasons as short strings shown in the expanded row.

### 6.7 Money maths

- Per year = amount times periods per year. Per month = per year divided by 12.
- Total counts **active** items in sure and probably. "maybe" is shown separately and left out of the headline unless the visitor includes it.
- Cost so far = sum of charges inside the statement span, per item.

### 6.8 What is coming up

Predict the next charge date from the last one and the cadence (for monthly, use the stable day of month). List renewals in the next 30 days. The .ics file has one all-day event per renewal, dated 3 days earlier with the text "check whether you still want <name>". Write RFC 5545 correctly (CRLF, line folding at 75 octets, stable UIDs made from a hash of key and date).

### 6.9 Decisions

Per item: keep, worth a look, cancel, not a subscription. Stored in memory. Optionally "remember on this device" saves decisions (keys and choices only, no amounts) in `localStorage`. Off by default. "forget everything" removes it.

## 7. The catalogue

`site/src/data/catalogue.json`: about 300 popular services to start. It powers name matching, categories, cancel help, typical prices and trial lengths.

```json
{
  "id": "netflix",
  "name": "Netflix",
  "category": "video",
  "aliases": ["NETFLIX(\\.COM)?", "NETFLIX\\s+INTL"],
  "billedVia": ["direct", "apple", "google", "paypal"],
  "regions": ["US", "GB", "EU"],
  "cancel": {
    "kind": "online",
    "url": "https://www.netflix.com/cancelplan",
    "note": "sign in, then choose cancel membership",
    "lastChecked": "2026-11-01"
  },
  "typical": [{ "region": "US", "plan": "standard", "amount": 17.99, "per": "month", "asOf": "2026-10" }],
  "trial": { "days": 0 },
  "sources": ["https://help.netflix.com/en/node/407"]
}
```

The example values above are placeholders from memory and must be verified. Treat every URL and price as unverified until checked.

Rules
- **Processor-aware cancel help.** If the charge is billed by Apple or Google, show the Apple or Google subscription steps instead of the merchant's page. This is one of pantry's most useful details.
- **Prefer stable help-centre links** over deep links that rot. Each entry has `lastChecked`. The UI shows "link checked <date>" and a "report a broken link" button that opens a prefilled GitHub issue containing no user data.
- **Never fetch at runtime.** The catalogue ships with the page.
- **`scripts/check-links.mjs`** (run by the owner locally, needs network) sends HEAD requests to every URL, reports failures and updates `lastChecked`. The cloud session may write it but must not run it against the live web unless asked.
- **Honest labels.** Entries the agent wrote from memory have `"verified": false` and the UI says "not checked recently" for those.
- **Regions.** Start with US, GB and EU services. Each entry lists regions. Add more through pull requests.
- **Licence.** Release the catalogue as CC0 so others can reuse and improve it. This is an invitation to contribute.

## 8. Architecture and files

Static site, vanilla ES modules, no build step for the page itself. All logic in DOM-free modules so Node can test it.

```
pantry/
  DESIGN.md                 this file
  README.md                 short, in Tuftle style (see section 18)
  package.json              type: module, scripts: test, privacy-scan, make-fixtures, build-single
  vercel.json               headers only (section 10); no functions
  site/                     the only thing that is deployed
    index.html
    assets/style.css
    assets/logo.svg
    src/
      app.js                wires the UI to the worker
      worker.js             runs the pipeline off the main thread
      ui/                   drop.js columns.js results.js upcoming.js share.js proof.js
      core/
        types.js            JSDoc typedefs
        decode.js csv.js columns.js dates.js money.js
        normalise.js cluster.js recurrence.js prices.js trials.js overlaps.js
        confidence.js insights.js decisions.js ics.js redact.js
      card/card.js          canvas share card, no DOM beyond canvas
      data/catalogue.json
    sample/example-statement.csv
  tests/
    unit/                   one file per core module
    gen/                    fixture generator and ground truth
    fixtures/               generated, committed so runs are repeatable
    privacy/                static scan and runtime test
  scripts/
    make-fixtures.mjs privacy-scan.mjs check-links.mjs build-single.mjs
  .github/ISSUE_TEMPLATE/   broken-link.md  format-problem.md
```

The worker takes `File[]` (or text) and returns a plain result object. The main thread never parses or analyses, so the page stays responsive on large statements.

`scripts/build-single.mjs` produces `dist/pantry.html`: the whole app inlined into one file. Offer it as a download ("save this page and open it offline"). It is a strong trust signal.

## 9. Design

Follows the Tuftle look. Reuse these exact tokens.

| token | light | dark | use |
|---|---|---|---|
| `--bg` | `#f4efe6` | `#211c17` | page |
| `--card` | `#fbf8f2` | `#2a231c` | cards, panels |
| `--text` | `#3b2f25` | `#e6dccd` | body |
| `--soft` | `#8a7b6b` | `#9a8b7a` | secondary text |
| `--link` | `#8a4b2a` | `#d9a073` | links, accents, buttons |
| `--line` | `#ddd2c1` | `#3a3128` | borders |
| `--glow` | `rgba(224,168,112,.3)` | `rgba(217,160,115,.14)` | focus, hover |
| `--leaf` | `#7a9a5a` | `#7a9a5a` | small green accent (savings) |

- Dark mode follows `prefers-color-scheme`. Add `<meta name="darkreader-lock">` and `color-scheme: light dark`.
- Type: headings in Georgia with fallbacks `"Iowan Old Style", "Palatino Linotype", Palatino, serif`. Body in the system sans. No web fonts.
- Lowercase, calm copy. One accent colour. Soft borders. No gradients beyond a faint glow behind the headline.
- The headline numbers are large serif. Savings use `--leaf`. Warnings (price rise) use the link colour, never red.
- Motion: gentle count-up on the summary, nothing else. Respect `prefers-reduced-motion`.
- Accessibility: keyboard reachable drop zone (also a real file input), visible focus, contrast at least 4.5:1, labels on every control, announce results with `aria-live`, no information carried by colour alone. Works at 360 px wide.
- The share card: 1080 by 1350, cream background, big serif numbers, one line "found with pantry. it ran in my browser and nothing was uploaded." No service logos (trademarks), no names unless the visitor opts in.

## 10. Privacy and security design

This is the product. Treat each item as a requirement with a test.

1. **Content-Security-Policy**, sent as a header and repeated in a `<meta>` tag:
   `default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data: blob:; font-src 'self'; connect-src 'none'; worker-src 'self'; manifest-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`.
   No inline scripts or styles, so the policy can stay strict. `connect-src 'none'` means even a bug cannot send a request.
2. Other headers: `Referrer-Policy: no-referrer`, `X-Content-Type-Options: nosniff`, `Permissions-Policy` denying every feature, `Cross-Origin-Opener-Policy: same-origin`.
3. **No third parties.** No analytics, fonts, CDNs, error reporters or social buttons.
4. **No storage by default.** No cookies. `localStorage` only for the optional remembered decisions.
5. **Proof panel** ("how do i know?"): shows the policy, a live counter of network requests since the file was dropped (a `PerformanceObserver` on resources, with `fetch`, `XMLHttpRequest`, `WebSocket` and `sendBeacon` wrapped to throw), and plain steps to verify: turn wi-fi off and try it, open the developer tools network tab, or save the page and run it offline. Be honest in the text: the page itself was downloaded once, and the counter covers everything after.
6. **Redact helper for bug reports.** A button "copy a safe sample of my file's shape" produces the header row plus 5 data rows with every letter replaced by `a` or `A` and every digit by `9`, keeping punctuation and lengths. It is shown in a box to copy by hand. It is never sent anywhere.
7. **Supply chain.** Zero runtime dependencies and no lockfile to poison. Dev tools, if any, pinned to exact versions.
8. **Tests** enforce items 1, 3 and 5 (section 11).

## 11. Testing

The agent never sees real statements. Everything below uses synthetic data made by a deterministic generator.

### 11.1 Fixture generator (`tests/gen/`)

A seeded pseudo-random generator that writes statements and a `truth.json` listing the exact subscriptions inside each.

- **Personas:** student, family, freelancer, retiree; 12 and 24 month spans.
- **Bank layouts (at least 14):** single signed amount; debit and credit columns; amount plus DR/CR type; semicolon delimited European; tab delimited; preamble lines above the header; footer totals; BOM and UTF-16; Windows-1252 accents; DMY and MDY dates; thousands separators in both styles; parentheses negatives; positive-means-spending card export.
- **Subscriptions to plant:** monthly, weekly, every 4 weeks, quarterly, yearly; a price rise; a skipped month; a weekend shift; a trial that converts; a cancelled service; two plans at one merchant (for example several things billed by apple.com/bill at different amounts); a double charge on one day; a refund; foreign-exchange noise on one subscription; messy merchant strings like `NETFLIX.COM 866-579-7172 CA`, `AMZN Mktp US*2K4TT3`, `PAYPAL *SPOTIFYAB`, `GOOGLE *YouTubePremium`.
- **Noise to plant:** groceries, fuel, coffee several times a week, rent, salary, transfers.

### 11.2 What must pass

- **Unit tests** per core module: csv edge cases, date and number parsing, column detection, normalisation, clustering, cadence scoring, price change detection, trial detection, ics output.
- **Accuracy on the generated set:** recall at least 0.95 and precision at least 0.90 for subscriptions overall; precision at least 0.98 in the "sure" tier; cadence correct for at least 95% of found items; price change recall at least 0.9; fewer than 5 false positives per 1000 non-subscription transactions.
- **Property tests:** shuffling row order changes nothing; changing delimiter changes nothing; loading the same file twice changes nothing (deduplication); flipping the sign toggle and the data together changes nothing; merging two overlapping files equals the single combined file.
- **Privacy tests (`tests/privacy/`):**
  - *static scan* (`scripts/privacy-scan.mjs`): fail if anything under `site/` contains `fetch(`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `sendBeacon`, a dynamic `import(` of a URL, a `<script>`, `<link>` or `@import` pointing at `http`, other than plain anchors the visitor clicks.
  - *runtime:* load the page in headless Chrome, record every request, drop the sample file, and assert zero requests after load and a correct result.
  - *header check:* `vercel.json` carries the full policy and it equals the `<meta>` policy.
- **Accessibility:** automated checks (axe) in the runtime test and a manual keyboard pass.
- **Performance budget:** a 50,000 row statement parses and analyses in under 1.5 seconds on a mid-range laptop, with no main-thread task over 50 ms.

### 11.3 Real statements (owner only)

The owner runs pantry on their own statements and reports problems using the redact helper's output, never the file. Real-world format failures become new fixtures.

## 12. Deployment

- A Vercel static project named `pantry`, serving `site/`, under the owner's account. `vercel.json` sets the headers in section 10 and clean URLs. No functions, no environment variables, no secrets.
- Domain later: the owner plans tuftle.com. Until then `pantry.vercel.app` or `tuftle.vercel.app/pantry` (a folder in the tuftle site repo) are both fine.
- GitHub repository `Tuftle/pantry`, public, MPL 2.0 for the code like the other Tuftle projects, CC0 for `catalogue.json`. Licence choice is the owner's.
- The agent prepares everything and stops. The owner runs `vercel deploy --prod --yes`.

## 13. Milestones and acceptance

Each milestone ends with tests passing and a report.

- **M0 scaffold.** Repo layout, package.json, test runner (`node --test`), CI-free local scripts, house-rule linter (a tiny script that fails on em dashes, emojis and a list of banned buzzwords in `site/`, docs and commit messages). Accept: `npm test` runs and passes an empty suite; linter works.
- **M1 reading files.** decode, csv, columns, dates, money. Fixture generator for layouts. Accept: every generated layout is read to the correct rows and columns; ambiguity prompts appear only when truly ambiguous.
- **M2 finding subscriptions.** normalise, cluster, recurrence, prices, trials, overlaps, confidence. Accept: the accuracy targets in 11.2 on the generated set.
- **M3 catalogue and insights.** About 300 catalogue entries (unverified flag), processor-aware cancel help, totals, renewals, ics, decisions. `check-links.mjs` written but not run. Accept: catalogue schema test; ics validates against a reference parser in tests; matching works on all planted messy strings.
- **M4 the page.** Drop zone, column check, results, upcoming, decisions, share card, exports, "forget everything". Dark mode, mobile width, keyboard. Accept: manual walkthrough script in the report with screenshots from headless Chrome.
- **M5 trust.** CSP header and meta, proof panel, redact helper, privacy scan, runtime zero-request test, `build-single.mjs`. Accept: privacy tests pass; the single file works from `file://`.
- **M6 hardening.** Property tests, performance budget, accessibility pass, error states (empty file, wrong file type, huge file, no recurring charges found: say so kindly), copy review against house rules. Accept: all targets met.
- **M7 launch kit.** README, screenshots, the example statement, FAQ page, issue templates, a "how it works" write-up for the launch post. Accept: owner review. The owner deploys.

## 14. Launch plan

- Timing: ready by mid December. Launch the first Tuesday to Thursday of January, 8 to 10 in the morning US Eastern, when people clear subscriptions. A soft Black Friday mention is optional.
- **Hacker News:** "Show HN: pantry, find forgotten subscriptions from a bank CSV, entirely in your browser". The link goes straight to the working page (no sign-up). The first comment explains how recurrence is detected and how to verify the privacy claim. The author stays in the thread for the first hours.
- **X:** a 20 second screen recording: drop file, results appear, savings counter, share card. One sentence: "found 14 subscriptions, 1,326 a year. it never left my laptop." Post the card, not the list.
- A "how it works" post covers the recurring-charge algorithm, the messy merchant string problem and the CSP. Technical readers like that.
- Honest odds: nobody can promise a top-10 spot. This has a real need, a personal result and a checkable privacy claim, which are the things that tend to travel. Execution quality decides it.

## 15. Risks

- **Accuracy on real statements** is the main risk. Mitigations: wide fixture coverage, visible reasons, "maybe" tier, easy correction, redact helper for reports.
- **Catalogue rot.** Cancel links change. Mitigation: help-centre links, `lastChecked`, report button, honest labels.
- **Name collision.** "pantry" is a common word and brand. Check the name and domains first (section 16).
- **Legal.** Say plainly it is not financial advice and not affiliated with any service. Use service names as plain text only.
- **Statements are sensitive.** Even local-only, people hesitate. The proof panel and single-file build exist for them.
- **Peak season.** January is the best window and also crowded. Be ready early.

## 16. Decisions for the owner before building

1. Final name (pantry, larder, cellar, other) and whether a domain is wanted now.
2. Regions for the first catalogue (suggested: US, GB, EU).
3. Repo and licence (suggested: public `Tuftle/pantry`, MPL 2.0 code, CC0 catalogue).
4. Whether the share card may ever show service names (suggested: opt-in only).
5. Whether to also offer the single-file download at launch (suggested: yes).

## 17. How the building session reports

After each milestone, send:
- what is done, in plain words
- test counts and the accuracy numbers (from 11.2) where relevant
- anything uncertain or guessed (especially catalogue facts and bank layouts)
- what you will do next and what you need from the owner
- cost note: roughly how much of the budget this milestone used, if known

## 18. House rules from the Tuftle handoff (for consistency)

- Copy is lowercase and calm. No em dashes, buzzwords or emojis in anything written for people.
- The cream and brown palette is deliberate. Do not "fix" it.
- Git author is `tuftle <tuftle@proton.me>`. No Claude co-author lines.
- Keep secrets out of chat and out of the repo.
- Siblings for reference: wren (browser), nook (search), burrow (password manager). burrow's README (https://github.com/Tuftle/burrow) is the style model for READMEs: centred icon, badges, short link row, overview table, install section, privacy table, layout tree, licence, small footer. The sibling repositories are public at https://github.com/Tuftle.
