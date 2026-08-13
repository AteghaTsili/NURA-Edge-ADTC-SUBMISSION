# NURA Edge — ADTC 2026 Technical Report

**Domain:** Healthcare & Medical (clinical Q&A, triage support, patient education)
**Team:** INTELLIKAM · Cameroon
**Model:** NURA-Edge-Qwen2.5-1.5B-Q4_K_M (llama.cpp, GGUF Q4_K_M)

---

## 1. Problem

Maternal complications and pregnancy loss are common and often dangerous, and in
much of Sub-Saharan Africa the warning signs appear in the gaps between clinic
visits, where there is no clinician on hand. Cloud AI that could help is blocked
by the same three barriers everywhere: recurring API fees, unreliable
connectivity, and intermittent power. A community health worker at a rural health
post cannot depend on a system that needs stable internet and a monthly
subscription to answer a triage question.

**Target user:** a community health worker (CHW) serving pregnant women in a
low-connectivity African setting — for example a rural health area in Cameroon or
the wider region — equipped with a modest laptop and no reliable internet or power.

NURA Edge is the offline clinical core of the NURA maternal-health platform. It
runs on the hardware a clinic already owns and answers maternal-triage and
patient-education questions with no cloud, no API fees, and no internet during
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

**Quantization — Q4_K_M.** We quantized the model to 4-bit Q4_K_M using
llama.cpp's `llama-quantize`. This reduced the model from a ~3.1 GB F16 GGUF to a
**945 MB** file — roughly a 3× reduction — while preserving answer quality. We
evaluated the trade-off against lighter (Q3, Q2) and heavier (Q5, Q8)
quantizations: lower bit-widths risked degrading clinical accuracy (a safety
concern), while higher ones gave no benefit given our RAM headroom. Q4_K_M is the
standard sweet spot and is where we landed.

**RAG instead of a larger or fine-tuned medical model.** A 1.5B model has weak
factual recall, so we do not rely on it for clinical facts. Instead, an offline
retrieval layer — a local sentence-embedding model (all-MiniLM-L6-v2) with FAISS
vector search — grounds each triage answer in a clinician-approved maternal-health
corpus (WHO danger-sign protocols and vetted guidance). The model's job is to
reason over retrieved guidance and phrase a safe reply, not to recall medicine
from its weights. This keeps RAM low, improves accuracy, and preserves NURA's
core safety promise: answers map to clinician-approved content rather than being
invented.

**Alternatives considered.** We evaluated Phi-4-mini (MIT, faster) as a fallback
and Llama-3.2-3B (broader but larger); the 1.5B Qwen won on the
licence/RAM/multilingual combination. We chose RAG grounding over fine-tuning
because fine-tuning a small model on medical text risks introducing hallucinated
clinical claims — a safety and accuracy hazard — whereas retrieval keeps answers
tied to vetted guidance.

**Safety by construction.** Format and safety rules that a small model does not
reliably follow are enforced deterministically in code rather than trusted to the
model: the system prompt escalates danger signs to referral and never confirms a
pregnancy loss (only a clinician can), and post-processing strips disallowed
formatting. This mirrors the production NURA backend, which enforces its
red-flag triage in code rather than relying on the LLM.

---

## 3. Constraints

- **Hardware:** the ADTC 8 GB RAM / 4 vCPU / integrated-GPU laptop profile, with a
  strict 7 GB RAM ceiling (exceeding it is an automatic disqualification). We
  designed for CPU-only inference (`-ngl 0`) to match this target.
- **Connectivity:** zero external network access during inference. The model, the
  embedding model, the FAISS index, and the datastore are all local; the only
  online step is one-time setup (fetching the base model and embedder).
- **Runtime:** llama.cpp with GGUF weights only, as required by the evaluation
  framework.
- **Data:** the corpus is limited to clinician-approved maternal-health guidance,
  by design — the safety guarantee depends on grounding only in vetted content.

---

## 4. Benchmarks

Measured with the official ADTC profiler (`adtc-profiler run --mode participant`)
on the development machine.

**Development machine:** Intel Core i7-9750H @ 2.60 GHz, 16 GB RAM, integrated
GPU only (no dedicated GPU), macOS. Note this is more capable than the 8 GB target
laptop, so numbers on the evaluation hardware will be more conservative; our large
RAM headroom means we remain well within limits regardless.

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

**Interpretation.** Throughput is well above the reference threshold, and peak RAM
uses roughly a quarter of the budget, leaving substantial headroom on weaker
hardware. No thermal throttling was observed. The local accuracy self-check
(arc_easy) indicates solid general reasoning; domain accuracy in evaluation comes
from the model's grounded answers to maternal-health prompts, where RAG retrieval
over the clinician-approved corpus is the primary driver of correctness and safety.

---

## 5. Relationship to the full NURA platform

NURA Edge is the offline brain of NURA, a production maternal-health platform
(patient management, SMS and app channels, clinician takeover, hospital alerts,
scheduled care). The platform normally uses a hosted LLM; NURA Edge is that same
brain, quantized to run locally, removing per-token fees and connectivity
requirements. The clinical grounding corpus and safety logic are drawn from the
real system, so this offline build is a faithful distillation of a working
platform rather than a standalone prototype.
