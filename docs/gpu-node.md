# GPU Node Guide

How to give the Kind node an NVIDIA GPU, so workloads such as local LLM inference run on
it instead of the CPU. Optional: the sandbox runs fine without a GPU.

AMD works too, through ROCm and the AMD device plugin rather than the steps below; the
`ollama` package takes a `gpuVendor` parameter for it. Only the NVIDIA path is documented
here, because it is the one that has been run.

Measured on a laptop RTX 3500 Ada (12 GB VRAM) with `mistral:7b`: **5.9 tokens/s on CPU,
72.5 on GPU** — one generation drops from 11 s to 1 s.

Kind needs four things wired in order. Skipping any one of them fails silently or with a
misleading error, so the checks at each step matter.

## 1. Host

The NVIDIA driver must already be installed (`nvidia-smi` works). Add the container
toolkit and make it the default Docker runtime:

```sh
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo nvidia-ctk config --in-place --set accept-nvidia-visible-devices-as-volume-mounts=true
sudo systemctl restart docker
```

The last flag is what lets a volume mount ask for devices, which is how the node claims
the GPU in step 2.

```sh
docker run --rm --gpus all ubuntu:24.04 nvidia-smi -L   # must list the GPU
```

## 2. Cluster

The node claims the GPU through one extra mount. The path is not a real file: the NVIDIA
runtime reads it as a request for every device **and** the driver userspace. Without it the
node gets `/dev/nvidia*` but no libraries, and nothing works.

The runtime reads that mount when it creates the node container, so an existing cluster
cannot be converted: it has to be recreated, and the platform and project installs replayed
on top (see the [README](../README.md)). Wire the GPU before installing anything.

```sh
kind delete cluster --name okdp-sandbox

cat > /tmp/okdp-sandbox-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: okdp-sandbox
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
  - containerPort: 30443
    hostPort: 443
  - containerPort: 30053
    hostPort: 30053
    protocol: UDP
  extraMounts:
  - hostPath: /dev/null
    containerPath: /var/run/nvidia-container-devices/all
EOF

kind create cluster --config /tmp/okdp-sandbox-config.yaml
docker exec okdp-sandbox-control-plane nvidia-smi -L   # must list the GPU
```

## 3. Inside the node

The node now sees the GPU, but its containerd still runs `runc`, so pods inherit nothing.
Install the toolkit in the node too:

```sh
docker exec okdp-sandbox-control-plane bash -c '
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -qq && apt-get install -y -qq curl gnupg
  curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
    | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
  curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
    | sed "s#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g" \
    > /etc/apt/sources.list.d/nvidia-container-toolkit.list
  apt-get update -qq && apt-get install -y -qq nvidia-container-toolkit
  nvidia-ctk runtime configure --runtime=containerd --set-as-default
  systemctl restart containerd'

docker exec okdp-sandbox-control-plane ln -sf /sbin/ldconfig /sbin/ldconfig.real
```

## 4. Device plugin

```sh
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
kubectl get node okdp-sandbox-control-plane -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
```

Keep the plugin's **default** device list strategy. Forcing `DEVICE_LIST_STRATEGY=volume-mounts`
(often suggested for Kind) makes the node's toolkit fail with `identifier is not a valid
UUID or index`, because the two ends then disagree on how devices are named.

If the plugin logs `Incompatible strategy detected auto`, step 3 was skipped or containerd
was not restarted.

## 5. Check

```sh
kubectl run gpu-check --rm -i --restart=Never --image=nvidia/cuda:12.6.2-base-ubuntu24.04 \
  --overrides='{"spec":{"containers":[{"name":"gpu-check","image":"nvidia/cuda:12.6.2-base-ubuntu24.04","command":["nvidia-smi"],"resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
```

## Using the GPU

A workload claims it with a resource limit; the `ollama` package exposes a `gpu` parameter
that sets this for you.

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
```

## Laptop power (read this before concluding the GPU is useless)

On a laptop, the GPU power budget is negotiated with the firmware and **cannot be raised
with `nvidia-smi -pl`** — it answers `not supported`. Powered over the display's USB-C
port instead of its own AC adapter, an RTX 3500 Ada caps at **25 W instead of 90 W**: it
stays at 210 MHz under 100 % load and inference runs *slower than on the CPU*, while the
model is correctly loaded in VRAM. Nothing in the Kubernetes layer hints at it.

```sh
nvidia-smi -q -d POWER | grep 'Current Power Limit'   # must show the default limit
nvidia-smi --query-gpu=clocks.sm,power.draw --format=csv   # under load, not the idle clock
```
