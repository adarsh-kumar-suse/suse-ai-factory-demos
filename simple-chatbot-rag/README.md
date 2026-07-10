# Simple Chatbot with RAG — Install & Configuration Guide

> **Simple Chatbot with RAG** using `ollama`, `open-webui`, and `open-webui-mcpo`.
> A CPU‑inference blueprint suitable for PoCs and quick explorations. Retrieval‑augmented
> generation uses the **ChromaDB** vector store embedded in open‑webui.

This blueprint is a showcase of how quickly a complete, private RAG chatbot — model serving,
chat UI, document retrieval, and MCP tool calling — can be stood up with **SUSE AI Factory**.
You deploy the blueprint, then adjust a handful of values for your environment.

---

## What you can demo

| Capability | What the audience sees |
|---|---|
| **Private LLM chat** | A ChatGPT‑style UI answering from a locally‑served model (`gemma:2b`), no external API. |
| **RAG (grounded answers)** | Upload your own documents; the bot answers questions from them **with citations**, and visibly *doesn't* know the same facts without the docs. |
| **MCPO tool calling** | The bot uses a real **MCP tool** (a persistent knowledge‑graph "memory") through the open‑webui‑mcpo OpenAPI proxy — "remember this" / "what do you know about me?". |

---

## Architecture

The blueprint deploys three apps into one namespace (`simple-chatbot-with-rag-system`):

| App | Role |
|---|---|
| `ollama` | Serves the LLM (`gemma:2b`) at `ollama:11434`. |
| `open-webui` | Chat UI + built‑in RAG (ChromaDB + embeddings). Exposed via Ingress. |
| `open-webui-mcpo` | MCP‑to‑OpenAPI proxy exposing MCP tools (the `memory` server) at `:8000`. |

---

## Prerequisites

- **SUSE AI Factory** installed and configured.
- **A default StorageClass** — open‑webui requests a PersistentVolume. Verify:
  ```bash
  kubectl get storageclass          # exactly one should say "(default)"
  ```
- **An ingress controller** — Traefik (RKE2 default) or NGINX. Note its IngressClass name; you set it in the values (`traefik` in the examples).
- **cert-manager** *(required for the HTTPS options; skippable only for the HTTP‑only quick start)*:
  ```bash
  kubectl get pods -n cert-manager  # controller / cainjector / webhook Running
  ```
- **(GPU option only) NVIDIA GPU Operator / device plugin** so the node advertises `nvidia.com/gpu`:
  ```bash
  kubectl get nodes -o jsonpath='{.items[*].status.capacity.nvidia\.com/gpu}{"\n"}'
  ```
- **Resources**: ~6 vCPU / 24 GiB is enough for CPU inference with a small model. Keep to small models
  (`gemma:2b`, `qwen2.5:3b`) — larger CPU models are slow or OOM. With a GPU you can run larger models.

---

## Certificate options at a glance

The chat UI is served over an Ingress. How you terminate TLS is the main decision:

| Option | TLS | Needs | Browser result | Best for |
|---|---|---|---|---|
| **1 - HTTP only** | none | nothing | "Not secure" label, no warning page | fastest demo / air‑gapped PoC |
| **2 - Self‑signed (default)** | cert‑manager self‑signed CA | cert-manager | one‑time "untrusted cert" warning | any cluster with cert‑manager |
| **3 - Let's Encrypt** | real trusted cert | cert-manager + a public DNS zone (e.g. Cloudflare DNS‑01) | green padlock | a real hostname you own |

> The `open-webui` value `global.tls.source` selects the mechanism: `suse-private-ai` (built‑in
> self‑signed via cert‑manager — **the blueprint default**), `secret` (bring‑your‑own / cert‑manager
> issuer / none), or `letsEncrypt`. Each option below tells you what to set.

---

## Editing the blueprint in SUSE AI Factory

Deploy the **Simple Chatbot with RAG** blueprint from the AI Factory catalog. To customize it, open the
blueprint and edit each app's values (`ollama`, `open-webui`, `open-webui-mcpo`) in the values editor,
then re‑deploy/upgrade.

![Edit the blueprint](../assets/Simple_Chatbot_with_RAG-edit-blueprint.gif)

**Everything below is expressed as changes to the shipped default values** — you only touch the keys shown.
Fields not mentioned keep their defaults (the RAG settings `VECTOR_DB: chroma`, the MiniLM embedding model,
`DEFAULT_MODELS: gemma:2b`, signup, persistence, etc. are already set for you).

---

## Two open-webui settings you'll add (in every option)

Two environment variables go in the **`open-webui`** app's values, inside its **`extraEnvVars`** list — the
same list that already contains `DEFAULT_MODELS`, `VECTOR_DB`, etc. You **append them as new list entries**
(don't replace the existing ones). **Each option below already includes the exact `extraEnvVars` block to
paste**, with the correct mcpo URL filled in. This section just explains what the two do:

- **`INSTALL_NLTK_DATASETS: "false"`** — identical in every option. The default `"true"` re-downloads NLTK data
  on every restart and can hang on GitHub rate-limits. Turn it off.
- **`TOOL_SERVER_CONNECTIONS`** — auto-registers the mcpo "memory" tool at boot, so no one adds it in the UI.
  Its value embeds the mcpo URL, which differs per option (a NodePort for HTTP, an Ingress host for HTTPS).
  Always keep the **`/memory`** suffix — the tool won't load without it.

---

## Optional: Enable the GPU

By default the blueprint runs **CPU inference**. If your cluster has NVIDIA GPUs (via the GPU Operator),
enable GPU in the **`ollama`** values:

```yaml
ollama:
  gpu:
    enabled: true                  # was false
    type: nvidia
    number: 1                      # GPUs to request
    nvidiaResource: nvidia.com/gpu
```

This makes the ollama pod request `nvidia.com/gpu: 1`. Notes:
- Requires the NVIDIA GPU Operator/device plugin (see Prerequisites) so `nvidia.com/gpu` is schedulable.
- If your cluster requires it, also set a runtime class on ollama: `runtimeClassName: nvidia`.
- With a GPU you can move up from `gemma:2b` to a larger model — set `DEFAULT_MODELS` and the ollama
  `models.pull`/`models.run` lists accordingly (e.g. `llama3.1:8b`).

---

## Option 1 — Quickest & easiest (ignore certs, HTTP)

No cert‑manager, no DNS certs. Serve the UI over plain HTTP and reach mcpo via a NodePort.

> ⚠️ **Scheme must match.** Serving open‑webui over **HTTP** means the mcpo tool URL must also be **HTTP**
> (a NodePort). An HTTPS page calling an HTTP tool URL is blocked by the browser as mixed content.

**`open-webui` value changes**
```yaml
global:
  tls:
    source: secret                 # was suse-private-ai — stops the chart creating an issuer
ingress:
  class: traefik                   # was "" — set your IngressClass
  host: suse-ollama-webui.example.local
  tls: false                       # was true — plain HTTP, no cert
  annotations: {}                  # was { nginx.ingress.kubernetes.io/ssl-redirect: "true" } — drop the redirect
```
**In the same `open-webui` values, add these to the `extraEnvVars` list** (fill in your node IP):
```yaml
extraEnvVars:                        # append to the existing list — keep the entries already there
  - name: INSTALL_NLTK_DATASETS
    value: "false"                   # was "true"
  - name: TOOL_SERVER_CONNECTIONS    # mcpo over the NodePort (HTTP, to match the HTTP page)
    value: '[{"url":"http://<NODE_IP>:31800/memory","path":"openapi.json","type":"openapi","auth_type":"bearer","headers":null,"key":"","config":{"enable":true,"function_name_filter_list":"","access_control":null},"spec_type":"url","spec":"","info":{"id":"","name":"memory","description":""}}]'
```

**`open-webui-mcpo` value changes** (expose mcpo on a fixed NodePort)
```yaml
service:
  type: NodePort                   # was ClusterIP
  nodePort: 31800                  # pin it so the URL stays stable
```

**Reach it:** point a hosts/DNS entry at the ingress node, then browse over `http://`:
```
<NODE_IP>  suse-ollama-webui.example.local
```
Open **`http://suse-ollama-webui.example.local`**.

---

## Option 2 — Cluster with cert-manager (blueprint default)

This is closest to the shipped defaults: keep `global.tls.source: suse-private-ai` and cert‑manager mints a
self‑signed cert automatically. HTTPS works with a one‑time "untrusted CA" browser warning. No public DNS needed.

**`open-webui` value changes** (only the host + IngressClass)
```yaml
ingress:
  class: traefik                   # was "" — set your IngressClass
  host: suse-ollama-webui.example.local
  # global.tls.source stays "suse-private-ai" (default) — no change
```
**In the same `open-webui` values, add these to the `extraEnvVars` list**:
```yaml
extraEnvVars:                        # append to the existing list — keep the entries already there
  - name: INSTALL_NLTK_DATASETS
    value: "false"                   # was "true"
  - name: TOOL_SERVER_CONNECTIONS    # mcpo over its HTTPS Ingress (matches the HTTPS page)
    value: '[{"url":"https://suse-ollama-mcpo.example.local/memory","path":"openapi.json","type":"openapi","auth_type":"bearer","headers":null,"key":"","config":{"enable":true,"function_name_filter_list":"","access_control":null},"spec_type":"url","spec":"","info":{"id":"","name":"memory","description":""}}]'
```

**`open-webui-mcpo` value changes** (give mcpo its own HTTPS Ingress on the same self‑signed CA)
```yaml
ingress:
  enabled: true                    # was false
  className: traefik
  hosts: [suse-ollama-mcpo.example.local]
  annotations:
    cert-manager.io/issuer: suse-private-ai
  tls:
    - hosts: [suse-ollama-mcpo.example.local]
      secretName: suse-ollama-mcpo-tls
```

**Reach it:** two hosts entries pointing at the ingress node, then browse `https://` (accept the cert once):
```
<NODE_IP>  suse-ollama-webui.example.local
<NODE_IP>  suse-ollama-mcpo.example.local
```

---

## Option 3 — System like mine (Let's Encrypt already set up)

You have cert‑manager **and** a working ACME `ClusterIssuer` (e.g. `letsencrypt-prod` using **Cloudflare
DNS‑01**) for a domain you control. This yields a real, trusted certificate.

**`open-webui` value changes**
```yaml
global:
  tls:
    source: secret                 # was suse-private-ai — see note below
ingress:
  class: traefik                   # was ""
  host: suse-ollama-webui.dna-42.com          # your FQDN
  tls: true                        # (default)
  existingSecret: suse-ollama-webui-tls        # was "" — cert-manager fills this
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # was the nginx ssl-redirect annotation
```
**In the same `open-webui` values, add these to the `extraEnvVars` list**:
```yaml
extraEnvVars:                        # append to the existing list — keep the entries already there
  - name: INSTALL_NLTK_DATASETS
    value: "false"                   # was "true"
  - name: TOOL_SERVER_CONNECTIONS    # mcpo over its HTTPS Ingress on your domain
    value: '[{"url":"https://suse-ollama-mcpo.dna-42.com/memory","path":"openapi.json","type":"openapi","auth_type":"bearer","headers":null,"key":"","config":{"enable":true,"function_name_filter_list":"","access_control":null},"spec_type":"url","spec":"","info":{"id":"","name":"memory","description":""}}]'
```

> **Why `source: secret`?** With any other value the chart auto‑injects a second `cert-manager.io/issuer`
> annotation alongside your `cluster-issuer` — cert‑manager refuses both and issues nothing. `secret` leaves
> only your annotation, so ingress‑shim issues the real cert into `suse-ollama-webui-tls`.

**`open-webui-mcpo` value changes** (mcpo Ingress with a trusted cert too)
```yaml
ingress:
  enabled: true                    # was false
  className: traefik
  hosts: [suse-ollama-mcpo.dna-42.com]
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  tls:
    - hosts: [suse-ollama-mcpo.dna-42.com]
      secretName: suse-ollama-mcpo-tls
```

**DNS:** create A records for both hosts pointing at the ingress node IP. If the node IP is private
(e.g. `10.9.0.113`), use **DNS‑only** (grey cloud in Cloudflare) — DNS‑01 validation writes a TXT record
via the API, so the cert issues even though the host resolves to a private address.

**Watch issuance**
```bash
kubectl -n simple-chatbot-with-rag-system get certificate,order,challenge
# READY=True on the certificate → trusted cert served
```

> **DNS‑01 gotcha:** cert‑manager verifies the challenge TXT through its configured recursive resolver.
> If that's `8.8.8.8` and Google is slow/negative‑cached, the challenge can hang for minutes even though the
> record is live on Cloudflare. Since your zone is Cloudflare, point cert‑manager's self‑check at `1.1.1.1`:
> ```
> --dns01-recursive-nameservers-only=true
> --dns01-recursive-nameservers=1.1.1.1:53
> ```

---

## How the MCPO auto‑wiring works (and why the URL matters)

open‑webui runs OpenAPI tool servers **from the browser**, so:

1. **mcpo must be reachable from the browser** — via a NodePort (Option 1) or its own Ingress (Options 2/3).
   A cluster‑internal `open-webui-mcpo:8000` URL will *not* work.
2. `TOOL_SERVER_CONNECTIONS` **pre‑registers** the tool at boot, so users never add it in the UI. The URL
   must include the **`/memory`** subpath, and the object must include `type: openapi` and `spec_type: url`
   — the exact shape shown in the common changes.
3. Keep **`INSTALL_NLTK_DATASETS: "false"`** — the NLTK download otherwise re‑runs on every restart and can
   hang on GitHub rate‑limits.

---

## Verify the deployment

```bash
NS=simple-chatbot-with-rag-system
kubectl -n $NS get pods
# ollama, open-webui-0, open-webui-mcpo all Running/Ready

# model present?
kubectl -n $NS exec deploy/ollama -- ollama list      # -> gemma:2b

# UI reachable?
curl -kI https://suse-ollama-webui.<your-domain>/     # -> HTTP 200
```

---

# Demo Instructions

### 0. First‑run setup
1. Open the chat URL. **The first account you create becomes the admin.**
2. Confirm **`gemma:2b`** is selected in the model dropdown (it's the default).
3. Send "hi" to warm the model up before the audience arrives — the first CPU response is the slowest.

### 1. RAG demo — grounded answers with citations
1. **Workspace → Knowledge → Create** a knowledge base named **"Project Orion"**.
2. Upload the sample document below (save it as `project-orion.md`).
3. Open a new chat, type **`#`** and select **Project Orion** to attach it.
4. Ask the questions in the table — answers come back **with citations** (click them to show the retrieved text).
5. **The reveal:** ask the same question in a chat **without** the `#` knowledge base — the model doesn't know.
   Same model, one attaches your data, one doesn't. That contrast *is* the demo.

| Ask | Expected grounded answer |
|---|---|
| "When is the Project Orion production release date?" | November 14, 2026 |
| "Who are the lead architects?" | Sarah Jenkins and David Vance |
| "What must a repository pass before it can merge?" | Security Checkpoint Echo |
| "Where is the datacenter and why there?" | Reykjavík, Iceland — geothermal renewable power |
| "What's the failover site's codename?" | Project Meridian |

### 2. MCPO demo — the memory tool
1. In a chat, click the **🔧 tools icon** in the message bar and toggle **memory** on.
2. Say: *"Use the memory tool to remember that my favorite database is PostgreSQL and I work on the Orion team."*
3. Then: *"What do you know about me?"* — the bot calls the memory tool and recalls it from the knowledge graph.

> **Model note:** `gemma:2b` is light on tool‑calling and may not always invoke the tool. For a crisp
> tool demo, pull a tool‑capable small model that still runs on CPU:
> ```bash
> kubectl -n simple-chatbot-with-rag-system exec deploy/ollama -- ollama pull qwen2.5:3b
> ```
> then set `DEFAULT_MODELS=qwen2.5:3b` (and add it to the ollama `models.pull`/`models.run` lists).

---

## Sample RAG data — `project-orion.md`

> Fictional internal document. It contains crisp, unique facts that a base model can't know, so retrieval
> is obvious and verifiable. Keep it short for fast CPU embedding.

```markdown
# Project Orion — Datacenter Operations Handbook (2026 Edition)

Project Orion is Aurora Systems' next-generation datacenter initiative.

## Key facts
- Production release date is hard-locked for **November 14, 2026**.
- Lead Architects are **Sarah Jenkins** and **David Vance**.
- All code repositories must pass **Security Checkpoint Echo** before merging.
- The primary datacenter is in **Reykjavík, Iceland**, chosen for renewable geothermal power.
- Maximum power draw is capped at **12 kW per rack**.
- Nightly backup snapshots run at **02:00 UTC** and are retained for **30 days**.
- The on-call escalation hotline is **extension 4471**.
- The failover site's codename is **Project Meridian**.

## Change policy
Any change to production requires sign-off from a Lead Architect and a passing
Security Checkpoint Echo scan. Emergency changes are logged in the Orion Runbook
and reviewed the following business day.
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| UI returns **500 / redirects to `/error`** | `INSTALL_NLTK_DATASETS=true` hit a GitHub 429 and hangs on a download prompt | Set `INSTALL_NLTK_DATASETS: "false"` and restart. |
| Cert stuck **`Certificate READY=False`**, challenge pending for minutes | DNS‑01 self‑check resolver (`8.8.8.8`) not seeing the record | Point cert‑manager at `1.1.1.1` (see Option 3 note). |
| **"Failed to connect … OpenAPI tool server"** in the UI | Browser can't reach a cluster‑internal mcpo URL | Expose mcpo (NodePort or Ingress) and use that URL, matching the page's scheme. |
| Tool server added but **no tools appear** | URL missing the `/memory` subpath, so open‑webui loads the empty root spec | Use `…/memory` in the tool URL. |
| Tool calls silently fail on an **HTTPS** page | Mixed content — HTTP tool URL on an HTTPS page | Match schemes: HTTPS page → HTTPS mcpo Ingress (not a plain NodePort). |
| Pods **ImagePullBackOff** from `dp.apps.rancher.io` | No pull secret | Ensure the `application-collection` secret exists in the namespace, or set `global.imagePullSecrets`. |
| GPU model stays on CPU | `nvidia.com/gpu` not schedulable / no runtime class | Confirm the GPU Operator is installed and the node advertises `nvidia.com/gpu`; set `runtimeClassName: nvidia` if required. |
| Browser **cert warning** on Options 1–2 | Expected (HTTP / self‑signed) | Use Option 3 for a trusted cert. |

---

# vLLM Inference Endpoint Services — Access Guide

## Services Overview

| Service | Type | Purpose | Cluster Port | NodePort |
|---|---|---|---|---|
| `vllm-router-service` | NodePort | Request router/load balancer | 80 | 31777 |
| `vllm-llama-3-2-1b-engine-service` | NodePort | LLM inference engine | 80 | 31142 |

---

## Access Methods

### 1. External Access (from outside cluster)

Use the node IP + NodePort:

```
http://<NODE_IP>:31777
```

### 2. Internal Access (from within cluster)

Use service DNS name + port:

```
http://vllm-router-service.<namespace>.svc.cluster.local:80
```

### 3. Test Direct Pod Access (debugging)

Use `kubectl port-forward` or exec into a pod to reach the service directly.

---

## Port Mappings

**vllm-router-service:**

| Port | Target | Description | NodePort |
|---|---|---|---|
| 80 | 8000 | OpenAI API endpoint | 31777 |
| 9000 | 9000 | LMCache | 32581 |

**vllm-llama-3-2-1b-engine-service:**

| Port | Target | Description | NodePort |
|---|---|---|---|
| 80 | 8000 | vLLM API endpoint | 31142 |
| 55555 | 55555 | ZMQ messaging | 32539 |
| 9999 | 9999 | UCX transport | 30921 |

---

## Example Usage

### Chat Completion via Router (recommended)

```bash
curl http://<NODE_IP>:31777/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [{"role": "user", "content": "Hello"}],
    "temperature": 0.7
  }'
```

### Direct Inference on Engine

```bash
curl http://<NODE_IP>:31142/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "prompt": "Hello world",
    "max_tokens": 100
  }'
```

### Check Models Available

```bash
curl http://<NODE_IP>:31777/v1/models
```

> **Note:** Replace `<NODE_IP>` with the actual IP address of any node in your cluster.

---

# LiteLLM + vLLM Blueprint — Step-by-Step Deployment

## Deployment Order

1. Create namespace
2. Create required secrets
3. Apply SUSE AI Factory Blueprint
4. Wait for LiteLLM + vLLM pods
5. Open LiteLLM UI / API
6. Test OpenAI-compatible API

---

## Step 1. Create a Namespace

Choose one namespace for this demo. Example:

```bash
export NS=llm-gateway-demo

kubectl create namespace $NS
```

> **Important:** Everything must go into this same namespace.

---

## Step 2. Create LiteLLM Credentials Secret

Run this:

```bash
kubectl create secret generic litellm-credentials \
  --namespace $NS \
  --from-literal=PROXY_MASTER_KEY="sk-$(openssl rand -hex 24)" \
  --from-literal=LITELLM_SALT_KEY="sk-$(openssl rand -hex 24)" \
  --from-literal=UI_USERNAME="admin" \
  --from-literal=UI_PASSWORD="ChangeThisStrongPassword123!"
```

This creates four values:

| Secret value | Meaning |
|---|---|
| `PROXY_MASTER_KEY` | Admin API key for LiteLLM |
| `LITELLM_SALT_KEY` | Encryption key used by LiteLLM |
| `UI_USERNAME` | LiteLLM Admin UI username |
| `UI_PASSWORD` | LiteLLM Admin UI password |

> ⚠️ **Important:** Do not change `LITELLM_SALT_KEY` later, otherwise stored LiteLLM provider keys may break.

---

## Step 3. Create Hugging Face Token Secret

If your model is private or gated (for example `meta-llama/*`), you need a Hugging Face token:

```bash
kubectl create secret generic huggingface-token \
  --namespace $NS \
  --from-literal=HF_TOKEN="hf_xxxxxxxxxxxxxxxxxxxxxxxx"
```

> If you are using only public models, this may not be required.
> For a demo, if your Blueprint expects this secret, create it with a valid token to avoid deployment failure.

---

## Step 4. Verify Secrets

```bash
kubectl get secret -n $NS
```

You should see:

- `litellm-credentials`
- `huggingface-token`

Or check directly:

```bash
kubectl get secret litellm-credentials -n $NS
kubectl get secret huggingface-token -n $NS
```

---

## Step 5. Apply the Blueprint

If you have the Blueprint YAML file (for example `litellm-vllm-blueprint.yaml`):

```bash
kubectl apply -n $NS -f litellm-vllm-blueprint.yaml
```

Or from **SUSE AI Factory / Rancher UI**:

1. Rancher UI → **AI Factory** → **Blueprints**
2. Select the **LiteLLM + vLLM Blueprint**
3. Choose namespace: `llm-gateway-demo`
4. Deploy

> The key point is: deploy the Blueprint into the same namespace where you created the secrets.

---

## Step 6. Check if Pods Are Running

```bash
kubectl get pods -n $NS
```

You should expect pods related to:

- `litellm`
- `vllm`
- `database` / `postgresql`

Watch until everything becomes Running:

```bash
kubectl get pods -n $NS -w
```

If something fails:

```bash
kubectl get events -n $NS --sort-by=.lastTimestamp
kubectl describe pod <pod-name> -n $NS
```
