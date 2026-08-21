# 12 — Code map: which file calls what

The other documents describe layers. **This one names functions and line
numbers**, so you can follow any behaviour from the browser to the database
without guessing.

Every location below was extracted from the source, not written from memory.
Line numbers drift as the code changes — the function names are the stable
part, so `grep` the name if a number looks off.

---

## 1. The eight files that matter

| File | Lines | Owns | Imports from us |
|---|---:|---|---|
| `engine/domain.py` | 415 | Every data type. Frozen dataclasses, all the derived-value logic. | *nothing* |
| `engine/engine.py` | 590 | The five stages. All quantity and cost arithmetic. | `domain`, `catalogue` |
| `engine/catalogue.py` | 58 | `CatalogueView` — the read-only snapshot the engine queries. | `domain` |
| `engine/repository.py` | 305 | SQLite → `CatalogueView`. The only file that runs SQL for the engine. | `domain`, `catalogue`, `resourcing` |
| `engine/resourcing.py` | 208 | People cost: build days and ops FTE. Independent of the token engine. | *nothing* |
| `api/schemas.py` | 478 | Pydantic request/response contracts. | *nothing* |
| `api/service.py` | 2,709 | Orchestration. DTO↔domain mapping, traces, confidence. | all of the above |
| `api/main.py` | 712 | Routes, auth, response assembly. Thin by design. | `service`, `schemas`, `domain`, `engine` |

**Dependency direction is strictly one-way.** `domain.py` imports nothing of
ours and `engine.py` never imports `service` or `main`. If you find yourself
wanting an engine file to import from `api/`, the logic belongs in `service.py`
instead.

---

## 2. The main path, end to end

This is what happens when someone presses **Submit to ledger**. Every arrow is
a real call.

```
BROWSER  v3.html
  buildRequest()                          v3.html:2868
      reads the form into the API's JSON shape
      │  POST /estimates
      ▼
API      main.py
  create_estimate(req, x_api_key)         main.py:257
      ├─ service.resolve_role(key)        service.py:974     key → role
      ├─ service.create_estimate(req)     service.py:320
      └─ _estimate_response(out, req)     main.py:193        dict → EstimateResponse
              └─ _band_low / _band_high   main.py:49 / 55    per-line ±40%

SERVICE  service.py
  create_estimate(req)                    service.py:320
      ├─ self._price(req)                 service.py:262     ◄── THE SHARED PATH
      └─ repo.save_estimate(...)          ◄── the ONLY line that writes a ledger row

  _price(req)                             service.py:262
      ├─ _to_spec(req)                    service.py:138     DTO → frozen SolutionSpec
      ├─ repo.load_catalogue()            repository.py:42   one snapshot
      ├─ engine._effective_usage()        engine.py:74       merge archetype defaults
      ├─ _to_resourcing_spec(req, months) service.py:182     (only if a team was given)
      ├─ engine.run_estimate(...)         engine.py:475      ◄── THE FIVE STAGES
      ├─ resourcing.price_resourcing()    resourcing.py:136  people cost
      ├─ _build_derivations()             service.py:46      the trace string per line
      ├─ engine.observability_coverage()  engine.py          coverage check
      ├─ _assumption_report()             service.py:2525    per-field source + confidence
      └─ _line_meta()                     service.py:2609    per-line nature + rationale

ENGINE   engine.py
  run_estimate(spec, catalogue, priced_on, band)   engine.py:475
      ├─ resolve(spec, catalogue)         engine.py:142   ① quantities
      ├─ apply_prices(lines, cat, date)   engine.py:241   ② dated prices  ◄── ONLY price read
      ├─ calculate(priced, catalogue)     engine.py:282   ③ qty × price
      ├─ aggregate(costed, band)          engine.py:306   ④ roll-up + band
      ├─ sanity_check(result, ...)        engine.py:345   ⑤ range warnings
      ├─ usage_gaps(spec, catalogue)      engine.py:367       missing drivers
      └─ usage_coherence(spec, catalogue) engine.py:420       contradictory drivers
```

### Inside each stage

```
resolve()                                 engine.py:142
   ├─ _effective_usage()                  engine.py:74    archetype merge
   ├─ u.effective_input_tokens()          domain.py:239   ── prompt size
   ├─ u.effective_cached_tokens()         domain.py:260   ── caching derivation
   ├─ u.corpus.index_tokens()             domain.py:177   ── corpus stock
   ├─ _expand_dependencies()              engine.py:49    ── transitive, cycle-safe
   └─ _line_quantity()                    engine.py:40    ── × horizon, or one-off

calculate()                               engine.py:282
   └─ _flat / _per_unit / _per_token / _tiered   engine.py:259 / 262 / 265 / 269
```

### The derived values, in `domain.py`

These call each other in a chain — this is where the "ask for the decision,
derive the number" principle actually lives:

```
effective_cached_tokens()      domain.py:260
   ├─ cacheable_prefix()        domain.py:245
   │     └─ resolved_anatomy()  domain.py:212  ◄── the hub
   ├─ resolved_anatomy()        domain.py:212
   └─ effective_input_tokens()  domain.py:239
         └─ resolved_anatomy()  domain.py:212

resolved_anatomy()             domain.py:212
   └─ retrieval.context_tokens()  domain.py:153   top_k × chunk_tokens
```

`resolved_anatomy()` is the single most-called function in the domain — five
callers across `domain.py`, `engine.py` and `service.py`. **Change it and you
change RAG context, history, prompt size, caching and confidence at once.**

---

## 3. The three pricing routes share one function

This is the most important fact in the codebase.

| Route | Handler | Service method | Writes a row? |
|---|---|---|---|
| `POST /estimates` | `main.py:257` | `create_estimate` → `service.py:320` | **Yes** |
| `POST /estimates/dry-run` | `main.py:274` | `dry_run_estimate` → `service.py:330` | No |
| `POST /estimates/preview` | `main.py:282` | `preview_estimate` → `service.py:336` | No |

All three call `_price()` at `service.py:262`. They cannot disagree about
arithmetic — only about whether a row is written. If you add a fourth way to
price something, route it through `_price()` too.

---

## 4. Route → what the handler calls

Advisor routes omitted (out of handover scope for the PRD, but present in code).

| Route | Calls |
|---|---|
| `POST /estimates` | `service.resolve_role`, `service.create_estimate`, `_estimate_response` |
| `POST /estimates/dry-run` | `service.dry_run_estimate`, `_estimate_response` |
| `POST /estimates/preview` | `service.preview_estimate` |
| `POST /estimates/projection` | `service.project` |
| `POST /estimates/compare` | `service.compare` |
| `GET /estimates` | `service.list_estimates` |
| `GET /estimates/{id}` · `/export` | `service.get_estimate` |
| `POST /estimates/{id}/actuals` | `service.add_actual` |
| `GET /catalogue` | `service.get_catalogue_listing` |
| `GET /roles` | `service.list_roles` |
| `GET|POST /drafts`, `GET /drafts/{id}` | `service.save_draft`, `list_drafts`, `get_draft` |
| `GET /whoami` | `service.resolve_role` |
| `POST /admin/components` | `service.add_component` |
| `POST /admin/components/{code}/archive`·`/restore` | `service.set_component_archived` |
| `POST /admin/prices` | `service.change_price` |
| `GET /admin/prices/{code}` | `service.price_history` |
| `POST /admin/users` ·`/activate` ·`/deactivate` | `service.add_user`, `set_user_active` |
| `GET /admin/users` · `/admin/log` | `service.list_users`, `service.admin_log` |
| `GET /dashboard/business` · `/export` | `service.business_dashboard` |

Every admin route goes through the `require_admin` dependency at `main.py:69`,
which calls `service.resolve_role` and raises 403 for a non-Admin key.

---

## 5. Frontend — 109 functions, the fifteen that matter

`api/static/v3.html`. No framework: **state changes are explicit calls.** If
you change a value and nothing moves, you did not call the renderer.

| Function | Line | Role |
|---|---:|---|
| `buildRequest()` | 2868 | Reads the whole form into the API's JSON shape. **Start here** when a field isn't reaching the server. |
| `scheduleLivePrice()` | 2941 | Debounces, then POSTs `/estimates/preview` for the live bar. |
| `autoPrice()` | 3042 | On landing at stage 5, POSTs `/estimates/dry-run`. |
| `renderResult(r)` | 3317 | Builds the entire result page from the response. |
| `breakdownRows(r,t,m)` | 3230 | The three-level table: area → component → metered lines. |
| `lineRow(l,owner,anc,lvl)` | 3213 | One metered line, with its nature and depth class. |
| `confChip(l)` | 3198 | The confidence chip and its rationale panel. |
| `renderConfidencePanel(r)` | 3428 | Countable vs behavioural split. |
| `renderAssumptionsPanel(r)` | 3468 | Editable assumptions. |
| `runWhatIf()` | 3494 | Re-prices an edit through `/estimates/preview`; never mutates the frozen record. |
| `renderAnatTotal()` | 4049 | Total input tokens per call, **including derived slices**. |
| `derivedSlices()` | 4067 | **The client-side mirror** of the RAG/history formulas. |
| `renderDerived()` | 4117 | Shows the RAG and history arithmetic. |
| `renderCaching()` | 4143 | Shows the caching derivation, cliff and cold-start. |
| `renderCorpus()` | 4212 | Shows index / churn / query arithmetic. |
| `goStep(n)` | 4477 | Moves between wizard stages; triggers the stage-4 renderers. |
| `openAssist(field)` | 4650 | The `? guide` popover. |

### The mirror, and why it is a hazard

`derivedSlices()`, `renderCaching()` and `renderCorpus()` **re-implement server
formulas in JavaScript** so the user sees arithmetic update as they type.

Nothing enforces that the two agree. If you change a formula in `domain.py` you
must change it here too, or the panel will confidently quote a number the
engine does not bill. Reconcile by hand: set the inputs, read the panel, POST
the same inputs to `/estimates/dry-run`, compare.

---

## 6. "I want to change X" — the file order

### Add a usage driver

1. `engine/domain.py` — field on `UsageAssumptions` (~line 180), default meaning "not supplied"
2. `engine/engine.py` — carry it through `_effective_usage()` (line 74), then use it in `resolve()` (line 142)
3. `api/schemas.py` — field on `UsageIn`, with bounds
4. `api/service.py` — map it in `_to_spec()` (line 138); add to `_USAGE_DEFAULTS` and `_USAGE_KIND`; add to `_line_meta()` drivers (line 2609) if it affects a line's confidence
5. `api/static/v3.html` — input markup, then `buildRequest()` (line 2868), then a live readout
6. `api/service.py` — an `ASSIST_PLAYBOOKS` entry so the field gets a `? guide`
7. Tests — assert the rule, not just a number

Skip step 4 and the field will price correctly but never appear in the
confidence report. Skip step 5 and it will exist in the API and be invisible.

### Change how a quantity is computed

`engine/engine.py::resolve()` (line 142) — and if it is a derived value,
`engine/domain.py` (lines 212–280). Then **the client mirror** in `v3.html`.

### Change a price

No code at all. See [10-RUNBOOK.md](10-RUNBOOK.md).

### Add a component type

1. `db/schema.sql` — the `CHECK` constraint on `component_type`
2. `engine/domain.py` — the `ComponentType` enum
3. `engine/engine.py::resolve()` — a branch for its quantity formula
4. `api/service.py::_line_meta()` — its nature and drivers
5. `api/service.py::_build_derivations()` — its trace string

### Change what a line's confidence says

`api/service.py::_line_meta()` (line 2609) for the grade and rationale;
`_DRIVER_LABEL` just below it for the reader-facing names; `confChip()` in
`v3.html` (line 3198) for how it renders.

### Add an endpoint

`api/main.py` for the route, `api/schemas.py` for the contract,
`api/service.py` for the logic. **Keep the handler thin** — if it contains
business logic, that logic belongs in `service.py`.

---

## 7. Reading order for a new developer

Half a day, in this order:

1. **`engine/domain.py`** — the vocabulary. Everything else manipulates these types. Read `UsageAssumptions` and its three derived methods (lines 180–280) carefully; they carry most of the product thinking.
2. **`engine/engine.py`, lines 142–345** — the five stages. This is the entire costing model in ~200 lines.
3. **`api/service.py::_price()`, line 262** — how a request becomes a priced result. Twelve calls, in order.
4. **[07-COST-MATH.md](07-COST-MATH.md)** — the arithmetic with a worked example whose figures match the engine exactly.
5. **`api/run_api_tests.py`** — reads as a specification of what the system guarantees.

Leave `v3.html` until last. It is the largest file and the least surprising.

---

## 8. How this map was produced

By parsing the source with Python's `ast` module: every function definition,
every call site, and the import graph — then verifying the key chains by
reading the code. Regenerate it after significant refactoring rather than
trusting the line numbers indefinitely.

Two things the extractor got wrong, worth knowing if you re-run it: it cannot
distinguish our `resolve()` from `Path.resolve()`, and it reports
route handlers as having no callers because FastAPI invokes them by decorator.
Neither affects the chains above, which were checked by hand.
