# Security Copilot — Post-Training Specialization POC

A proof of concept for fine-tuning a local, open-weight LLM into a **Security Copilot**: an assistant that gives SOC analysts remediation guidance grounded in an organization's actual incident-response playbooks, instead of generic textbook advice.

**Status:** Phase 1 — local validation on a MacBook Pro (M5 Pro, 64GB). Training on public seed data to prove the pipeline before any proprietary client data is involved.

---

## Why this exists

A general-purpose model knows security broadly, but has no idea how *your* SOC actually responds — your escalation paths, your isolation procedures, your VIP watchlist handling. Mid-incident, it defaults to generic NIST-style advice.

The fix is **LoRA fine-tuning**: train a small adapter on real incident tickets and resolutions so the model learns *"when we see detection X, here is our real response flow."* It's cheap enough to run per-client, fast enough to iterate on, and doesn't require touching the full base model.

See [`post-training-specialization-guide.md`](./post-training-specialization-guide.md) for the full working guide — foundational concepts, tooling stack, candidate base models, the local training flow, and the planned cloud/multi-user phase.

---

## Repository structure

```
.
├── post-training-specialization-guide.md   # Primary working guide — read this first
├── fine-tuning-test-data-todo.md            # Original data-sourcing to-do list
├── convert_playbook_to_chat.py              # Converts raw playbook JSONL → mlx-lm chat format
├── data/
│   ├── train.jsonl                          # 137 examples, mlx-lm chat format
│   └── valid.jsonl                          # 37 examples, mlx-lm chat format
├── incident_response_playbook_dataset.jsonl # Seed data: 174 structured IR playbooks (public)
├── Incident_response.txt                    # Field reference for incident_event_log.csv
├── incident_event_log.csv                   # ServiceNow-style ticket log (anonymized codes)
├── Attack_Dataset.csv                       # Attack technique catalog w/ MITRE mapping
├── cybercrime_forensic_dataset.csv          # Forensic activity log w/ anomaly labels
├── linux_auth_logs_*.csv (4 files)          # SSH/auth log sets, various label balances
└── Behavioral Analysis...csv                # User session/behavioral telemetry
```

**Note on the large CSVs:** several of the raw log files (`linux_auth_logs_*.csv`, `incident_event_log.csv`) are 38–63MB each. That's under GitHub's hard 100MB-per-file limit but well past the point where Git starts to feel it — worth moving these to [Git LFS](https://git-lfs.com/) or an external bucket rather than committing them as plain blobs, especially since they'll get rewritten in history on every edit.

---

## Quickstart

### 1. Environment setup (macOS, Apple Silicon only — MLX doesn't run on Intel)

```bash
xcode-select --install
python3 -m venv ~/mlx-env && source ~/mlx-env/bin/activate
pip install "mlx-lm[train]"
pip install huggingface_hub
hf auth login
```

### 2. Data is already converted

`data/train.jsonl` and `data/valid.jsonl` are ready to use, generated from `incident_response_playbook_dataset.jsonl` via:

```bash
python3 convert_playbook_to_chat.py incident_response_playbook_dataset.jsonl ./data
```

Re-run this if the source playbook file changes.

### 3. Run a baseline fine-tune

```bash
mlx_lm.lora \
  --model mlx-community/Qwen2.5-7B-Instruct-4bit \
  --train \
  --data ./data \
  --iters 200 \
  --batch-size 2 \
  --num-layers 8 \
  --adapter-path ./adapters-baseline \
  --save-every 50
```

Watch train/validation loss in the terminal output — if validation loss climbs while train loss keeps falling, that's overfitting on the small dataset and worth stopping early.

### 4. Try it

```bash
mlx_lm.generate \
  --model mlx-community/Qwen2.5-7B-Instruct-4bit \
  --adapter-path ./adapters-baseline \
  --prompt "Incident type: Ransomware
Detection source: EDR Alert - Suspicious File Encryption
Target asset: Windows AD Server
Initial vector: Email Attachment (Phishing)
Severity: High
MITRE ATT&CK: Initial Access - Phishing"
```

Full setup detail, hardware sizing guidance, and the Phase 2 cloud path are in the [guide](./post-training-specialization-guide.md), Sections 4–7.

---

## Data

**Training data schema:** `chat` format, one JSON object per line —
```json
{"messages": [
  {"role": "system", "content": "..."},
  {"role": "user", "content": "..."},
  {"role": "assistant", "content": "..."}
]}
```
The `user` turn carries the detection context (incident type, asset, detection source, initial vector, severity, MITRE ATT&CK tags — treated as pre-classified input, mirroring what a SIEM/EDR would hand the model). The `assistant` turn is the phase-by-phase remediation flow (Identification → Containment → Eradication → Recovery → Lessons Learned). Each record also carries `source` and `seed_incident_id` metadata for provenance — `mlx-lm` ignores these extra keys, so they're safe to keep for filtering and auditing later.

**Current dataset:** 174 public, synthetic-but-structured incident playbooks (not real client data). One malformed source record (`IR-2025-0116`, invalid JSON in the source file) was found and repaired during conversion.

**Not yet used for training** (see Repository structure above for why): the auth logs and forensic/behavioral CSVs are better suited to anomaly detection or RAG-style live-signal grounding than instruction-tuning pairs, and `incident_event_log.csv`'s fields are anonymized codes with no free-text remediation narrative.

**This is public seed data only.** No proprietary or client incident data is in this repository. When real (anonymized) incident history is introduced per the Phase 2 plan, treat that as a hard boundary — data sensitivity changes, and generation/training should stay local rather than moving to a hosted API at that point.

---

## Roadmap

- [x] Source public seed dataset (Kaggle incident-response playbooks)
- [x] Build JSON-L conversion pipeline (raw playbooks → mlx-lm chat format)
- [x] Fix data quality issue in source file
- [ ] Run baseline LoRA fine-tune on 174-record seed set, evaluate output quality
- [ ] Build synthetic data generation pipeline (anchor real `playbook_steps`, vary context fields) to scale past 174 examples
- [ ] Decide if RAG should layer on top for live data (e.g., Sentinel KQL results)
- [ ] Phase 2: retrain natively on cloud GPU stack (HF Transformers + PEFT, or Unsloth/vLLM) for multi-user access — see guide Section 7
- [ ] Transition to real, anonymized client incident history once the pipeline is validated

---

## Open items

- **Large file handling:** decide on Git LFS vs. external storage for the raw CSVs before the first real commit (see note above).
- **Data sensitivity boundary:** confirm the local-only handling plan for real client data is in place (see guide Section 7) before any proprietary incident history is added to this repo or its history.

---

*Working repo for a security incident-response copilot POC. See [`post-training-specialization-guide.md`](./post-training-specialization-guide.md) for full technical detail.*
