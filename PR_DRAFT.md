## What does this PR do?

Adds `providers/talentlink.mjs`, a single-company ATS provider for Lumesse TalentLink (Cornerstone) career sites. It auto-detects pages on `*.recruitmentplatform.com`; an employer's own page that embeds the TalentLink widget works with an explicit `provider: talentlink`.

- **Data path.** The career page is a static shell that names its FO API host and its career-site id, in either of the two attribute sets the widget uses (the standard one and the accessible "WCAG" one). The provider reads these two values from the page and the posting page name from the site's FO config. It then reads the postings from the public FO jobs API that the page itself calls, with the fixed guest login every visitor's browser sends.
- **Fields per posting:**
  - title and URL;
  - location from the tenant's location lists (plus the bound country, and Remote/Hybrid from the work mode);
  - posting date when the tenant lists it;
  - `requisitionId` from the job reference;
  - `language` from the posting's own locale;
  - the full description, which comes in the list response, so there are no extra requests.

Before merging, please read the robots.txt question in the related issue: the FO API hosts disallow `User-agent: *`.

## Related issue

Refs #<issue number> (robots.txt on the TalentLink FO API hosts; this PR waits for a decision there).

## Type of change

- [ ] Bug fix
- [x] New feature
- [x] Documentation / translation
- [ ] Refactor (no behavior change)

## Checklist

- [x] I have read [CONTRIBUTING.md](https://github.com/career-ops-hq/career-ops/blob/main/CONTRIBUTING.md)
- [x] If this is a new feature or architecture change, I opened an issue first (bug fixes, providers, docs & translations are exempt — send those straight in)
- [x] My PR does not include personal data (CV, email, real names, scan results, or pipeline data)
- [x] I ran `node test-all.mjs` and all tests pass
- [x] My changes respect the [Data Contract](https://github.com/career-ops-hq/career-ops/blob/main/DATA_CONTRACT.md) (no modifications to user-layer files)
- [x] My changes align with the [project roadmap](https://github.com/career-ops-hq/career-ops/discussions/156)

## Tests

**Commits:**

- `feat(providers): talentlink reads Lumesse TalentLink FO career sites`. Contents:
  - The provider.
  - `tests/providers/talentlink.test.mjs` (24 node:test cases), covering:
    - `detect()` positive and negative cases;
    - path handling;
    - the host guard firing before any request (call count 0);
    - an explicit `provider: talentlink` on a branded host, with the FO host still validated;
    - `redirect: 'error'` and the guest headers on every request;
    - normalization of the real payload shape;
    - dropped rows;
    - empty board vs. a wrong envelope;
    - a page without the widget;
    - 403 mapped to 404;
    - pagination with a short page and with `jobsCount`;
    - `DEFAULT_MAX_PAGES` with the "raise max_pages" warning;
    - `ctx.maxPages: 1` issuing exactly one list request and no criteria request;
    - a failed page 2 (keeps page 1, warns, no max_pages hint);
    - unchanged propagation of a fetch rejection while probing;
    - a failed criteria call;
    - the id cache, including dropping a cached pair after a 403.
  - A lone-surrogate case and the `CONVERTED` entry in `tests/providers/url-encoding-surrogate.test.mjs`.
- `docs(providers): list talentlink in the supported boards and the example portals`. Contents:
  - one row in `docs/SUPPORTED_JOB_BOARDS.md`;
  - in `templates/portals.example.yml`: the auto-detection line, an example stanza, a branded-domain stanza with `provider: talentlink`, and one live entry shipped with `enabled: false`.
- `feat(providers): register talentlink hosts for vendor labels, history seeds, trust and host checks`. Contents:
  - `ATS_HOST_PATTERNS` in `ats-vendor.mjs`;
  - `atsBoardUrlOf` in `history-ats-seeds.mjs`, which keeps the site path and drops the page file;
  - `DEFAULT_ATS_ALLOWLIST` in `providers/_trust-validator.mjs`. Before the change, a posting whose company name differs from the tenant host was flagged `company_domain_mismatch`; after it, it is not;
  - `AGGREGATOR_HOSTS` in `verify-portals.mjs`. Before the change, `companyFromHost` suggested "recruitmentplatform" as a company name; after it, it returns null;
  - a test case for each of these.
- `fix(providers): talentlink reads an empty board answered without a jobs key`. The live check, on a site whose employer has moved to another ATS, found that an empty board, and a page past the end, answer `{"globals":{"jobsCount":0}}` with no `jobs` key. That shape is now an empty board. A missing `jobs` with a positive count still throws.
- `feat(providers): talentlink reads the accessible widget and employer-hosted career pages`. Contents:
  - the page attributes of the accessible widget (`data-talentlink-fo-host`, which carries the scheme, and `data-talentlink-fo-site-tech-id`);
  - the career page the entry names is fetched as given (`jobsearch.html`, `details.html`), and a directory URL gets `index.html`;
  - posting URLs use `navigationPages.jobDescriptionURL` from `/fo/rest/config`. A relative value resolves against the career page; an absolute one is kept only on the career page's own host and over https; anything else falls back to `details.html`. On the vendor-hosted sites checked, this is `details.html`, so their posting URLs are unchanged;
  - the config is read once with the page, and all three values are cached together;
  - new tests for both attribute sets, an employer-hosted page, the posting-page rules, and a 403 on the config call.
- `docs(providers): describe talentlink employer-hosted pages in the board list and the example portals`.

**Live check** (serial requests, 1 s between hosts), done through `loadProviders` + `resolveProvider` and `makeHttpCtx()`:

- 11 FO widget sites (10 on vendor tenant hosts, 1 on an employer's own domain with the accessible widget):
  - every count matched the count on the rendered page (46 of 46 on the employer-hosted site);
  - with `ctx.maxPages: 1`, each site issued exactly one list request (100 rows);
  - the first and last returned URL of each of the 9 sites with jobs (18 URLs) were opened in a browser, and the title on the page matched every time;
  - the empty board returned `[]`. That site still answers, but its employer now posts on iCIMS, so it is a live example of the empty-board answer.
- Other answers:
  - an unknown tenant answered 404;
  - a syndicated page failed with a named error;
  - a posting URL given as the entry resolved to its site's board;
  - the employer-hosted site without `provider: talentlink` is not claimed by any provider.
- Full suite: `node test-all.mjs`, <result of the final run>.

**Left out:**

- **The older syndicated job list.** Found on 2 of 12 tenants that answer; it has no FO widget.
- **Locations in an unlabelled free-text field.** One tenant uses one, and nothing in any response names it.
- **Liveness and JD text.** A closed posting answers `400 {"errorMsg":"Requisition not found"}` on `/fo/rest/jobs/<id>/details`, and its page says the posting has expired. An unknown id answers a different 400, so the signal is reliable. But the request needs the site id and FO host, and a posting URL carries neither, so a check needs two requests: the career page, then the details call. An `ATS_PROVIDERS` entry in `liveness-api.mjs` builds one request from the URL. This is a follow-up that needs a two-step entry there.
- **`VENDOR_ANALYSIS_SCOPE` in `analyze-patterns.mjs`.** Not changed (a fixed taxonomy).
- **The existing providers.** `providers/csod.mjs` reads a different Cornerstone product. `providers/rheinmetall.mjs` reads the employer's own server-rendered page. Neither shares code with this provider.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
