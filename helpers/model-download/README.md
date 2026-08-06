# Model Download Helper

Download any HuggingFace model to the host-path cache (`/mnt/local/hf-cache`) so model server pods can load weights locally.

## Prerequisites

1. Create the HuggingFace token secret — see [hf-token.md](../hf-token.md).

## Usage

1. Edit the `MODEL` env var value in `model-download.yaml` to the model you want to download.
2. Apply the manifest:

```bash
kubectl create -f helpers/model-download/model-download.yaml
```

> Use `kubectl create` (not `apply`) since `generateName` creates a new Job each time.

## Monitoring

```bash
kubectl logs -f job/<job-name>
```

Find the job name with `kubectl get jobs`.
