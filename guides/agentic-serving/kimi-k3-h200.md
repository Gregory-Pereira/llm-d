# Agentic Code Generation — Kimi-K3 on H200

This is one of the accelerator-specific deployments of the agentic code-generation workload; see the
[agentic-serving README](README.md#deployments) for the workload framing and the alternatives:
[NVIDIA-Nemotron-3-Ultra-550B on H200](nemotron-3-ultra-550b-h200.md),
[GLM-5.2-FP8 on H200](glm-5-2-h200.md),
[Qwen3-Coder-480B on TPU v7](qwen3-coder-480b-tpu.md).

## Overview

This guide deploys [moonshotai/Kimi-K3](https://huggingface.co/moonshotai/Kimi-K3) (1T MoE) on
8 H200 nodes (64 GPUs), **prefill/decode disaggregated** with tensor parallelism across 2 nodes
(TP=16) and expert parallelism. The configuration layers the agentic optimizations onto
disaggregated serving:

- **P/D disaggregation** (NIXL KV transfer) — prefill and decode use separate pools so heavy
  prefill never stalls token generation, stabilizing ITL.
- **TP=16 + Expert Parallelism** — the model is tensor-parallel sharded across 2 nodes (16 GPUs)
  with expert parallelism enabled. Each LWS group is a 2-node pair.
- **MultiConnector KV transfer** — combines NixlConnector for P/D KV transfer with
  MooncakeStoreConnector for tiered prefix caching (async load and lookup).
- **DSpark speculative decoding** — uses the
  [Inferact/Kimi-K3-DSpark](https://huggingface.co/Inferact/Kimi-K3-DSpark) draft model with 7
  speculative tokens to reduce decode latency.
- **FLASHMLA attention** — uses the FLASHMLA backend optimized for Kimi-K3's MLA architecture.
- **Disagg-aware prefix-cache routing** — the EPP scores prefix caches when picking a prefill
  endpoint, with token-load scoring for load balancing.

> [!NOTE]
> This deployment requires the Mooncake metadata service (`mooncake-master-store`) and a
> `mooncake-kimi-k3-config` ConfigMap to be deployed separately. See the
> [tiered-prefix-cache guide](../tiered-prefix-cache/README.md) for Mooncake infrastructure setup.

## Default Configuration

The default deployment is **`p3w2d1w2`** — 3 TP16 prefill replicas and 1 TP16 decode replica,
8 nodes / 64 GPUs.

| Parameter                | Value                                                                        |
| ------------------------ | ---------------------------------------------------------------------------- |
| Model                    | [moonshotai/Kimi-K3](https://huggingface.co/moonshotai/Kimi-K3)              |
| Accelerator              | NVIDIA H200 (8 GPUs per node, 8 nodes)                                       |
| Serving topology         | P/D disaggregated — 3 prefill replicas + 1 decode replica, each TP=16 + EP (2 nodes) |
| KV transfer              | MultiConnector (NixlConnector + MooncakeStoreConnector)                      |
| Speculative decoding     | DSpark (Inferact/Kimi-K3-DSpark, 7 draft tokens)                             |
| Attention backend        | FLASHMLA                                                                     |
| MOE backend              | Marlin                                                                       |
| Reasoning / tool-call    | kimi_k3 / kimi_k3                                                            |
| Container image          | `vllm/vllm-openai:nightly-5e35a6f4f9bbc217c599692157ca985c894373f7`          |

### Supported Hardware Backends

| Backend           | Directory                                      | Notes                                                  |
| ----------------- | ---------------------------------------------- | ------------------------------------------------------ |
| NVIDIA GPU (vLLM) | `modelserver/gpu/vllm/kimi-k3/`                | Composes `wide-ep-lws/modelserver/gpu/vllm-kimi-k3/` (H200, P/D disaggregated) |

## Prerequisites

- Installed proper client tools (kubectl, helm).
- Set the following environment variables:

  ```bash
  export REPO_ROOT=$(realpath $(git rev-parse --show-toplevel))
  source ${REPO_ROOT}/guides/env.sh
  export GUIDE_NAME="agentic-serving"
  export NAMESPACE=llm-d-agentic-serving
  ```

- Install the Gateway API Inference Extension CRDs:

  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${GAIE_VERSION}/v1-manifests.yaml
  ```

- Create a target namespace for the installation:

  ```bash
  kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
  ```

- [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../../helpers/hf-token.md) to pull models.
<!-- llm-d-cicd:skip start -->
  ```bash
  export HF_TOKEN=<your HuggingFace token>
  kubectl create secret generic llm-d-hf-token \
    --from-literal="HF_TOKEN=${HF_TOKEN}" \
    --namespace "${NAMESPACE}" \
    --dry-run=client -o yaml | kubectl apply -f -
  ```
<!-- llm-d-cicd:skip end -->

- Deploy the [LeaderWorkerSet controller](https://lws.sigs.k8s.io/docs/installation/) (the
  model server is a pair of LeaderWorkerSets).

- Deploy the Mooncake metadata service (the model server's MooncakeStoreConnector
  requires it):

  ```bash
  kubectl apply -k ${REPO_ROOT}/helpers/mooncake-master-store/base/
  ```

  The `mooncake-kimi-k3-config` ConfigMap is deployed automatically by the model server
  overlay via a Kustomize component ([`mooncake/kimi-k3`](mooncake/kimi-k3/)).
  The default uses embedded mode with RDMA. If your cluster needs a different `device_name`
  or `protocol`, edit [`mooncake/kimi-k3/configmap.yaml`](mooncake/kimi-k3/configmap.yaml)
  before deploying.

## Installation Instructions

### 1. Deploy the llm-d Router

This deployment uses the Kimi-K3 router values
([`router/kimi-k3.values.yaml`](router/kimi-k3.values.yaml)), which run
separate `prefill` and `decode` scheduling profiles with prefix-cache and token-load
scoring:

```bash
helm install ${GUIDE_NAME} \
    ${ROUTER_STANDALONE_CHART} \
    -f ${REPO_ROOT}/guides/recipes/router/base.values.yaml \
    -f ${REPO_ROOT}/guides/${GUIDE_NAME}/router/kimi-k3.values.yaml \
    -n ${NAMESPACE} --version ${ROUTER_CHART_VERSION}
```

### 2. Deploy the Model Server

Apply the Kustomize overlay for the recommended deployment (`p3w2d1w2`):

```bash
kubectl apply -n ${NAMESPACE} -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/gpu/vllm/kimi-k3/
```

This deploys 3 prefill LWS replicas (each spanning 2 nodes for TP=16) and 1 decode LWS
replica (also 2 nodes). Wait for pods to become ready (model load takes time; the startup
probe allows up to 60 minutes):

```bash
kubectl get pods -n ${NAMESPACE} -l llm-d.ai/model=Kimi-K3 -w
```

## Verification

### 1. Get the IP of the Proxy

```bash
export IP=$(kubectl get service ${GUIDE_NAME}-epp -n ${NAMESPACE} -o jsonpath='{.spec.clusterIP}')
```

### 2. Send Test Requests

Open a temporary interactive shell inside the cluster:

```bash
kubectl run curl-debug --rm -it \
    --image=cfmanteiga/alpine-bash-curl-jq \
    --env="IP=$IP" \
    --env="NAMESPACE=$NAMESPACE" \
    -- /bin/bash
```

Send a completion request:

```bash
curl -X POST http://${IP}/v1/completions \
    -H 'Content-Type: application/json' \
    -d '{
        "model": "moonshotai/Kimi-K3",
        "prompt": "Explain how a simple agent loop works in 3 sentences."
    }' | jq
```

Send a chat request with tool calling:

```bash
curl -X POST http://${IP}/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{
        "model": "moonshotai/Kimi-K3",
        "messages": [
            {"role": "user", "content": "What is the weather in San Francisco?"}
        ],
        "tools": [
            {
                "type": "function",
                "function": {
                    "name": "get_weather",
                    "description": "Get current weather for a city",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "city": {"type": "string", "description": "City name"}
                        },
                        "required": ["city"]
                    }
                }
            }
        ]
    }' | jq
```

## Cleanup

To clean up resources:

```bash
helm uninstall ${GUIDE_NAME} -n ${NAMESPACE}
kubectl delete -n ${NAMESPACE} -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/gpu/vllm/kimi-k3/
kubectl delete namespace ${NAMESPACE}
```
