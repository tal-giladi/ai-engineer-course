# Module 9 quiz — Fine-Tuning II: Preference Tuning & Distillation

Ten scenario questions on improving a small model beyond what SFT can reach and then shipping it: building preference data that teaches one quality without a length confound, reading DPO runs well enough to catch reward hacking, distilling a frontier model's narrow task into a cheap student with routing, and serving adapters with quantization you have actually measured. They test reasoning, not recall. Most give you an audit, a training run, a cost sweep or a serving result and ask what it shows or what to do next.

**Covers**

- [09.1 · Preference Data: Defining What "Better" Means](../lessons/module-09/lesson-01.md) — what SFT cannot teach, RLHF to DPO to RFT, sources of pairs, one dimension per pair, the length confound.
- [09.2 · DPO and ORPO in Practice](../lessons/module-09/lesson-02.md) — the reference model and `beta`, ORPO vs DPO, reading margin, accuracy and KL, reward hacking, capability collapse.
- [09.3 · Distilling a Frontier Model into a Small One](../lessons/module-09/lesson-03.md) — behavioral distillation, the four conditions, licensing, routing and the escalation threshold, reasoning distillation.
- [09.4 · Shipping Adapters: Merging, Quantizing, Serving](../lessons/module-09/lesson-04.md) — merge or mount, measuring quantization cost, vLLM multi-LoRA, version records, base-model deprecation.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on build is in [the Module 9 practical](../projects/module-09-practical.md).
