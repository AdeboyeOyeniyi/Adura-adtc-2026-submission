# Technical Report — Adura Copilot: Offline PLC Program Generation

**Team ID:** enigma-labs-adura
**Domain:** coding_assistants
**Model:** Qwen3-8B-Q5_K_M

---

## Problem

More than 80% of the world's manufacturing capacity sits outside the Fortune 500. In Nigeria, Ghana, Kenya, Vietnam, and dozens of other high-growth markets, tens of thousands of small factories run on manual labour and ageing, uncontrolled machines — not because automation is unwanted, but because the price of entry is prohibitive.

A branded PLC system with HMI, sensors, and wiring costs $3,000–$15,000. Worse, programming it requires IEC 61131-3 certification, and certified PLC programmers are concentrated in Europe and North America. A factory owner in Lagos who wants to automate a bottling line has one realistic option: fly an integrator in. Most don't bother.

**Target user:** a factory owner or mechanical engineer who understands their process deeply but has never written a line of ladder logic. They can describe precisely what a machine should do; they cannot program a PLC to do it.

Adura Copilot converts plain-English process descriptions into validated IEC 61131-3 Structured Text programs, targeting sub-$250-per-node ESP32-based hardware. This submission covers the language model layer.

**Why offline matters here specifically.** Industrial sites in these markets have poor or intermittent connectivity — factory floors are frequently in industrial estates with unreliable internet, and many operate on generator power with scheduled outages. A cloud-dependent copilot fails exactly when an engineer is standing at a machine trying to commission it. Beyond connectivity, process descriptions encode proprietary manufacturing detail that many operators are unwilling to send to a third-party API. Local inference resolves both.

---

## Design Decisions

- **Base model: Qwen3-8B (Apache 2.0).** Initial development used Llama 3.1 8B Instruct. We migrated to Qwen3-8B for two independent reasons.

  First, **licensing.** The Llama 3.1 Community License permits redistribution but imposes obligations that propagate into the product: the derivative model must be named with a "Llama" prefix, "Built with Llama" must be displayed in related documentation, a specific NOTICE file must ship with the weights, and commercial use is capped at 700M MAU. Since this submission requires publicly hosted weights fetched by `download_model.sh`, any Llama derivative would carry those terms permanently. Qwen3's Apache 2.0 license has none of this — an explicit patent grant, no naming constraint, no scale threshold.

  Second, **output quality.** Direct comparison on our four evaluation prompts showed Qwen3 producing materially better Structured Text. Llama 3.1 generated IL-style colon labels (`R1:`) inside ST, invalid statements (`WAIT T1;`), and invalid function-block syntax (`TON T1 : TIM`, `T1.ENABLE`). Qwen3 produced correct named-parameter FB calls (`TON(T1, PT := T#3s, IN := FILLER_VALVE)`) and well-formed `IF/ELSIF/END_IF` blocks.

- **Quantization: Q5_K_M.** This was a measured decision, not a default. Q4_K_M — the obvious choice on memory grounds — produced a specific and reproducible failure: single stray tokens corrupting otherwise valid JSON. Two consecutive runs emitted `{ ""tag": "WD_KICK", ...` and `{ ","tag": "DI_4", ...` — a doubled quote and a spurious comma, each at the start of one `io_map` array element, in responses that were otherwise complete and structurally correct.

  Because the copilot's entire contract is machine-parseable structured output feeding a validator, a single stray character invalidates the whole response. Re-quantizing from the same fp16 GGUF at Q5_K_M eliminated the corruption across all four evaluation prompts.

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

**Decision: retain the 8B.** A 13% throughput gain does not offset three programs with incorrect control logic when accuracy carries 50% of the score against throughput's 30%. The experiment is documented here because the result was not obvious in advance and the reasoning is reusable.

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
- **No GPU acceleration.** Pure CPU inference via llama.cpp. This is the binding constraint on throughput: an 8B model at Q5_K_M generates at roughly 4 tokens/second on CPU, and no runtime configuration changes that materially.
- **Runtime flags are not ours to set.** The submission ships a GGUF and `metadata.json`; the evaluation harness controls thread count, context size, and batch parameters. Optimisations that depend on runtime flags (`--threads` tuning in particular) were therefore excluded from consideration — they cannot be expressed in the submission.
- **Structured output is the contract, not a nicety.** The copilot's output feeds a validator and ultimately generates code for machinery with moving parts. Malformed JSON is not a cosmetic defect; it is a hard failure. This constraint drove the quantization decision above and outweighs memory and speed considerations.
- **Connectivity and power.** Target deployment sites have intermittent internet and scheduled power outages, which is why full offline operation is a product requirement rather than a competition constraint we are accommodating.

---

## Benchmarks

| Metric | Value |
|---|---|
| Machine | AMD Ryzen 7 PRO 5850U, 62 GB RAM, Ubuntu 22.04.5 LTS |
| RAM at peak | 5,835 MB (steady state 5,631 MB) |
| Time to first token | 48,344 ms |
| Generation speed | 3.98 t/s |
| Thermal throttling | Observed — peak core temp 90.0 °C, throttling flagged |
| CPU utilisation (p99) | 99.4% |

Measured via `adtc-profiler run --mode participant --skip-accuracy`.

**On the thermal result.** The 85 °C threshold is exceeded and throttling is flagged, incurring the 10-point penalty. This was accepted deliberately rather than overlooked. Sustained ~99% CPU across a multi-minute generation will heat any laptop-class chip, and the levers that would reduce it — a smaller model, or lower quantization — cost accuracy, which carries five times the weight of the thermal penalty. Thread capping, the remaining option, is controlled by the evaluation harness rather than the submission.

**On throughput.** 3.98 t/s and a 48-second time to first token are poor in absolute terms and we do not expect to score well on this axis. We measured where the time actually goes rather than guessing: with the system prompt cached (`cache_n: 774`), a representative request processed only 46 new prompt tokens in 6.2 s and spent 194.3 s of a 200.9 s total in generation. Prompt engineering therefore offers no meaningful speed gain — the cost is token generation, which is a function of model size and CPU.

**Functional validation.** All four internal evaluation prompts (conveyor bottle filler, pump level hysteresis, mixer dual-valve interlock with timer, packaging line with three safety guards) return valid, parseable JSON with `status: "ok"` on Q5_K_M. The same four against Llama 3.1 8B Q4_K_M produced one unparseable response, one empty `program.source` despite `status: "ok"`, one inverted hysteresis logic bug, and pervasive invalid ST syntax.

These are self-reported development benchmarks. Official scores are measured by the ADTC profiler on the standard evaluation machine.

---

## Known Gaps

Documented rather than concealed, because they define the next iteration.

**Fine-tuning was specified but not executed.** A corrective dataset spec was derived from documented baseline failures, and the pipeline (QLoRA on fp16 base → merge → GGUF conversion → quantization) was planned. It was not completed within the schedule. The submitted model is the unmodified Qwen3-8B base at Q5_K_M. The A/B result above changed which model that work should target — see below.

**Two recurring logic defects persist in generated programs**, consistent across all four evaluation prompts:

1. *Order-dependent safety logic.* The model sets `WD_KICK := FALSE` inside the E_STOP guard rung, then unconditionally sets `WD_KICK := TRUE` in the final watchdog rung — silently defeating the interlock, because the later rung wins in scan order. Correct output requires the watchdog kick to be conditioned on E_STOP state.
2. *Undeclared variables.* Timer instances (`T1`, `Timer1`) are referenced without any `VAR` block declaring them, so generated programs are not compilable as-is.

Both are precisely the class of defect fine-tuning addresses, and both are already specified as corrective examples in the dataset spec.

**Throughput is not competitive.** An 8B model on CPU will not win on tokens/second, and at 5.8 GB it scores modestly on memory efficiency. Both are structural consequences of the model size we chose for output quality.

### Next iteration: fine-tune the 4B, not the 8B

The A/B above pointed the fine-tuning work at a different target than we started with. The 4B's failures divide usefully:

*Convention and format errors* — shadow-variable declarations instead of io_map tag binding, `//` comments, quoted `PROGRAM` names, `Timer.EN` in place of `Timer.IN`. The model produces consistent, structurally valid output and simply follows the wrong conventions. This is what corrective fine-tuning addresses most reliably.

*Reasoning errors* — the inverted mixer sequence and the missing hysteresis setpoint are wrong logic, not wrong formatting. These are harder, requiring the dataset to cover control patterns (dual-setpoint hysteresis, timed sequences with correct ordering, multi-input interlocks) rather than just output shape.

Since the majority of the 4B's gap is mechanical, and since smaller models generally recover proportionally more from targeted fine-tuning, a corrected 4B plausibly outperforms the current 8B across accuracy, throughput, memory, and thermals simultaneously. That is the next iteration: roughly 150–300 corrective examples derived from the failures documented in this report, trained via QLoRA against the fp16 base, then merged, converted, and quantized — retaining the fine-tune-before-quantize ordering, since quantizing first bakes in precision loss that training cannot recover.