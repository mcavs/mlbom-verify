# Walkthrough: Detecting Silent Model Degradation in CI/CD

This walkthrough demonstrates how `mlbom-verify` solves the problem of silent machine learning model degradation within an automated software supply chain.

---

## 1. The Real-World Problem

Consider an open-source automated decision system (e.g., credit risk assessment or fraud scoring) packaged as a containerized microservice:

```
[Incoming Inference Stream] ---> [Model Binary (credit_model.onnx)] ---> [Decision: Approve / Deny]
```

1. **Standard SBOM tools (Syft, Trivy, Grype):**
   - Inspect the repository and dependencies (`python`, `numpy`, `onnxruntime`).
   - All packages report **0 CVEs**, and licenses are permissive (Apache 2.0).
   - **Scanner Verdict:** `PASSED (Secure)`.

2. **The Hidden Failure:**
   - Due to macro-economic shifts, incoming applicant income distributions drift significantly (covariate shift), and the underlying probability relationship shifts (concept drift).
   - Model accuracy drops by 24%, silently producing high rates of false rejections.
   - Standard security scanners are blind to this operational decay because the binary hashes and code dependencies haven't changed.

---

## 2. The Solution: `mlbom-verify` Quality Gate

`mlbom-verify` bridges statistical monitoring and supply chain security by evaluating runtime distribution shifts and packaging them into a standardized, cryptographically signed CycloneDX ML-BOM.

### Step 1: Pre-Deployment Audit in CI/CD

When a new model version is submitted via Pull Request or deployed to staging, the CI/CD pipeline triggers an automated audit:

```bash
mlbom-verify audit \
  --model models/credit_model.onnx \
  --baseline metadata/baseline_summary.json \
  --stream data/staging_traffic_sample.parquet \
  --drift-threshold 0.10 \
  --output bom.cdx.json
```

**Terminal Output:**
```text
[INFO] Loading baseline digest: sha256:7e8a93bf...
[INFO] Computing quantile sketches across 18 features (compress-then-explain grid: 50 bins)...
[INFO] Calculating statistical distances:
       - feature 'debt_to_income': Wasserstein = 0.041 (PASS)
       - feature 'annual_income': Wasserstein = 0.142 (DRIFT DETECTED > 0.10)
[INFO] Running Profile Drift Detection (PDD):
       - Partial Dependence Profile L1 shift = 0.128 (ALERT)
[ERROR] Deployment Gate Failed: Concept drift exceeds policy threshold (0.10).
[INFO] Serialising evaluation to CycloneDX 1.6 ML-BOM: bom.cdx.json
```

---

## 3. Cryptographic Verification & Supply Chain Ingestion

The resulting `bom.cdx.json` is signed using Sigstore/cosign:

```bash
# Keyless signing via GitHub Actions OIDC identity
cosign sign-blob \
  --bundle bom.bundle \
  --yes \
  bom.cdx.json
```

Downstream deployment systems (Kubernetes admission controllers, GitOps pipelines) inspect the ML-BOM before promoting the image to production:

```bash
mlbom-verify verify \
  --bom bom.cdx.json \
  --signature bom.bundle \
  --max-allowed-drift 0.10
```

Because drift was detected, the gate **blocks promotion**, preventing silent business failure and logging an auditable compliance record for EU AI Act Article 13 review.
