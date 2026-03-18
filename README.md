# Palo Alto Networks Prisma AIRS - Model Security in CI/CD (Jenkins)

<p align="center">
  <strong>Shift-left AI model security. Scan models before they reach production.</strong>
</p>

<p align="center">
  <a href="#overview">Overview</a> &bull;
  <a href="#how-it-works">How It Works</a> &bull;
  <a href="#quick-start">Quick Start</a> &bull;
  <a href="#testing-the-deployed-model">Testing</a> &bull;
  <a href="#configuration-reference">Configuration</a>
</p>

---

## Overview

This repository demonstrates how to integrate **Palo Alto Networks Prisma AIRS Model Security** scanning into a CI/CD pipeline using **Jenkins**. When an AI model changes in the configuration in this example, the pipeline automatically scans the model against AIRS security policies and **only deploys to Google Cloud Vertex AI if the model passes the security assessment**. Vertex AI is used here as an example deployment target — the security scanning pattern can be adapted to any infrastructure (e.g., AWS SageMaker, Azure ML, on-prem).

> **Note:** This example is scoped to **HuggingFace models deployed as self-hosted endpoints on Vertex AI**.

### What Gets Scanned

Prisma AIRS Model Security evaluates AI models for:

- **Model provenance and supply chain risks** - Verifying the model source and integrity
- **Known vulnerabilities** - Checking against known CVEs and security advisories
- **Malicious payloads** - Detecting embedded malicious code or backdoors
- **Compliance violations** - Ensuring models meet organizational security policies
- **Training data risks** - Identifying potential data poisoning indicators

### Architecture

| Component | Technology |
|-----------|------------|
| **AI Model** | Google Gemma 3 1B (via Vertex AI Model Garden + HuggingFace) |
| **Model Hosting** | Google Cloud Vertex AI Endpoint |
| **Security Gate** | Palo Alto Networks Prisma AIRS Model Security ([Python SDK](https://docs.paloaltonetworks.com/ai-runtime-security/ai-model-security/model-security-to-secure-your-ai-models/get-started-with-ai-model-security/install-ai-model-security)) |
| **CI/CD** | Jenkins Pipeline |
| **Trigger** | Changes to `config/model-config.yaml` |

---

## How It Works

```
  Developer changes config/model-config.yaml
  (e.g., updates the model ID or version)
                    |
                    v
        Jenkins Pipeline Triggered
        (SCM polling or manual build)
                    |
                    v
       +----------------------------+
       | Prisma AIRS Model Security |
       |       Scan                 |
       +----------------------------+
                    |
              +-----+-----+
              |           |
           ALLOWED     BLOCKED
              |           |
              v           v
      +-------------+   Pipeline fails.
      | Deploy to   |   Model is NOT
      | Vertex AI   |   deployed.
      | Endpoint    |   Developer is
      +-------------+   notified.
              |
              v
      +-------------+
      | Validate    |
      | Endpoint    |
      +-------------+
```

**On Non-Main Branches:** The security scan runs and reports pass/fail status, but the model is not deployed. This allows developers to verify model compliance before merging.

**On Main Branch:** If the security scan passes, the model is automatically deployed to a Vertex AI endpoint and validated.

---

## Quick Start

### Prerequisites

- A Jenkins instance (2.400+) with the [Pipeline plugin](https://plugins.jenkins.io/workflow-aggregator/) installed
- Python 3.11+ available on the Jenkins agent
- `gcloud` CLI installed on the Jenkins agent
- `jq` and `curl` installed on the Jenkins agent
- A Google Cloud project with the [Vertex AI API](https://console.cloud.google.com/apis/library/aiplatform.googleapis.com) enabled
- A GCP service account with Vertex AI permissions (JSON key)
- A Palo Alto Networks Prisma AIRS subscription with API credentials

### 1. Fork This Repository

Fork this repository to your GitHub (or other SCM) account.

### 2. Configure Jenkins Credentials

Go to **Jenkins > Manage Jenkins > Manage Credentials** and add the following credentials in the appropriate scope:

| Credential ID | Type | Description | How to Obtain |
|---------------|------|-------------|---------------|
| `gcp-project-id` | Secret text | Your Google Cloud project ID | Find it in the [GCP Console dashboard](https://console.cloud.google.com/home/dashboard) or run `gcloud config get-value project` |
| `gcp-region` | Secret text | GCP region for deployment (e.g., `us-central1`) | Choose from [Vertex AI available regions](https://cloud.google.com/vertex-ai/docs/general/locations#available-regions). Use `us-central1` for the widest GPU availability. |
| `gcp-sa-key` | Secret file | GCP service account JSON key (with Vertex AI Admin role) | Create a service account and download the key via [GCP IAM Console](https://console.cloud.google.com/iam-admin/serviceaccounts). Grant it the **Vertex AI Administrator** role. See [Creating service account keys](https://cloud.google.com/iam/docs/keys-create-delete). |
| `model-security-client-id` | Secret text | Prisma AIRS OAuth client ID (service account) | Generate in the Prisma AIRS console under **Settings > Access Control > Service Accounts**. See [Prisma AIRS API Authentication](https://pan.dev/sase/docs/getstarted/). |
| `model-security-client-secret` | Secret text | Prisma AIRS OAuth client secret | Generated alongside the client ID when creating a service account in the Prisma AIRS console. |
| `model-security-profile-id` | Secret text | Prisma AIRS security profile UUID | Found in the Prisma AIRS console under your security profile settings. |
| `model-security-api-endpoint` | Secret text | Prisma AIRS API endpoint URL | Use `https://api.sase.paloaltonetworks.com/aims` for US deployments. See [AIRS API reference](https://pan.dev/airs/) for regional endpoints. |
| `tsg-id` | Secret text | Prisma AIRS Tenant Service Group ID | Found in the Prisma AIRS console under **Settings > Tenant Service Groups**, or embedded in the service account email (the numeric portion). See [TSG ID documentation](https://pan.dev/sase/docs/tenant-service-groups/). |
| `hf-token` | Secret text | HuggingFace access token (for gated models) | Create a token at [HuggingFace Settings > Access Tokens](https://huggingface.co/settings/tokens). Required for gated models like Gemma — you must also accept the model's license on its HuggingFace page. |

A `.env.example` file is provided as a reference for the required values. Copy it to `.env` and fill in your values for local development — the scripts load it automatically via `python-dotenv`. The `.env` file is gitignored and will not be committed.

### 3. Create the Jenkins Pipeline Job

1. In Jenkins, click **New Item**
2. Enter a name (e.g., `model-security-scan`) and select **Pipeline**
3. Under **Pipeline**, select **Pipeline script from SCM**
4. Set **SCM** to **Git** and enter your repository URL
5. Set **Branch Specifier** to `*/main`
6. Set **Script Path** to `Jenkinsfile`
7. Click **Save**

The pipeline is configured to poll SCM every 5 minutes, or you can trigger it manually with the **Build with Parameters** option.

### 4. Change the Model

Edit `config/model-config.yaml` to specify your desired model:

```yaml
model:
  huggingface_id: "google/gemma-3-1b-it"
  display_name: "gemma-3-1b-it"
  version: "1.0"
```

### 5. Push and Watch

Commit and push your changes. The pipeline will:

1. Detect the model configuration change
2. Run a Prisma AIRS security scan on the specified model
3. Deploy the model to Vertex AI (if the scan passes and on the `main` branch)
4. Validate the deployed endpoint

You can also use **Build with Parameters** and check **FORCE_RUN** to trigger the pipeline without a config change, or check **SKIP_DEPLOY** to run just the security scan.

---

## Testing the Deployed Model

Once the model is deployed, you can test it locally using the provided test script.

### Using the Test Script

```bash
# Set your GCP credentials
export GCP_PROJECT_ID="your-project-id"
export GCP_REGION="us-central1"

# Authenticate with GCP
gcloud auth login
gcloud config set project $GCP_PROJECT_ID

# Run the test script
python scripts/test_model.py \
  --config config/model-config.yaml \
  --prompt "What are the benefits of AI model security scanning?"
```

### Using curl with the REST API

The deployed model uses a dedicated Vertex AI endpoint with vLLM serving. Use `rawPredict` with the dedicated endpoint DNS:

```bash
# Get your access token
ACCESS_TOKEN=$(gcloud auth print-access-token)

# Find the endpoint ID and dedicated DNS
ENDPOINT_ID=$(gcloud ai endpoints list \
  --region=us-central1 \
  --filter="displayName~gemma-3-1b-it-secure" \
  --format="value(name)" | head -1 | awk -F/ '{print $NF}')

DEDICATED_DNS=$(gcloud ai endpoints describe $ENDPOINT_ID \
  --region=us-central1 \
  --format="value(dedicatedEndpointDns)")

# Send a request via rawPredict
curl -X POST \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  "https://${DEDICATED_DNS}/v1/projects/${GCP_PROJECT_ID}/locations/us-central1/endpoints/${ENDPOINT_ID}:rawPredict" \
  -d '{
    "prompt": "What is AI model security?",
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

---

## Configuration Reference

### `config/model-config.yaml`

| Field | Description | Example |
|-------|-------------|---------|
| `model.huggingface_id` | HuggingFace model ID available in Vertex AI Model Garden | `google/gemma-3-1b-it` |
| `model.display_name` | Display name for the Vertex AI endpoint | `gemma-3-1b-it` |
| `model.description` | Human-readable model description | `Google Gemma 3 1B IT` |
| `model.version` | Version identifier for tracking changes | `1.0` |
| `deployment.machine_type` | GCP machine type for the endpoint | `g2-standard-12` |
| `deployment.accelerator_type` | GPU accelerator type | `NVIDIA_L4` |
| `deployment.accelerator_count` | Number of GPUs | `1` |
| `deployment.region` | GCP region | `us-central1` |
| `security.scan_enabled` | Enable/disable the security scan gate | `true` |
| `security.security_profile_id` | Prisma AIRS security profile UUID | UUID string |

### Swapping Models

To change the deployed model, edit the `model` section in `config/model-config.yaml`:

```yaml
# Example: Switch to Gemma 2 2B
model:
  huggingface_id: "google/gemma-2-2b-it"
  display_name: "gemma-2-2b-it"
  version: "2.0"
```

Commit and push - the pipeline will scan the new model before deploying.

---

## Cost Management

The default configuration deploys on a `g2-standard-12` machine with an `NVIDIA_L4` GPU. This incurs ongoing compute costs while the endpoint is active.

### Estimated Costs

| Resource | Approximate Cost |
|----------|-----------------|
| g2-standard-12 + NVIDIA L4 | ~$1.40/hour |

### Cleaning Up

To stop incurring costs, undeploy the model endpoint:

```bash
export GCP_PROJECT_ID="your-project-id"
export GCP_REGION="us-central1"

bash scripts/undeploy_model.sh
```

This script will find the deployed endpoint, undeploy all models, and delete the endpoint.

---

## Pipeline Details

### Jenkinsfile

The pipeline (`Jenkinsfile`) runs four stages:

1. **Detect Changes** - Checks if `config/model-config.yaml` was modified in the latest changeset
2. **Prisma AIRS Model Security Scan** - Installs the [Prisma AIRS Model Security SDK](https://docs.paloaltonetworks.com/ai-runtime-security/ai-model-security/model-security-to-secure-your-ai-models/get-started-with-ai-model-security/install-ai-model-security) (`model-security-client`) from a private PyPI and scans the model. If the scan returns `BLOCKED`, the pipeline stops here.
3. **Deploy Model to Vertex AI** - Deploys the scanned model to Vertex AI via Model Garden (main branch only)
4. **Test Model Endpoint** - Sends a test prompt to verify the endpoint is responding

### Pipeline Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `FORCE_RUN` | `false` | Run the pipeline even if model config has not changed |
| `SKIP_DEPLOY` | `false` | Run security scan only, skip deployment and testing |

### Adapting This Pipeline

The pipeline is split into **AIRS Model Security integration** stages and **customer-specific** stages. See the comments in the `Jenkinsfile` for details.

| Stage | Type | Notes |
|---|---|---|
| **Detect Changes** | Customer-specific | Adapt the changeset detection to your config structure |
| **Prisma AIRS Model Security Scan** | AIRS Model Security | Keep as-is — authenticates with Prisma AIRS, installs the SDK, and scans the model |
| **Deploy Model to Vertex AI** | Customer-specific | Replace with your deployment target (e.g., AWS SageMaker, Azure ML, on-prem) |
| **Test Model Endpoint** | Customer-specific | Replace with your own smoke tests or integration tests |

To add Prisma AIRS scanning to an existing pipeline, copy the **Prisma AIRS Model Security Scan** stage and its supporting files (`scripts/scan_model.py`, `scripts/get_pypi_url.sh`), then configure the required credentials.

---

## Repository Structure

```
.
├── Jenkinsfile                        # Jenkins pipeline definition
├── config/
│   └── model-config.yaml             # Model configuration (trigger file)
├── scripts/
│   ├── deploy_model.sh               # Vertex AI deployment script
│   ├── get_pypi_url.sh               # Authenticates with SCM to get private PyPI URL
│   ├── scan_model.py                 # Prisma AIRS security scan (uses SDK)
│   ├── test_model.py                 # Endpoint validation script
│   └── undeploy_model.sh             # Cleanup / cost control script
├── requirements.txt                   # Python dependencies
├── LICENSE
└── README.md
```

---

## Learn More

- [Prisma AIRS Model Security](https://docs.paloaltonetworks.com/ai-runtime-security)
- [Prisma AIRS API Documentation](https://pan.dev/airs/)
- [Google Vertex AI Model Garden](https://cloud.google.com/vertex-ai/generative-ai/docs/model-garden/use-models)
- [Vertex AI HuggingFace Integration](https://cloud.google.com/vertex-ai/generative-ai/docs/open-models/use-hugging-face-models)
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)

---

<p align="center">
  <sub>Built with Palo Alto Networks Prisma AIRS &bull; Powered by Google Cloud Vertex AI</sub>
</p>
