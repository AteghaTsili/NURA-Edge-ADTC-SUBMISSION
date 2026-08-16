# NURA Edge — ADTC 2026 Technical Report

**Domain:** Healthcare & Medical (maternal-health triage, danger-sign recognition, patient education)
**Team:** NURA (INTELLIKAM) · Cameroon
**Model:** NURA-Edge-Qwen2.5-1.5B-Q4_K_M (llama.cpp, GGUF Q4_K_M)

---

## 1. Problem

Maternal complications and pregnancy loss are common and often dangerous, and in
much of Sub-Saharan Africa the warning signs appear in the gaps between clinic
visits — where there is no clinician on hand, and where the cost of running modern
AI is out of reach. The intelligence that could help is locked behind recurring
cloud API fees, unreliable connectivity, and intermittent power.

**Two users, one assistant.** NURA Edge serves both:

- The pregnant or postpartum woman, asking directly — "I am 30 weeks pregnant and
  my vision is blurry this morning, should I be worried?"
- The community health worker, asking on a woman's behalf — "my patient is soaking
  more than one pad an hour after a miscarriage and has a fever, what should I do?"

The same assistant answers both voices, recognising danger signs and directing the
woman to care in time.

NURA Edge is the complete NURA maternal-health platform, re-engineered so its
intelligence runs on a small quantized model instead of an expensive cloud LLM.
That change removes per-token API fees and lets the whole system be hosted and
managed on far cheaper infrastructure — the real goal of this work. Running fully
offline for this challenge is the proof, not the point: it demonstrates the model
is small and cheap enough to run where it is needed most.

The full platform also includes GPS coordinate provision, automatic emergency
calls to the hospital when a patient is flagged as urgent, clinician takeover when
the AI needs a human, and antenatal reminders. For this challenge we demonstrate
the functionalities that directly involve the language model — triage, tips,
check-ins, and the patient journey — because those are what the evaluation
measures.

---

## 2. Design Decisions

**Base model — Qwen2.5-1.5B-Instruct.** Chosen for its permissive Apache-2.0
licence, small footprint (large headroom under the 8 GB RAM profile), and strong
multilingual support including French for the Francophone-African context.

**Quantization — Q4_K_M.** We quantized the model to 4-bit Q4_K_M using llama.cpp,
producing a **945 MB** file — roughly a 3× reduction from the F16 GGUF. We compared
Q4_K_M against the smaller Q4_0 and Q3_K_M directly on our own maternal-health
danger-sign prompts. The lighter quantizations degraded the model's unaided
danger-sign handling — on a postpartum-haemorrhage case, lower-bit versions buried
the urgency and drifted toward generic advice — so we kept Q4_K_M, where the model
leads with urgent referral and avoids unsafe suggestions. On a clinical tool the
small size saving was not worth weaker danger-sign behaviour.

**RAG for grounded deployment.** A 1.5B model has weak factual recall, so in
deployment NURA Edge does not rely on it for clinical facts. An offline retrieval
layer — a local sentence-embedding model (all-MiniLM-L6-v2) with FAISS vector
search — grounds each answer in a clinician-approved maternal-health corpus, so the
model reasons over retrieved guidance rather than recalling medicine from its
weights. This keeps memory low, improves accuracy, and preserves NURA's safety
promise: answers map to clinician-approved content. (The ADTC benchmark evaluates
the model file directly; the retrieval layer is part of the deployed system
described here.)

**African-context corpus.** The corpus is grounded in WHO and regional guidance and
covers the danger signs that matter most in Sub-Saharan Africa: pregnancy and
postpartum danger signs, bleeding and miscarriage, emotional support after loss,
routine care and referral, and — added for regional relevance — malaria in
pregnancy, anaemia, obstructed and prolonged labour, and newborn danger signs
(neonatal sepsis).

**Safety by construction.** Danger-sign escalation, "never confirm a loss," and
"never prescribe" are enforced in the prompt and in code, not left to the model,
mirroring the production NURA backend's deterministic red-flag logic.

---

## 3. Constraints

- **Hardware:** the ADTC 8 GB RAM / 4 vCPU / integrated-GPU laptop profile, with a
  strict 7 GB RAM ceiling. We designed for CPU-only inference (`-ngl 0`).
- **Connectivity and power:** zero external network access during inference; the
  model, embedder, retrieval index, and datastore are all local.
- **Runtime:** llama.cpp with GGUF weights only.
- **Data:** the corpus is limited to clinician-approved maternal-health guidance —
  the safety guarantee depends on grounding only in vetted content.
- **Two audiences:** answers must be safe and clear whether the reader is the woman
  herself or a health worker, without assuming clinical training.

---

## 4. Benchmarks

Measured with the official ADTC profiler on real 8 GB target-class hardware
(Intel Core i5-1035G1, 8 GB RAM, no GPU, Ubuntu 24.04, CPU-only).

| Metric | Value | Notes |
|---|---|---|
| Model size on disk | 945 MB | Q4_K_M GGUF |
| Parameters | 1.54B | matches claimed 1.5B |
| Peak RAM (RSS) | 1,688 MB (1.65 GB) | far under the 7 GB ceiling |
| Steady-state RAM | 1,605 MB | |
| Throughput (generation) | 9.37 tokens/sec | on a passively-cooled 8 GB laptop |
| First-token latency | 8,855 ms | |
| CPU utilization (p99) | 97.8% | |
| Thermal throttling | yes (this laptop) | see note below |
| Local accuracy (arc_easy, 50 samples) | 0.80 acc_norm | general-reasoning self-check |

**Interpretation.** Peak RAM stayed near 1.65 GB throughout — about a quarter of the
budget — so efficiency headroom is large and disqualification on memory is not a
risk. On this particular passively-cooled laptop the CPU reached ~98°C and throttled
under sustained benchmarking; on better-cooled hardware of the same class (and on a
higher-spec machine we also tested) no throttling occurs and throughput is higher
(~18–26 tokens/sec). Domain accuracy in evaluation comes from the model's answers
to maternal-health prompts; we selected Q4_K_M specifically because it handles
danger-sign prompts more safely than lighter quantizations.

---

## 5. Relationship to the full NURA platform

NURA Edge is the complete NURA maternal-health platform — patient onboarding and
risk scoring, personalized tips and check-ins, danger-sign triage, GPS provision,
automatic hospital emergency calls, clinician takeover, and antenatal reminders —
with its intelligence moved from an expensive hosted LLM to a small local model.
The grounding corpus, the dual-audience voice, and the safety logic are drawn from
the real system, so this build is a faithful version of a working platform made
affordable enough to run on cheap infrastructure — not a standalone prototype.
