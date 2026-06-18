# FallAdviser v2 · sovereign multi-client UK financial-advice tool

**One HTML file.** Multi-firm, multi-adviser, multi-client. Runs entirely in your browser. No server, no cloud, no telemetry. Client data never leaves the device.

Part of the IFA bundle: FallAdviser v2 · FallOnboard · FallPaper · FallPractice. All four tools share a single client schema and exchange records over a `BroadcastChannel` mesh when open on the same device.

---

## For advisers · the 60-second walkthrough

1. **Open `index.html`** in any modern browser (Chrome 113+ recommended). On first launch you set up your firm (name, FCA ref, address) and add the first adviser.
2. **Sidebar lists every client.** Filter by adviser, risk grade, ATR, or review-due. Click a client to make them active. Search by name, email, or postcode.
3. **Profile tab** captures the full FCA-aligned record: KYC/AML, vulnerability, suitability, engagement, income/contributions, estate (for IHT).
4. **Tax tab** computes income tax (England/Wales/NI **or** Scotland bands), NI, dividend tax, savings tax (PSA + starting rate band), CGT, plus marriage allowance interaction and HICBC clawback — every figure tied to your client's numbers.
5. **Pension tab** projects the pot, models drawdown, applies tapered annual allowance + 3-year carry-forward, supports MPAA flag, and runs a tax-aware withdrawal sequence.
6. **Portfolio tab** tracks holdings with CGT lots (FIFO matching on disposal), surfaces Bed-and-ISA opportunities, flags concentration > 10%, and compares to ATR-target allocation (DT / Defaqto 1–7).
7. **Suitability tab** has the 10-item Likert ATR questionnaire that auto-scores to 1–7, plus CFL, knowledge & experience, horizon, income needs, objectives.
8. **Scenarios tab** snapshots the client's full state into named scenarios (retire at 60, retire at 67, downsize, max SIPP carry-forward) and compares up to three side-by-side.
9. **Q & A tab** — type a question. T0 (offline rules) covers the 13 most common UK-finance questions instantly; with an API key (Anthropic, Gemini, OpenAI, OpenRouter) the T3 path gets your client's full context for nuanced answers.
10. **Reports tab** generates a COBS-aligned annual review pack (Markdown), CSV exports (holdings, lots, disposals, firm-wide), full state JSON backup, and audit-chain export.

### What's new vs v1

- **Multi-client / multi-adviser / multi-firm** per the shared IFA-bundle schema
- **Scotland income tax bands** (starter 19% → top 48%)
- **Marriage allowance** transfer (£1,260, basic-rate recipient)
- **HICBC** with post-2024 taper £60k–£80k
- **Tapered annual allowance** (£260k adjusted income, £10k floor)
- **Carry-forward** 3 prior tax years' unused AA
- **BPR / APR** with post-2026 reform cap (£1m at 100%, then 50%)
- **Personal savings allowance** + starting rate band for savings
- **CGT lot tracking** with FIFO matching on disposal
- **Bed-and-ISA** suggestion engine
- **Tax-aware withdrawal sequencing**
- **ATR questionnaire** (10 Likert → 1-7 score)
- **Scenarios** with side-by-side compare
- **Annual review workflow** with FallPaper handoff
- **`fall-client` BroadcastChannel mesh** for cross-tool sync
- **Audit chain** per shared extended P3 schema (tool, adviser, client, action, reasoning, prevHash, docHash)

### Disclaimer

FallAdviser is **informational, not regulated financial advice**. UK rules calibrated to 2025-26 tax year. The bundle is sovereign — client data never leaves the device. For binding decisions, consult an FCA-authorised adviser.

---

## For developers · the architecture

### Single-file principle

Everything lives in `index.html`: HTML, CSS, vanilla JS, PWA manifest (data: URL), SVG icon, all engines, all views. No build step. No dependencies. ~190 KB compressed; ~3,000 lines.

### Storage

Primary: **IndexedDB** (`STORE='falladviser-v2'`, `DB_VERSION=2`) with stores:

| store | keyPath | holds |
|---|---|---|
| `firms` | id | single firm record |
| `advisers` | id | many advisers |
| `clients` | id | many clients (shared schema) |
| `scenarios` | id | client scenarios |
| `audit` | id | append-only Mansoor P3 extended audit chain |
| `settings` | keyed by 'app' / 'meta' | API keys, current adviser, filters, chat |

Fallback: **localStorage** with keys prefixed `falladviser-v2::<store>`. Auto-engages if IDB unavailable (hostile sandbox, private mode).

### Shared client schema

Conforms to `IFA-BUNDLE-SHARED-SCHEMA.md`. The client record has the canonical FCA-aligned fields (identity, contact, KYC, suitability, engagement, relationships, notes, links). FallAdviser-specific calculation fields live under `client.financials` to avoid polluting the shared schema. Other bundle tools (FallOnboard, FallPaper, FallPractice) read/write the same `clients` store and exchange updates via the `fall-client` BroadcastChannel mesh.

### BroadcastChannel mesh

Two channels:

- **`fall-client`** — sync of client/adviser/firm records. Boot-time `sync.request` / `sync.snapshot`, then `client.created`/`client.updated`/`client.archived` etc. Debounced (~300ms) on rapid edits. Receiver merges by `updatedAt` (later wins).
- **`fall-signal`** — discovery + Q&A integration with si-didy. Emits `hello` on boot; responds to `ping` with `pong`; answers `si-didy/query` with the T0/T3 answer.

### Tax engines (calibrated 2025-26)

| function | purpose |
|---|---|
| `adjustedPersonalAllowance(income)` | PA taper £1/£2 over £100k |
| `incomeTaxEngland(income, paAdj?)` | E/W/NI bands |
| `incomeTaxScotland(income, paAdj?)` | starter/basic/intermediate/higher/advanced/top |
| `nationalInsurance(income)` | Class 1 employee 8% / 2% |
| `dividendTax(div, salary, region)` | allowance + 8.75/33.75/39.35% |
| `capitalGainsTax(gain, salary)` | 18/24% post-October 2024 |
| `savingsTax(int, salary, region)` | PSA + starting rate band |
| `marriageAllowanceImpact(client)` | £1,260 transfer eligibility + impact |
| `hicbcCharge(client)` | post-2024 £60k-£80k taper |
| `adjustedNetIncome(client)` | for PA taper + HICBC |
| `thresholdIncome(client)` | pension taper test |
| `adjustedIncomeForAA(client)` | pension taper test |
| `taperedAA(client)` | £260k adj income, £10k floor |
| `carryForwardAA(client)` | 3 prior years unused |
| `ihtEstimate(client)` | NRB + RNRB + BPR/APR cap |
| `disposeFIFO(holding, units, price)` | CGT lot matching |
| `bedAndIsaSuggestions(client)` | non-ISA holdings with gain + ISA headroom |
| `withdrawalSequence(client, annual)` | cash → ISA → SIPP TFC → GIA → SIPP taxable |
| `scoreATR(answers)` | 10-question Likert → 1-7 |
| `portfolioAnalysisFor(client)` | total, allocation, concentration, target |
| `generateSuggestionsFor(client)` | per-client prioritised recommendations |
| `totalTaxFor(client)` | aggregate liability + composition |

### Q&A · T0 / T2 / T3 cascade

13 T0 deterministic patterns cover: ISA vs SIPP, CGT, marriage allowance, HICBC, carry-forward, tapered AA, BPR/APR, Scotland bands, state pension, 60% trap, personal savings allowance, Bed-and-ISA, withdrawal sequencing. T2 (Ollama local on 11434) and T3 (BYOK Claude / Gemini / GPT / OpenRouter) get full active-client context.

### Audit chain (Mansoor P3 extended)

Every mutation to firm/adviser/client/scenario appends:

```js
{i, ts, tool:'falladviser', adviserId, clientId, action, reasoning,
 configVersion:'falladviser@2.0.0', prevHash, docHash, payload}
```

Chain is append-only in IDB, 7-year FCA retention. Export as JSON for compliance.

### Postable API (`postMessage`)

Sibling tools or si-didy can post `{target:'falladviser', action:..., ...}`:

- `ping` → version, prime, firmId
- `ask` `{question}` → T0/T3 answer
- `list-clients` → all active clients
- `get-client` `{clientId}` → single record
- `list-advisers` → all advisers
- `get-firm` → firm record
- `get-tax` `{clientId}` → tax breakdown
- `get-suggestions` `{clientId}` → recommendations
- `select-client` `{clientId}` → switch active client

### 14-point gate compliance

| | |
|---|---|
| Single HTML file | ✓ |
| < 500 KB (bundle-scope override) | ✓ (~190 KB) |
| IDB primary | ✓ (6 stores) |
| localStorage fallback | ✓ |
| KONOMI shim | ✓ `window.KONOMI` |
| `fall-signal` channel | ✓ |
| `fall-client` channel | ✓ |
| PWA manifest data URL | ✓ |
| T0 offline always works | ✓ (13 patterns) |
| Mobile-first ≤880px | ✓ (drawer for client list) |
| MIT license | ✓ `LICENSE` |
| Two-audience README | ✓ this file |
| "not regulated advice" disclaimer | ✓ banner + Q&A footer |
| Audit chain on by default | ✓ |
| Oxblood / brass / cream / void palette | ✓ |
| Forkable brand | ✓ via firm.brandColor |

### Verifying / hacking

The single HTML file is also the artifact. To smoke-test the JS engines:

```bash
node -e "
const html=require('fs').readFileSync('index.html','utf8');
const js=html.match(/<script>([\\s\\S]*?)<\\/script>/)[1];
require('fs').writeFileSync('_engine.js',js);
" && node -c _engine.js
```

The verification suite that shipped with v2.0.0 confirmed: income tax @£50k = £7,486, @£100k = £27,432, 60% trap marginal = 0.60, Scotland @£50k = £9,028, tapered AA @£300k = £32,500, HICBC @£70k with £2,400 CB = £1,200, ATR scoring 1↔7, FIFO disposal 120u @£15 sale on 100-cost £10 + 50-cost £16 lots = £480 realised gain, IHT estate £1.1m with £200k BPR = £160,000 tax / £400k taxable.

### License

MIT · 2026 Simon Gantley.
