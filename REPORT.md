# NURA Edge — ADTC 2026 Technical Report

**Domain:** Healthcare & Medical (maternal-health triage, danger-sign recognition, patient education)
**Team:** NURA (INTELLIKAM) · Cameroon
**Model:** NURA-Edge-Qwen2.5-1.5B-Q4_K_M (llama.cpp, GGUF Q4_K_M)

---

## 1. Problem

Maternal complications and pregnancy loss are common and often dangerous, and in much of
Sub-Saharan Africa the warning signs appear in the gaps between clinic visits — where there is
no clinician on hand, and where the cost of running modern AI is out of reach. The intelligence
that could help is locked behind recurring cloud API fees, unreliable connectivity, and
intermittent power.

**Two users, one assistant.** NURA Edge serves both the pregnant or postpartum woman asking
directly ("I am 30 weeks pregnant and my vision is blurry this morning, should I be worried?")
and the community health worker asking on a woman's behalf ("my patient is soaking more than one
pad an hour after a miscarriage and has a fever, what should I do?"). The same assistant answers
both voices, recognising danger signs and directing the woman to care in time.

NURA Edge is the complete NURA maternal-health platform, re-engineered so its intelligence runs
on a small quantized model instead of an expensive cloud LLM. That change removes per-token API
fees and lets the whole system be hosted on far cheaper infrastructure — the real goal of this
work. Running fully offline for this challenge is the proof, not the point: it demonstrates the
model is small and cheap enough to run where it is needed most.

The full platform also includes GPS coordinate provision, automatic emergency calls to the
hospital when a patient is flagged urgent, clinician takeover when the AI needs a human, and
antenatal reminders. For this challenge we focus on the language model, as that is what the
evaluation measures.

---

## 2. Design Decisions

**Base model — Qwen2.5-1.5B-Instruct.** Chosen for its permissive Apache-2.0 licence, small
footprint (large headroom under the 8 GB RAM profile), and strong multilingual support including
French for the Francophone-African context.

**Model selection was evidence-driven.** We did not assume the smallest or largest model was
best — we measured. We profiled Qwen2.5-1.5B and Qwen2.5-3B (both Q4_K_M) on the same 8 GB
target hardware. The 1.5B scored *higher* on the accuracy benchmark (0.80 vs 0.72 acc_norm),
ran faster (17.5 vs 6.3 tokens/sec), and used roughly half the RAM (1.69 GB vs 3.27 GB). The 3B
offered no advantage on any scored dimension, so we selected the 1.5B on the evidence.

**Quantization — Q4_K_M.** We quantized to 4-bit Q4_K_M with llama.cpp, producing a 945 MB file.
We compared Q4_K_M against lighter Q4_0 and Q3_K_M on maternal-health danger-sign prompts; the
lighter quantizations degraded the model's unaided danger-sign handling (burying urgency,
drifting to generic advice), so we kept Q4_K_M, which leads with urgent referral and avoids
unsafe suggestions.

**RAG for grounded deployment.** A 1.5B model has weak factual recall, so in deployment NURA
Edge does not rely on it for clinical facts. An offline retrieval layer — a local
sentence-embedding model (all-MiniLM-L6-v2) with FAISS vector search — grounds each answer in a
clinician-approved maternal-health corpus, so the model reasons over retrieved guidance rather
than recalling medicine from its weights. This keeps memory low, improves accuracy in
deployment, and preserves NURA's safety promise: answers map to clinician-approved content. (The
ADTC benchmark evaluates the model file directly; the retrieval layer is part of the deployed
system described here.)

**African-context corpus.** Grounded in WHO and regional guidance, covering the danger signs
that matter most in Sub-Saharan Africa: pregnancy and postpartum danger signs, bleeding and
miscarriage, emotional support after loss, routine care and referral, malaria in pregnancy,
anaemia, obstructed and prolonged labour, and newborn danger signs (neonatal sepsis).

**Safety by construction.** Danger-sign escalation, "never confirm a loss," and "never
prescribe" are enforced in the prompt and in code, not left to the model, mirroring the
production NURA backend's deterministic red-flag logic.

---

## 3. Constraints

- **Hardware:** the ADTC 8 GB RAM / 4 vCPU / integrated-GPU laptop profile, 7 GB RAM ceiling.
  CPU-only inference (`-ngl 0`).
- **Connectivity and power:** zero external network at inference; model, embedder, index, and
  datastore are all local.
- **Runtime:** llama.cpp with GGUF weights only.
- **Data:** corpus limited to clinician-approved maternal-health guidance.
- **Two audiences:** answers must be safe and clear for both the woman and the health worker.

---

## 4. Benchmarks

Measured with the official ADTC profiler on real 8 GB target-class hardware (Intel Core
i5-1035G1, 8 GB RAM, no GPU, Ubuntu 24.04, CPU-only, bare-metal).

| Metric | Value | Notes |
|---|---|---|
| Model size on disk | 945 MB | Q4_K_M GGUF |
| Parameters | 1.54B | matches claimed 1.5B |
| Peak RAM (RSS) | 1,690 MB (1.69 GB) | ~24% of the 7 GB budget |
| Steady-state RAM | 1,611 MB | |
| Throughput (generation) | 17.48 tokens/sec | exceeds the 15 tok/s performance ceiling |
| First-token latency | 4,075 ms | |
| Local accuracy (arc_easy, 50 samples) | 0.80 acc_norm | general-reasoning benchmark |
| Thermal | throttled on test laptop | see note |

**Interpretation.** Peak RAM stayed near 1.69 GB throughout — about a quarter of the budget — so
efficiency headroom is large and there is no memory-disqualification risk. Throughput of 17.5
tok/s clears the profiler's 15 tok/s performance ceiling. On accuracy, the 1.5B (0.80)
outperformed the 3B (0.72) we also tested, confirming our model choice.

**Thermal note.** On this specific passively-cooled test laptop the CPU reached a brief 96°C
spike under the sustained accuracy benchmark and registered as throttled, despite thermald and
reduced swappiness. This is a property of the laptop's passive cooling under burst load, not of
the model — average CPU utilisation during the run was moderate (p99 ~66%). Data-center-class
audit hardware of the same 8 GB profile is not expected to throttle under the same load.

---

## 5. Relationship to the full NURA platform

NURA Edge is the complete NURA maternal-health platform — onboarding and risk scoring,
personalized tips and check-ins, danger-sign triage, GPS provision, automatic hospital emergency
calls, clinician takeover, and antenatal reminders — with its intelligence moved from an
expensive hosted LLM to a small local model. The grounding corpus, dual-audience voice, and
safety logic are drawn from the real system, so this build is a faithful version of a working
platform made affordable enough to run on cheap infrastructure — not a standalone prototype.
