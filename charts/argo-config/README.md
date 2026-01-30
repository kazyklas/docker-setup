````md
# NVIDIA GPU Support in k3s (Arch Linux + Argo CD)

This document describes the **exact steps required** to enable **NVIDIA GPU scheduling** in a k3s cluster running on **Arch Linux**, using **containerd** and **Argo CD**.

This setup is **tested and confirmed working**.

---

## Overview

k3s does **not** enable GPU support automatically. To make GPUs available to Kubernetes pods, all of the following must be configured:

- NVIDIA drivers on the host
- NVIDIA Container Toolkit
- containerd GPU runtime configuration for k3s
- NVIDIA device plugin deployed via Argo CD
- Device plugin running with `runtimeClassName: nvidia`
- GPU Feature Discovery (GFD) enabled
- k3s restarted after runtime configuration

Missing **any** of these steps will prevent GPU workloads from running.

---

## 1. Install NVIDIA Drivers (Host)

Install NVIDIA drivers and utilities:

```bash
sudo pacman -S nvidia nvidia-utils
reboot
````

Verify the driver installation:

```bash
nvidia-smi
```

The GPU must be visible before continuing.

---

## 2. Install NVIDIA Container Toolkit (Host)

Install GPU container runtime support:

```bash
sudo pacman -S nvidia-container-toolkit
```

Verify installation:

```bash
which nvidia-container-runtime
```

---

## 3. Configure containerd Runtime for k3s (**CRITICAL**)

k3s uses its own **containerd** instance (not Docker).

Create or edit the k3s configuration file:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo nano /etc/rancher/k3s/config.yaml
```

Add the following configuration:

```yaml
containerd:
  extraConfig:
    plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia:
      runtime_type: io.containerd.runc.v2
```

This configuration enables the NVIDIA runtime inside containerd.

---

## 4. Restart k3s (**MANDATORY**)

Apply the runtime configuration:

```bash
sudo systemctl restart k3s
```

GPU support will **not** work until k3s is restarted.

---

## 5. Validate GPU Runtime with containerd (Before Kubernetes)

This step verifies GPU access **outside Kubernetes**.

Pull a CUDA image:

```bash
sudo ctr --namespace k8s.io images pull \
  docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04
```

Run a GPU test container:

```bash
sudo ctr --namespace k8s.io run --rm \
  --gpus 0 \
  docker.io/nvidia/cuda:12.2.0-base-ubuntu22.04 \
  test nvidia-smi
```

You must see `nvidia-smi` output.

If this fails, Kubernetes GPU workloads will not work.

---

## 6. Deploy NVIDIA Device Plugin via Argo CD

Create an Argo CD `Application` manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nvidia-device-plugin
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://nvidia.github.io/k8s-device-plugin
    chart: nvidia-device-plugin
    targetRevision: "0.15.0"
    helm:
      releaseName: nvidia-device-plugin
      values: |
        runtimeClassName: nvidia
        gfd:
          enabled: true

  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Apply the manifest:

```bash
kubectl apply -f nvidia-device-plugin.yaml
```

---

## 7. Restart NVIDIA Plugin Pods

Restart the device plugin and GFD pods to ensure correct initialization:

```bash
kubectl delete pod -n kube-system \
  -l app.kubernetes.io/name=nvidia-device-plugin

kubectl delete pod -n kube-system \
  -l app.kubernetes.io/component=gpu-feature-discovery
```

Argo CD will recreate the pods automatically.

---

## 8. Verify GPU is Advertised by the Node

Get the node name:

```bash
kubectl get nodes
```

Describe the node:

```bash
kubectl describe node <node-name> | grep -A5 Capacity
```

Expected output:

```text
nvidia.com/gpu: 1
```

---

## 9. Test GPU Workload in Kubernetes

Create a test pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  containers:
  - name: cuda
    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
```

Apply and check logs:

```bash
kubectl apply -f gpu-test.yaml
kubectl logs gpu-test
```

Seeing `nvidia-smi` output confirms GPU scheduling works.

---

## Common Pitfalls

* **Docker-based GPU guides do not apply** (k3s uses containerd)
* **Missing `runtimeClassName: nvidia` will break the device plugin**
* **Skipping the k3s restart will prevent GPU registration**

---

## Required Packages

| Package                    | Purpose             |
| -------------------------- | ------------------- |
| `nvidia`                   | NVIDIA GPU driver   |
| `nvidia-utils`             | NVML and tools      |
| `nvidia-container-toolkit` | GPU runtime support |
| `k3s`                      | Kubernetes          |
| `helm`                     | Helm charts         |
| `argocd`                   | GitOps deployment   |

---

## Result

The k3s cluster can now schedule GPU workloads using:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

This setup is suitable for production and GitOps-managed clusters.

```
```

