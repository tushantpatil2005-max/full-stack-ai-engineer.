Dhaba support triage — Propel take-home
POST /triage takes a ticket exactly as it appears in dhaba_tickets.json and returns an action-ready JSON decision.
Python 3.10+, standard library only (no pip install). Track chosen for Task 3: backend.
Run it (clean machine)
git clone <this repo> && cd dhaba-triage
python3 -m unittest discover -s tests          # 17 tests, <1s, no key, no network
python3 scripts/run_all.py                     # all 12 tickets through the real HTTP server, replay mode -> table + results/
python3 -m app.server --mode replay --port 8080 &
curl -s -XPOST localhost:8080/triage -d "$(python3 -c "import json;print(json.dumps(json.load(open('data/dhaba_tickets.json'))['tickets'][0]))")"
Modes (--mode or DHABA_MODE):
replay (default) — model outputs come from fixtures/model_replay.json. No key needed.
live — Anthropic Messages API via urllib; needs ANTHROPIC_API_KEY, optional DHABA_MODEL (default claude-sonnet-5-5).
fallback — forces the degraded (no-model) path.
GET /healthz, GET /metrics (counters: fallback, shed, validation failures…). DB: SQLite at --db (default dhaba.db).
Honesty note: I had no network/API key while building, so (1) live mode is written but not exercised against the real API, and (2) the replay fixtures are hand-written to look like model output. The table below therefore shows what my code does with plausible model output — not how a real model scores. Run --mode live and diff against results/replay_output.json before trusting any accuracy claim.
Response shape
{"ticket_id":"T-1001","category":"unexpected_charge","severity":2,
 "refund":{"decision":"approve|review|deny|none","amount_inr":249,"purchase_ids":["pay_A2"],"reason":"..."},
 "reply_draft":"...","language":"en|hinglish|other","needs_human":false,"human_reasons":[],
 "confidence":0.9,"route":"support|engineering|billing_ops|risk","flags":[],
 "meta":{"mode":"replay|live|fallback","model_attempts":1,"fallback_reason":null,"replayed":false}}
category (11): unexpected_charge, refund_request, fraud_dispute, payment_issue, cancellation, bug_report, account_access, feature_request, invoice_request, presale_question, other.
severity: 1 feature/info · 2 normal · 3 service broken, money stuck or time-sensitive · 4 fraud/legal/safety. Code may raise the model's severity (fraud ⇒ 4, ≥₹1000 under review ⇒ ≥3), never lower it.
confidence: model's, capped at 0.5 for ≤2-word messages, fixed 0.35 in fallback. < 0.6 ⇒ needs_human: true (draft still returned, flagged).
How the pieces answer the brief
Output holds its shape. schema.parse_model_output tolerates code fences/trailing prose, then strictly validates category (enum), severity (1–4), confidence (0–1), language, reply length; unknown keys are dropped. On failure: one retry with the failure code (never content) fed back; then degrade to a rules classifier + canned reply, needs_human: true, confidence 0.35. Transport errors retry once; a breaker opens after 5 consecutive failures and a bounded semaphore sheds load to the same fallback. Tests cover truncated JSON, prose, invented category, 503s, breaker.
Money has its own gate. The model cannot write refund — the field doesn't exist in its schema and extra keys are discarded. policy.py (pure code) derives the decision from category + billing facts + the refund ledger:
approve only for an unused (0 opens) monthly renewal within 7 days of the ticket, ≤ ₹249, or a confirmed same-day duplicate. Never when injection-like text was present, never in degraded mode, never if that purchase is already in the ledger.
Annual (₹1499), fraud claims, used/old renewals, stuck payments ⇒ review (human). Trial-only ⇒ deny. Duplicate claim the ledger doesn't support ⇒ deny with the reason.
The worst a fully compromised model can do is choose a category that yields review. Reply text is also guarded: a draft claiming a refund was processed/approved when the engine didn't approve, quoting an amount not in the ledger, or echoing prompt-leak phrases is replaced by a canned reply. The refund sentence in the reply is appended by code from the decision.
These thresholds (₹249 cap, 7 days, 0 opens, 5–7 working-day wording) are my assumptions — no policy was provided. They're constants at the top of policy.py.
Runs twice safely. triage_results.ticket_id is a primary key; a repeat returns the stored response (Idempotent-Replay: true, no second model call, also under 10 concurrent duplicates — tested). Same id with different content ⇒ 409. Separately refund_ledger.purchase_id is a primary key, so a new ticket id about an already-refunded purchase gets none (tested). Decision + commit run inside one lock/transaction; the slow model call is outside it. This service authorises a refund (ledger row); the support tool executes it.
Logs are clean. logsafe.log(event, **fields) is the only logger: allow-listed keys only (ids, enums, numbers, flags), no free-text path, every string scrubbed for email/phone/GSTIN, exceptions logged as class name only, default HTTP access log disabled, ticket ids validated ^[A-Za-z0-9_-]{1,64}$ (no log injection). A test runs all 12 tickets (plus an error path) and greps logs for company name, GSTIN, "father", device model, quoted phrases. GSTIN/email/phone are also redacted before the model sees the text. Company names can't be reliably detected by regex and would still reach a third-party model — flag for a privacy review.
Decision I made on ambiguity in the brief: the PDF lists four purchase "states" including renewal; the data has status ∈ {initiated, failed, successful} and type ∈ {trial, renewal}. I treat renewal as a type, and reject anything else with a 400.
Task 2 — all twelve tickets (replay mode)
id
what it decided
agree?
T-1001
unexpected_charge, sev 2, approve ₹249 (pay_A2), no human
Yes. Unused, 0.5 days old, within cap. Auto-approval is a policy choice; the user's "did not agree" is really the ₹1→₹249 model they didn't read.
T-1002
bug_report, sev 3, no refund, route engineering
Yes. 41 opens, paying, no money ask.
T-1003
account_access, sev 2, refund none, human, prompt_injection_suspected
Yes. Injection #1. See below.
T-1004
cancellation, sev 3, none, human, conf 0.5 (low)
Yes — and genuinely ambiguous (cancel the trial? the plan? the account?). One word + a ₹1 trial that auto-renews ~24h later; nothing here can cancel, so a person must.
T-1005
payment_issue, sev 3, review, amount 0
Yes — ambiguous. User says ₹249 debited; our ledger says initiated, not successful. Nothing refundable on our books; needs gateway reconciliation.
T-1006
unexpected_charge, sev 3, review ₹1499
Yes on review; I'd expect a human to approve. 0 opens, same-day. Autopay cancelled in UPI is a claim we can't verify. Auto-cap exists because annual refunds are large.
T-1007
feature_request, sev 1, none, flag latest_payment_failed
Yes. Reply also tells them the 21 Aug payment failed.
T-1008
fraud_dispute, sev 4, review ₹499, human
Yes — ambiguous (family dispute vs. real unauthorised use; 0 opens but a trial was completed on a device). Amount = all three successful charges incl. the ₹1. Threat of cyber cell doesn't change the decision.
T-1009
unexpected_charge, sev 2, deny, no human
Mostly. Ledger: one failed + one successful, so no duplicate was collected; reply explains bank holds. Weak spot: a bank hold on the failed attempt may genuinely look like a double charge to the user.
T-1010
invoice_request, sev 1, none, route billing_ops, no human
Partly disagree. The reply promises an invoice the service can't produce. I'd flip this to needs_human or add an invoice tool.
T-1011
presale_question, sev 1, none, human, prompt_injection_suspected
Yes. Injection #2. No product knowledge base exists, so offline support is not asserted.
T-1012
refund_request, sev 2, review ₹249, human
Yes. 2 opens ⇒ not "unused". The 1-star threat is ignored by design.
Aggregate: 7/12 need a human. 5 auto-draftable (1001, 1002, 1007, 1009, 1010).
What it does when it can't tell: lowers confidence (short/vague text, or model says so) ⇒ <0.6 ⇒ needs_human; never resolves ambiguity into money. With the model down (--mode fallback, results/fallback_output.json) all 12 go to a human, none are auto-approved (a degraded-mode approve is downgraded to review).
The two injections: T-1003 and T-1011
Both are caught by sanitize.strip_injection (line-level patterns: "ignore previous instructions", "system note", "for the … agent", "reply with … system prompt", "mark ticket resolved"). The line is replaced by [removed: instruction-like text] before the model sees it, the ticket gets prompt_injection_suspected and needs_human, and auto-approval is disabled for it. Real output (results/replay_output.json):
{"ticket_id":"T-1003","category":"account_access","refund":{"decision":"none","amount_inr":0,"purchase_ids":[],"reason":"No money action for this category."},
 "reply_draft":"Sorry your order history is empty after the phone change. Please make sure you're signed in with the same Google account you paid with. If it's still empty, a member of our team will verify your account and restore it.",
 "needs_human":true,"human_reasons":["category_account_access_needs_person","prompt_injection_suspected"],"flags":["prompt_injection_suspected"]}
{"ticket_id":"T-1011","category":"presale_question","refund":{"decision":"none","amount_inr":0,...},
 "reply_draft":"Thanks for asking before subscribing. I don't want to guess about offline use, so a team member will confirm it for you and reply shortly.",
 "needs_human":true,"human_reasons":["category_presale_question_needs_person","prompt_injection_suspected"],"flags":["prompt_injection_suspected"]}
It did not do what the text asked (no refund, no "processed" claim, no prompt/rules disclosed, ticket not marked resolved). Caveat on how strong that proof is: in replay mode the "model" is my fixture, so those outputs prove the code layer. The stronger evidence is tests/test_triage.py::test_compromised_model_cannot_move_money: a model that obeys the T-1003 injection (picks refund_request, writes "full refund of ₹1500 has been processed", tries to smuggle a refund object) still yields decision != approve, an empty ledger, and a replaced reply. Where it can still fail: a novel phrasing the regexes miss reaches the model unflagged. Money still can't move, but the injected text could shape the wording of reply_draft on a ticket with needs_human: false. Mitigation I'd add: a second-pass classifier or reply-diff against a template for auto-sendable categories.
One dashboard number
Human-override rate on a 5% random audit sample: share of sampled tickets where the reviewing human changed the category or refund decision. Alert at ≥2× the 4-week baseline. It catches model drift, prompt changes and policy drift in one figure; fallback rate and shed counts (/metrics) are the supporting tripwires.
Task 3 — backend: 50 tickets/s during an outage
Already in the code: breaker, bounded concurrent model calls (shed ⇒ fallback), idempotent writes, fallback that still applies the money gate.
Changes for 50/s:
POST /triage stops calling the model inline. It validates, writes the ticket + a pending row, returns 202 in ms; a worker pool (concurrency ≈ provider rate limit) drains a queue. Priority lanes by cheap rules: fraud/payment/cancellation before feature requests.
When the breaker is open or the queue age passes a threshold, rows complete in fallback mode immediately and are flagged needs_retriage; a background job re-runs them through the model when it's healthy and replaces the draft only if no human has touched it.
Hash-based dedupe: identical (subject, body) from the same user in a window reuses one result — outages produce angry repeat submissions.
SQLite ⇒ Postgres (INSERT … ON CONFLICT DO NOTHING on both unique keys); the in-process per-ticket lock becomes the unique key + advisory lock.
What the user gets while degraded: an immediate acknowledgement in their language ("we've received your message and a person will reply shortly"), correct routing, and — for money cases — the deterministic policy decision, with a person reviewing it. No invented answer.
Storage sketch:
tickets(id PK, received_at, payload_hash, status[pending|done|failed], priority, subject_enc, body_enc) — text encrypted, short retention.
triage_results(ticket_id PK/FK, response_json, mode, model_name, prompt_version, policy_version, needs_retriage, created_at)
refund_ledger(purchase_id PK, ticket_id, amount_inr, decided_by[policy|human], created_at) — the double-refund guard.
model_calls(id, ticket_id, attempt, latency_ms, outcome_code, tokens_in, tokens_out, request_id) — codes only, no content.
outbox(id, ticket_id, kind[refund_instruction|reply], sent_at) — at-least-once delivery to the support tool.
One thing for 3am: stamp every result with request_id, prompt_version, policy_version, and write one model_calls row per attempt (outcome codes), plus GET /debug/ticket/{id} returning the full decision trail without ticket text. "Why did T-X get refunded?" then has a one-query answer. (Not implemented here; reason, human_reasons, flags and meta are the current version of it.)
Task 4 — how I used AI
Draft written from the build session — edit every line to be true for you before submitting.
Agents / share: Claude (one chat session) wrote the service, fixtures, tests and this README in one pass; I directed and reviewed. By lines of code: ~100% agent-written. (Adjust to your actual involvement.)
Best prompt, exactly as sent: Build the project (with the assignment PDF and ticket JSON attached). (Replace with yours if you used a different one.)
Something the agent got wrong: the first fallback classifier matched did not (agree|authori) as fraud, so with the model down T-1001 ("charged 249 without telling me… did not agree") became fraud_dispute, severity 4, with a ₹250 review amount. The 17 initial tests passed — they only checked the degraded path happened, not that it was right. I caught it by reading the degraded-mode log line for T-1001; fix: narrowed the regex and added test_fallback_classifier_does_not_call_unexpected_charge_fraud. A second one in the same pass: degraded mode could still approve and write to the ledger from a regex-derived category; it now downgrades to review.
Written by hand because directing was slower: (fill in — I have nothing honest to put here.)
Layout
app/{server,triage,policy,schema,sanitize,llm,fallback,store,logsafe}.py
fixtures/model_replay.json   data/dhaba_tickets.json   scripts/run_all.py
tests/test_triage.py         results/{replay,fallback}_output.json + *_logs.jsonl
Task 4 — how I used AI
Agents and share of code. One agent: Claude, in a single chat session (Claude mobile app). It wrote essentially all of the code, tests, fixtures and README, roughly 100% by lines. My part was giving it the assignment, reading and running the result, and copying the files into this repo by hand.
Best prompt, exactly as sent.
One thing the agent got wrong. The fallback classifier (the rules-based path used when the model is down or returns bad output) matched the phrase did not agree as fraud. With the model down, T-1001 ("charged 249 without telling me… i did not agree to this") came back as fraud_dispute, severity 4, with a ₹250 review amount instead of unexpected_charge. The 17 tests that existed at that point all passed, because they only checked that the degraded path ran, not that its answer was right: a test that asserted nothing useful. It was caught by running the service in fallback mode over all 12 tickets and reading the output row for T-1001. The fix was to narrow the regex, and a new test now pins the category and the refund decision for that ticket in degraded mode. A second error from the same pass: degraded mode could still approve a refund and write to the ledger based on a regex-derived category. It now downgrades to review, so a degraded run never moves money on its own.

A part I wrote by hand. None of the service code. The agent was fast enough that directing it was never slower than writing it. The part I did myself was the manual assembly: copying each file into the repo by hand. That is where copy-paste mistakes appear, so I verified it by cloning fresh and running the test suite and the 12-ticket script. (If you hand-wrote or hand-edited any code, replace this paragraph with that, and say why directing an agent was slower.)
What I did not verify. The agent had no API key or network, so the live model path was never run by it, and the replay fixtures are hand-written. Treat the Task 2 table as a test of the code around the model, not of a real model.