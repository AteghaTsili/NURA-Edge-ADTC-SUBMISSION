# NURA Edge — ADTC 2026 Technical Report

**Domain:** Healthcare & Medical (maternal-health triage, danger-sign recognition, patient education)
**Team:** INTELLIKAM · Cameroon
**Model:** NURA-Edge-Qwen2.5-1.5B-Q4_K_M (llama.cpp, GGUF Q4_K_M)

---

## 1. Problem

Maternal complications and pregnancy loss are common and often dangerous, and in
much of Sub-Saharan Africa the warning signs appear in the gaps between clinic
visits — where there is no clinician on hand. Cloud AI that could help is blocked
by three barriers at once: recurring API fees, unreliable connectivity, and
intermittent power. A pregnant woman in a rural area, or the community health
worker (CHW) supporting her, cannot depend on a system that needs stable internet
and a monthly subscription to answer an urgent question.

**Two users, one assistant.** NURA Edge is built to serve both:

- **The pregnant or postpartum woman**, asking directly in plain language — "I am
  30 weeks pregnant and my vision is blurry this morning, should I be worried?"
- **The community health worker**, asking on a woman's behalf — "my patient is
  soaking more than one pad an hour after a miscarriage and has a fever, what
  should I do?"

The same offline assistant answers both voices safely, recognising danger signs
and directing the woman to care in time. This mirrors the production NURA
platform, where women interact by SMS or app and health workers act as a
supported bridge to the clinic.

NURA Edge is the offline clinical core of that platform. It runs on hardware a
clinic or worker already owns, with no cloud, no API fees, and no internet during
operation. The offline benchmark is not the goal in itself — it is proof that the
model is small and cheap enough to remove the hosting and connectivity barriers
that keep AI out of frontline African maternal care.

---

## 2. Design Decisions

**Base model — Qwen2.5-1.5B-Instruct.** We chose a 1.5B instruction-tuned model
for three reasons: its Apache-2.0 licence is cleanly permissive for a competition
submission; its small size leaves large headroom under the 8 GB RAM profile,
directly improving the efficiency score; and Qwen's strong multilingual support
(including French) suits the Cameroonian and wider Francophone-African context.

**Quantization — Q4_K_M.** We quantized to 4-bit Q4_K_M using llama.cpp,
reducing the model from a ~3.1 GB F16 GGUF to **945 MB** — roughly a 3× reduction
— while preserving answer quality. We evaluated the trade-off against lighter
(Q3, Q2) and heavier (Q5, Q8) quantizations: lower bit-widths risked degrading
clinical accuracy (a safety concern), while higher ones gave no benefit given our
RAM headroom. Q4_K_M is the standard sweet spot.

**Dual-audience prompt.** The assistant's system prompt is written to answer both
the woman herself and a health worker asking on her behalf, in simple reassuring
language either can act on. This matters for real use — the same danger sign may
be reported as "I have..." or "my patient has..." — and it reflects how the
production platform works, where the prompts and clinical logic in this build are
drawn verbatim from the real NURA backend.

**RAG instead of a larger or fine-tuned model.** A 1.5B model has weak factual
recall, so we do not rely on it for clinical facts. An offline retrieval layer —
a local sentence-embedding model (all-MiniLM-L6-v2) with FAISS vector search —
grounds each answer in a clinician-approved maternal-health corpus. The model's
job is to reason over retrieved guidance and phrase a safe reply, not to recall
medicine from its weights. This keeps RAM low, improves accuracy, and preserves
NURA's core safety promise: answers map to clinician-approved content rather than
being invented. We chose grounding over fine-tuning because fine-tuning a small
model on medical text risks introducing hallucinated clinical claims — a safety
hazard — whereas retrieval keeps answers tied to vetted guidance.

**African-context corpus.** The corpus is grounded in WHO and regional guidance
and covers the danger signs that matter most in Sub-Saharan Africa: pregnancy and
postpartum danger signs, bleeding and miscarriage, emotional support after loss,
routine care and referral, and — added for regional relevance — malaria in
pregnancy, anaemia, obstructed and prolonged labour, and newborn danger signs
(neonatal sepsis). These reflect leading regional causes of maternal and newborn
death.

**Safety by construction.** Rules a small model does not reliably follow are
enforced in the prompt and in code rather than trusted to the model: danger signs
trigger an emergency rule (the reply begins directly with the instruction to get
to a facility now), the assistant never confirms a pregnancy loss (only a
clinician can), it never prescribes medicines or doses, and it grounds answers
only in the retrieved corpus. This mirrors the production NURA backend, which
enforces its red-flag triage deterministically rather than relying on the LLM.

---

## 3. Constraints

- **Hardware:** the ADTC 8 GB RAM / 4 vCPU / integrated-GPU laptop profile, with a
  strict 7 GB RAM ceiling. We designed for CPU-only inference (`-ngl 0`) to match
  the target and the reality of the machines a clinic or worker actually owns.
- **Connectivity and power:** zero external network access during inference. The
  model, embedding model, retrieval index, and datastore are all local; the only
  online step is one-time setup.
- **Runtime:** llama.cpp with GGUF weights only, as required.
- **Data:** the corpus is limited to clinician-approved maternal-health guidance —
  the safety guarantee depends on grounding only in vetted content.
- **Two audiences:** answers must be safe and clear whether the reader is the
  woman herself or a health worker, without assuming clinical training.

---

## 4. Benchmarks

Measured with the official ADTC profiler (`adtc-profiler run --mode participant`).

**Development machine:** Intel Core i7-9750H @ 2.60 GHz, 16 GB RAM, integrated GPU
only, macOS. This is more capable than the 8 GB target laptop, so evaluation
numbers will be more conservative; our large RAM headroom means we remain well
within limits regardless.

| Metric | Value | Notes |
|---|---|---|
| Model size on disk | 945 MB | Q4_K_M GGUF |
| Parameters | 1.54B | matches claimed 1.5B |
| Peak RAM (RSS) | 1,705 MB (1.7 GB) | far under the 7 GB ceiling |
| Steady-state RAM | 1,615 MB | |
| Throughput (generation) | 26.3 tokens/sec | above the 15.0 reference |
| First-token latency | 2,830 ms | |
| CPU utilization (p99) | 70.5% | |
| Thermal throttling | none | no penalty |
| Local accuracy (arc_easy, 50 samples) | 0.78 acc_norm | general-reasoning self-check |

**Interpretation.** Throughput exceeds the reference threshold, and peak RAM uses
about a quarter of the budget, leaving substantial headroom on weaker hardware.
No thermal throttling was observed. The local accuracy self-check (arc_easy)
indicates solid general reasoning; domain accuracy in evaluation comes from the
model's grounded answers to maternal-health prompts — where RAG retrieval over the
clinician-approved corpus is the primary driver of correctness and safety, for
both the woman's voice and the health worker's.

---

## 5. Relationship to the full NURA platform

NURA Edge is the offline brain of NURA, a production maternal-health platform
serving pregnant women by SMS and app with community health workers and clinicians
in support (patient onboarding and risk scoring, personalized tips and check-ins,
danger-sign triage, clinician takeover, hospital alerts, scheduled care). The
platform normally uses a hosted LLM; NURA Edge is that same brain, quantized to
run locally, removing per-token fees and connectivity requirements. The grounding
corpus, the dual-audience voice, and the safety logic are drawn from the real
system, so this offline build is a faithful distillation of a working platform
rather than a standalone prototype.
