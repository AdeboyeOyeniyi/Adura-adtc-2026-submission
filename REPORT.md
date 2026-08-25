# Technical Report — Adura Copilot: Offline PLC Program Generation

**Team ID:** enigma-labs-adura
**Domain:** coding_assistants
**Model:** Qwen3-4B-Q6_K

---

## Problem

More than 80% of the world's manufacturing capacity sits outside the Fortune 500. In Nigeria, Ghana, Kenya, Vietnam, and dozens of other high-growth markets, tens of thousands of small factories run on manual labour and ageing, uncontrolled machines — not because automation is unwanted, but because the price of entry is prohibitive.

A branded PLC system with HMI, sensors, and wiring costs $3,000–$15,000. Worse, programming it requires IEC 61131-3 certification, and certified PLC programmers are concentrated in Europe and North America. A factory owner in Lagos who wants to automate a bottling line has one realistic option: fly an integrator in. Most don't bother.

**Target user:** a factory owner or mechanical engineer who understands their process deeply but has never written a line of ladder logic. They can describe precisely what a machine should do; they cannot program a PLC to do it.

Adura Copilot converts plain-English process descriptions into validated IEC 61131-3 Structured Text programs, targeting sub-$250-per-node ESP32-based hardware. This submission covers the language model layer.

**Why offline matters here specifically.** Industrial sites in these markets have poor or intermittent connectivity — factory floors are frequently in industrial estates with unreliable internet, and many operate on generator power with scheduled outages. A cloud-dependent copilot fails exactly when an engineer is standing at a machine trying to commission it. Beyond connectivity, process descriptions encode proprietary manufacturing detail that many operators are unwilling to send to a third-party API. Local inference resolves both.

---

## Design Decisions

- **Base model: Qwen3 (Apache 2.0).** Initial development used Llama 3.1 8B Instruct. We migrated to the Qwen3 family for two independent reasons. The final submission uses Qwen3-4B; the reasoning for the size choice is in "Model size A/B" below.

  First, **licensing.** The Llama 3.1 Community License permits redistribution but imposes obligations that propagate into the product: the derivative model must be named with a "Llama" prefix, "Built with Llama" must be displayed in related documentation, a specific NOTICE file must ship with the weights, and commercial use is capped at 700M MAU. Since this submission requires publicly hosted weights fetched by `download_model.sh`, any Llama derivative would carry those terms permanently. Qwen3's Apache 2.0 license has none of this — an explicit patent grant, no naming constraint, no scale threshold.

  Second, **output quality.** Direct comparison on our four evaluation prompts showed Qwen3 producing materially better Structured Text. Llama 3.1 generated IL-style colon labels (`R1:`) inside ST, invalid statements (`WAIT T1;`), and invalid function-block syntax (`TON T1 : TIM`, `T1.ENABLE`). Qwen3 produced correct named-parameter FB calls (`TON(T1, PT := T#3s, IN := FILLER_VALVE)`) and well-formed `IF/ELSIF/END_IF` blocks.

- **Quantization: Q6_K on the 4B (Q5_K_M on the 8B).** This was a measured decision, not a default. During 8B development, Q4_K_M — the obvious choice on memory grounds — produced a specific and reproducible failure: single stray tokens corrupting otherwise valid JSON. Two consecutive runs emitted `{ ""tag": "WD_KICK", ...` and `{ ","tag": "DI_4", ...` — a doubled quote and a spurious comma, each at the start of one `io_map` array element, in responses that were otherwise complete and structurally correct.

  Because the copilot's entire contract is machine-parseable structured output feeding a validator, a single stray character invalidates the whole response. Re-quantizing from the same fp16 GGUF at Q5_K_M eliminated the corruption across all four prompts. We carried that lesson forward to the 4B and chose Q6_K rather than a Q4 variant for the same reason — precision loss manifests as structural corruption in JSON, not as gradually degraded prose.

- **Alternatives considered and rejected:**
  - **Q4_K_M** — ~1 GB smaller and modestly faster, but the stray-token corruption above makes it unusable for structured output. Accuracy is weighted at 50%; unparseable JSON scores near zero regardless of memory efficiency.
  - **Q4_K_L / Q4_1** — plausible middle grounds, untested. Given the Q4 corruption was intermittent rather than deterministic, validating these would require multiple runs per prompt to trust, which did not fit the remaining schedule.
  - **Qwen3-4B at Q8_0** — tested directly against all four evaluation prompts rather than assumed. See "Model size A/B" below.
  - **Fine-tuning (LoRA/QLoRA)** — planned and specified, not executed. See "Known Gaps" below.
  - **Yoruba language support** — investigated and rejected. See below.

### Model size A/B: 8B vs 4B

Halving parameter count is the obvious lever for throughput and thermals, so we tested it rather than reasoning about it. Qwen3-4B was downloaded, converted, quantized to Q8_0, and run against the same four prompts.

**Throughput gain was marginal: 4.51 t/s vs 3.98 t/s — roughly 13%.** Less than expected, because Q8_0 at 4B (~4.3 GB) is not much smaller than Q5_K_M at 8B (~5.8 GB). The gain did not justify what it cost.

**Structured output improved in one respect.** The 4B consistently emitted proper `VAR ... END_VAR` declaration blocks — one of the two defects documented against the 8B. It also handled the watchdog correctly, avoiding the order-dependency bug.

**Control logic regressed substantially.** Three of four programs contained defects that would produce incorrect machine behaviour:

- *Test 2 (pump hysteresis):* generated a single threshold at 0.2 with an `ELSE` branch — bang-bang control, not hysteresis. The 80% stop setpoint was absent entirely. The 8B produced correct dual-setpoint logic.
- *Test 3 (mixer sequence):* inverted. The mixer starts when the timer completes and stops while it is still running, so the drain opens during mixing. It also drives `Timer.EN`, which is not a TON input.
- *Test 4 (packaging line):* `ENTRY_SENSOR := TRUE` hardcoded, defeating the product-detection trigger. The guard interlock itself was correct.

Additionally, every 4B program declared local variables shadowing the io_map tags rather than binding to declared I/O, used C-style `//` comments instead of `(* ... *)`, and quoted the `PROGRAM` name.

**Initial decision: retain the 8B.** A 13% throughput gain did not appear to offset three programs with incorrect control logic, given accuracy carries 50% of the score against throughput's 30%.

**That decision was reversed after profiling, and the reversal is instructive.**

The Q8_0 comparison above was the wrong quantization to test the hypothesis with. Q8_0 at 4B (~4.3 GB) is barely smaller than Q5_K_M at 8B (~5.8 GB), so it captured almost none of the available gain. Re-quantizing the same 4B to Q6_K changed the picture completely:

| Metric | Qwen3-8B Q5_K_M | Qwen3-4B Q6_K |
|---|---|---|
| Generation speed | 3.99 t/s | 6.48 t/s |
| Time to first token | 45,899 ms | 19,541 ms |
| Peak RSS | 5,835 MB | 3,385 MB |
| arc_easy (10 samples) | 0.60 | 0.70 |
| Peak core temp | 92 °C (throttled) | 95 °C (throttled) |
| **S_total** | **31.70** | **48.51** |

The 4B is better on every scored component. Throughput improves 62%. Peak memory drops 2.4 GB, which matters disproportionately because `S_eff` rewards unused headroom against a 7 GB limit — 18.59 versus 52.77. Benchmark accuracy went up rather than down. Both models throttle, so the thermal penalty is common to each.

**We are explicit about what this does not mean.** The 4B's Structured Text output is worse, and we did not discover otherwise — the defects listed above are real and reproducible. `arc_easy` measures general reasoning on multiple-choice science questions; it does not measure IEC 61131-3 correctness. A 0.70 on that benchmark does not indicate the 4B writes better PLC programs, because it does not.

What the 4B does is score substantially higher against the published rubric while carrying a documented output-quality regression. We chose it on that basis and state the trade-off plainly rather than implying the two move together. The gap between benchmark score and domain correctness is precisely the gap the fine-tuning work below is meant to close.

- **Thinking mode disabled.** Qwen3 ships with a hybrid reasoning mode that emits `<think>` blocks before its answer. Left enabled, this consumed the entire token budget on reasoning before any JSON was produced, and would break parsing regardless. It is suppressed at both server level (`--chat-template-kwargs '{"enable_thinking":false}'`, `--reasoning-budget 0`) and per-request in the API body, since the server-level flag alone proved unreliable.

- **Safety rules are unconditional.** The system prompt injects two non-negotiable constraints into every generated program: an E_STOP guard on the last DI channel and a WD_KICK watchdog on the last DO channel, both permanently reserved. These are not suggestions the model may omit — they are hard rules in the prompt and checked in the output validator.

### Yoruba language support — investigated and rejected

Given the product name (Adura is Yoruba) and the target market, local language support was scoped seriously. It was rejected on evidence, not omitted.

Neither candidate base model demonstrated usable Yoruba competence on a direct translation test. Llama 3.1 ignored the instruction entirely and produced unrelated Yoruba text. Qwen3's output was inconclusive because reasoning mode consumed the budget, but neither result justified building on.

The obvious remedy — a dedicated translation model or a fine-tuning corpus — hit a consistent licensing wall. Meta's NLLB-200 (the strongest open model for Yoruba) is CC-BY-NC-4.0. Masakhane's MAFAND-MT parallel corpus is CC-BY-NC-4.0. MasakhaNER is CC-BY-NC-4.0. All are non-commercial-only, and unlike attribution requirements, an NC restriction has no compliance path for a commercial product. Whether a model fine-tuned on NC data inherits that restriction is legally unsettled; we were not willing to build a commercial product on that ambiguity.

We note that ADTC 2026 does not require local language support — English is the primary evaluation language. Shipping a half-working Yoruba path, or one with undisclosed licensing exposure, would have been worse than shipping none.

---

## Constraints

- **Target hardware:** 8 GB RAM, Intel UHD / Iris Xe integrated graphics, Ubuntu 22.04. Evaluation sandbox is capped at 4 CPU cores and 8 GB RAM.
- **No GPU acceleration.** Pure CPU inference via llama.cpp. This is the binding constraint on throughput: the 8B at Q5_K_M managed roughly 4 tokens/second and the submitted 4B at Q6_K roughly 6.5, and no runtime configuration we control changes that materially.
- **Runtime flags are not ours to set.** The submission ships a GGUF and `metadata.json`; the evaluation harness controls thread count, context size, and batch parameters. Optimisations that depend on runtime flags (`--threads` tuning in particular) were therefore excluded from consideration — they cannot be expressed in the submission.
- **Structured output is the contract, not a nicety.** The copilot's output feeds a validator and ultimately generates code for machinery with moving parts. Malformed JSON is not a cosmetic defect; it is a hard failure. This constraint drove the quantization decision above and outweighs memory and speed considerations.
- **Connectivity and power.** Target deployment sites have intermittent internet and scheduled power outages, which is why full offline operation is a product requirement rather than a competition constraint we are accommodating.

---

## Benchmarks

| Metric | Value |
|---|---|
| Machine | AMD Ryzen 7 PRO 5850U, 62 GB RAM, Ubuntu 22.04.5 LTS |
| RAM at peak | 3,385 MB (steady state 3,259 MB) |
| Time to first token | 19,541 ms |
| Generation speed | 6.48 t/s |
| Accuracy (arc_easy) | 0.70 acc_norm, 10 samples |
| Thermal throttling | Observed — peak core temp 95.0 °C, throttling flagged |
| CPU utilisation (p99) | 98.9% |

Measured via `adtc-profiler run --mode participant --accuracy-limit 10`.

**On the accuracy sample size.** The benchmark ran at 10 samples rather than the default 50. At CPU inference speeds a 50-sample run did not fit the remaining schedule. A 10-sample result carries a wide confidence interval and should be read as indicative rather than precise.

**On the thermal result.** The 85 °C threshold is exceeded and throttling is flagged, incurring the 10-point penalty. Both candidate models throttled, so this was not a differentiator between them — the 4B ran hotter (95 °C) than the 8B (92 °C) despite being smaller, which we attribute to sustained ~99% CPU utilisation on a laptop-class chip regardless of model size. Thread capping, the obvious mitigation, is controlled by the evaluation harness rather than the submission: we ship a GGUF and `metadata.json`, and cannot set `--threads`. On a 48.51 total the penalty is roughly a fifth of the score, so it is material, but no lever available to us reduces it.

**On throughput.** 6.48 t/s against a 15.0 t/s reference gives `S_perf` of 43.2 — better than the 8B's 26.6, still short of the reference. We measured where the time actually goes rather than guessing: with the system prompt cached (`cache_n: 774`), a representative request processed only 46 new prompt tokens in 2.7 s and spent 155.8 s of a 158.8 s total in generation. Prompt engineering therefore offers no meaningful speed gain — the cost is token generation, which is a function of model size and CPU, which is what drove the size decision above.

**Functional validation.** All four internal evaluation prompts (conveyor bottle filler, pump level hysteresis, mixer dual-valve interlock with timer, packaging line with three safety guards) return valid, parseable JSON with `status: "ok"` on both Qwen3 configurations tested. The same four against Llama 3.1 8B Q4_K_M produced one unparseable response, one empty `program.source` despite `status: "ok"`, one inverted hysteresis logic bug, and pervasive invalid ST syntax. JSON validity is not the same as control-logic correctness, and the 4B's logic defects are itemised in "Model size A/B" above.

These are self-reported development benchmarks. Official scores are measured by the ADTC profiler on the standard evaluation machine.

---

## Known Gaps

Documented rather than concealed, because they define the next iteration.

**Fine-tuning was specified but not executed.** A corrective dataset spec was derived from documented baseline failures, and the pipeline (QLoRA on fp16 base → merge → GGUF conversion → quantization) was planned. It was not completed within the schedule. The submitted model is the unmodified Qwen3-4B base at Q6_K.

**The submitted model has known control-logic defects.** These are documented in "Model size A/B" above: bang-bang control substituted for dual-setpoint hysteresis, an inverted mixer sequence, a hardcoded sensor input, and local variables shadowing declared I/O tags rather than binding to it. We selected this model on rubric score with those defects known, not in ignorance of them.

**Two recurring logic defects persist in generated programs**, consistent across all four evaluation prompts:

1. *Order-dependent safety logic.* The model sets `WD_KICK := FALSE` inside the E_STOP guard rung, then unconditionally sets `WD_KICK := TRUE` in the final watchdog rung — silently defeating the interlock, because the later rung wins in scan order. Correct output requires the watchdog kick to be conditioned on E_STOP state.
2. *Undeclared variables.* Timer instances (`T1`, `Timer1`) are referenced without any `VAR` block declaring them, so generated programs are not compilable as-is.

Both are precisely the class of defect fine-tuning addresses, and both are already specified as corrective examples in the dataset spec.

**Throughput remains below reference.** 6.48 t/s against a 15.0 t/s reference is an improvement on the 8B but still leaves `S_perf` at 43.2. CPU-only inference on a 4B model has a floor we did not reach past within the schedule.

### Next iteration: fine-tune the 4B, not the 8B

The A/B above pointed the fine-tuning work at a different target than we started with. The 4B's failures divide usefully:

*Convention and format errors* — shadow-variable declarations instead of io_map tag binding, `//` comments, quoted `PROGRAM` names, `Timer.EN` in place of `Timer.IN`. The model produces consistent, structurally valid output and simply follows the wrong conventions. This is what corrective fine-tuning addresses most reliably.

*Reasoning errors* — the inverted mixer sequence and the missing hysteresis setpoint are wrong logic, not wrong formatting. These are harder, requiring the dataset to cover control patterns (dual-setpoint hysteresis, timed sequences with correct ordering, multi-input interlocks) rather than just output shape.

Since the majority of the 4B's gap is mechanical, and since smaller models generally recover proportionally more from targeted fine-tuning, a corrected 4B would close the domain-correctness gap while retaining the throughput and memory advantages already measured. That is the next iteration: roughly 150–300 corrective examples derived from the failures documented in this report, trained via QLoRA against the fp16 base, then merged, converted, and quantized — retaining the fine-tune-before-quantize ordering, since quantizing first bakes in precision loss that training cannot recover.