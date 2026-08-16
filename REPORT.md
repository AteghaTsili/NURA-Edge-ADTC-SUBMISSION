# NURA Edge — ADTC 2026 Technical Report

**Domain:** Healthcare & Medical (maternal-health triage, danger-sign recognition, patient education)
**Team:** NURA (INTELLIKAM) · Cameroon
**Model:** NURA-Edge-Qwen2.5-1.5B-Q3_K_M (llama.cpp, GGUF Q3_K_M)

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

The same assistant answers both voices safely, recognising danger signs and
directing the woman to care in time.

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

**Quantization — Q3_K_M.** We quantized the model to Q3_K_M GGUF using llama.cpp,
producing an **~824 MB** file. We compared Q4_K_M, Q4_0, and Q3_K_M directly on our
own maternal-health prompts: Q3_K_M was smaller and faster while, once combined
with retrieval grounding, preserving clinical answer quality on danger-sign cases
(pre-eclampsia, postpartum haemorrhage, neonatal sepsis). Because retrieval — not
the model's weights — carries the clinical facts, the lower bit-width did not
degrade the grounded answers, so we adopted it for the efficiency and throughput
gains.

**RAG instead of a larger or fine-tuned model.** A 1.5B model has weak factual
recall, so we do not rely on it for clinical facts. An offline retrieval layer — a
local sentence-embedding model (all-MiniLM-L6-v2) with FAISS vector search —
grounds each answer in a clinician-approved maternal-health corpus. The model
reasons over retrieved guidance and phrases a safe reply rather than recalling
medicine from its weights. This keeps memory low, improves accuracy, and preserves
NURA's core safety promise: answers map to clinician-approved content rather than
being invented. Retrieval measurably corrected drift we observed in the raw model
on newborn danger signs, which is why the pairing is load-bearing.

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
| Model size on disk | ~824 MB | Q3_K_M GGUF |
| Parameters | 1.54B | matches claimed 1.5B |
| Peak RAM (RSS) | under 1.7 GB | far under the 7 GB ceiling |
| Throughput (generation) | ~21 tokens/sec (native, unconstrained); ~9–11 tokens/sec on a passively-cooled 8 GB laptop | CPU-only |
| Local accuracy (arc_easy, 50 samples) | ~0.80 acc_norm | general-reasoning self-check |

Note: on a passively-cooled 8 GB laptop the CPU reached high temperatures and
throttled under sustained benchmarking; peak RAM stayed near 1.6–1.7 GB throughout.
On better-cooled hardware of the same class, throughput is higher and no throttling
occurs. Domain accuracy in evaluation comes from the model's grounded answers to
maternal-health prompts, where RAG retrieval over the clinician-approved corpus is
the primary driver of correctness and safety, for both the woman's voice and the
health worker's.

---

## 5. Relationship to the full NURA platform

NURA Edge is the complete NURA maternal-health platform — patient onboarding and
risk scoring, personalized tips and check-ins, danger-sign triage, GPS provision,
automatic hospital emergency calls, clinician takeover, and antenatal reminders —
with its intelligence moved from an expensive hosted LLM to a small local model.
The grounding corpus, the dual-audience voice, and the safety logic are drawn from
the real system, so this build is a faithful version of a working platform made
affordable enough to run on cheap infrastructure — not a standalone prototype.
