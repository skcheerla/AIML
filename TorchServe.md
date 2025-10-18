Absolutely ✅ — you can deploy **TorchServe** on **Kubernetes** in a very similar way to TensorFlow Serving.
Let’s go step-by-step through **how to serve any PyTorch model with TorchServe on Kubernetes**, including model export, configuration, manifests (Deployment + Service), and production best practices.


https://www.youtube.com/watch?v=XlO7iQMV3Ik&t=45s

---

## 🔧 Overview: What is TorchServe?

**TorchServe** is an open-source model serving framework for **PyTorch**, built by AWS & Meta.
It lets you:

* Deploy trained PyTorch models as REST/gRPC endpoints.
* Serve multiple models at once.
* Handle batching, metrics, model versioning, and scaling.
* Integrate with Kubernetes for production inference workloads.

### 🔌 Default ports

| Protocol       | Port | Description                             |
| -------------- | ---- | --------------------------------------- |
| REST API       | 8080 | Inference requests                      |
| gRPC API       | 7070 | gRPC requests                           |
| Management API | 8081 | Register/unregister models, scale, etc. |
| Metrics        | 8082 | Prometheus metrics                      |

---

## 🧠 Step 1 — Prepare Your Model for TorchServe

TorchServe serves models in `.mar` (Model ARchive) format.

You can generate a `.mar` using `torch-model-archiver`.

### Example (ResNet18)

```python
# save_model.py
import torch
import torchvision.models as models

model = models.resnet18(pretrained=True)
model.eval()

# Save model state_dict
torch.save(model.state_dict(), "resnet18.pth")
```

Then create a **handler** (optional, for custom logic) or use built-in image classifier handler.

### Package the model:

```bash
pip install torch-model-archiver

torch-model-archiver \
  --model-name resnet18 \
  --version 1.0 \
  --model-file resnet18.pth \
  --serialized-file resnet18.pth \
  --handler image_classifier \
  --extra-files index_to_name.json \
  --export-path model-store
```

This creates `model-store/resnet18.mar`.

---

## 📁 Step 2 — Directory layout

You’ll typically have:

```
model-store/
 └── resnet18.mar
config/
 └── config.properties
```

### Example `config.properties`

```
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082
model_store=/home/model-server/model-store
load_models=resnet18.mar
```

---

## ☸️ Step 3 — TorchServe on Kubernetes (PVC example)

We’ll mount the model store (and config file) using a PVC.

### a) Create namespace (optional)

```bash
kubectl create namespace torchserve
```

### b) Create a PVC

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: torchserve-pvc
  namespace: torchserve
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 5Gi
```

Apply it:

```bash
kubectl apply -f pvc.yaml
```

Copy your model-store directory and config.properties into that PV.

---

### c) Deployment YAML

```yaml
# torchserve-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: torchserve
  namespace: torchserve
spec:
  replicas: 2
  selector:
    matchLabels:
      app: torchserve
  template:
    metadata:
      labels:
        app: torchserve
    spec:
      containers:
      - name: torchserve
        image: pytorch/torchserve:0.11.0-cpu  # or GPU image if needed
        command: ["torchserve"]
        args:
          - "--start"
          - "--ts-config"
          - "/home/model-server/config/config.properties"
        ports:
        - containerPort: 8080
          name: inference
        - containerPort: 8081
          name: management
        - containerPort: 8082
          name: metrics
        volumeMounts:
        - name: model-store
          mountPath: /home/model-server/model-store
        - name: config-volume
          mountPath: /home/model-server/config
        readinessProbe:
          httpGet:
            path: /ping
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /ping
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
      volumes:
      - name: model-store
        persistentVolumeClaim:
          claimName: torchserve-pvc
      - name: config-volume
        persistentVolumeClaim:
          claimName: torchserve-pvc
```

---

### d) Service YAML

```yaml
# torchserve-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: torchserve
  namespace: torchserve
spec:
  selector:
    app: torchserve
  ports:
  - name: rest
    port: 80
    targetPort: 8080
  - name: mgmt
    port: 81
    targetPort: 8081
  - name: metrics
    port: 82
    targetPort: 8082
  type: ClusterIP
```

Expose externally (optional):

```bash
kubectl port-forward svc/torchserve 8080:80 -n torchserve
```

---

## 🧪 Step 4 — Test the TorchServe API

### Health check

```bash
curl http://localhost:8080/ping
```

### Inference request

```bash
curl -X POST http://localhost:8080/predictions/resnet18 \
     -T sample_image.jpg
```

### Management API

```bash
# List models
curl http://localhost:8081/models

# Register a new model (if dynamic mgmt enabled)
curl -X POST "http://localhost:8081/models?url=another_model.mar"
```

---

## ⚙️ Step 5 — Alternative: InitContainer (download models dynamically)

If your `.mar` files live in **S3/GCS/Azure**, use an initContainer to pull them into an `emptyDir`.

Example:

```yaml
initContainers:
- name: download-models
  image: amazon/aws-cli:latest
  command: ["sh", "-c"]
  args:
    - aws s3 cp s3://mybucket/model-store/ /home/model-server/model-store --recursive
  volumeMounts:
  - name: model-store
    mountPath: /home/model-server/model-store
  envFrom:
  - secretRef:
      name: aws-credentials
```

---

## 🚀 Step 6 — Scaling and Production Tips

| Category           | Recommendation                                                                  |
| ------------------ | ------------------------------------------------------------------------------- |
| **Autoscaling**    | Use HPA based on CPU or custom metrics (req/sec).                               |
| **GPU serving**    | Use `pytorch/torchserve:latest-gpu` and schedule onto GPU nodes.                |
| **Security**       | Restrict management API (8081) internally.                                      |
| **Metrics**        | Scrape `8082/metrics` with Prometheus.                                          |
| **Canary rollout** | Use Service mesh (Istio/Kserve) for weighted routing.                           |
| **Batching**       | Configure dynamic batching in `config.properties`.                              |
| **Multi-model**    | Serve multiple `.mar` files from `model-store/` or register via Management API. |

---

## 🧩 Optional: Using TorchServe with KServe (advanced)

You can run TorchServe under **KServe** (Kubeflow Serving) for autoscaling, revision control, and inference graph features.

KServe spec example:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: resnet18
spec:
  predictor:
    pytorch:
      storageUri: "s3://mybucket/model-store/"
```

KServe automatically creates TorchServe pods and manages scaling.

---

## 🧭 Step 7 — Troubleshooting

| Issue                 | Resolution                                     |
| --------------------- | ---------------------------------------------- |
| Pod CrashLoopBackOff  | Check logs: `kubectl logs <pod> -n torchserve` |
| 404 on `/predictions` | Model not loaded or name mismatch with `.mar`  |
| 500 errors            | Model file corrupted or handler missing        |
| Management API fails  | Ensure `management_address` is exposed         |
| GPU not used          | Use GPU image and set CUDA_VISIBLE_DEVICES     |

---

## ✅ Summary Checklist

| Step | Task                                      |
| ---- | ----------------------------------------- |
| 1    | Export model to `.mar`                    |
| 2    | Create `config.properties`                |
| 3    | Store both in PVC or image                |
| 4    | Deploy TorchServe Deployment + Service    |
| 5    | Verify with `/ping` and `/predictions`    |
| 6    | Tune scaling, probes, and resource limits |

---

Would you like me to generate a **single ready-to-apply YAML manifest** (namespace, PVC, deployment, service, configmap) for TorchServe — similar to the TensorFlow Serving example — that you can directly `kubectl apply`?

If yes, please tell me **which model provisioning style** you prefer:

1. 🧱 **PVC mount** (models stored on volume)
2. ☁️ **InitContainer download from S3/GCS**
3. 📦 **Baked into image**

I’ll produce a full working manifest for that setup.
