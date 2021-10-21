# wasm-aks-demo

This repo contains a small rust-based demo app that can be deployed as a WebAssembly module on Azure Kubernetes Service.

> This guide is a draft!

## Tools & CLIs needed

- [Rust](https://www.rust-lang.org/tools/install)
- [wasmtime](https://wasmtime.dev/)
- [wasm-to-oci](https://github.com/engineerd/wasm-to-oci)

## Build wasm module

Run the app locally:

```bash
cargo run
```

Build a WebAssembly module:

```bash
rustup target add wasm32-wasi
cargo build --release --target wasm32-wasi
```

Run it:

```bash
wasmtime target/wasm32-wasi/release/wasm-demo.wasm
```

## Spin up the Azure resources

```bash
az feature register --namespace "Microsoft.ContainerService" --name "WasmNodePoolPreview"
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/WasmNodePoolPreview')].{Name:name,State:properties.state}"
az provider register --namespace Microsoft.ContainerService

az extension update --name aks-preview

id=$RANDOM

az group create -g wasm-demo

az acr create -n wasm0demo0$id \
  -g wasm-demo \
  --sku standard \
  -l westeurope

az aks create -n wasm-demo-aks \
  -g wasm-demo \
  -l westeurope \
  -c 1 \
  -s Standard_B2ms

az aks update -g wasm-demo -n wasm-demo-aks --attach-acr wasm0demo0$id

az aks nodepool add \
    --resource-group wasm-demo \
    --cluster-name wasm-demo-aks \
    --name wasi \
    --node-count 1 \
    -s Standard_B2ms \
    --workload-runtime wasmwasi

az aks get-credentials -g wasm-demo -n wasm-demo-aks
```

## Run the wasm module on AKS

Push module to the ACR:

```bash
az acr login -n wasm0demo0$id

wasm-to-oci push target/wasm32-wasi/release/wasm-demo.wasm wasm0demo0$id.azurecr.io/wasm-demo:latest
```

Apply the Deployment:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: wasm-demo
  name: wasm-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wasm-demo
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: wasm-demo
      annotations:
        alpha.wagi.krustlet.dev/default-host: "0.0.0.0:3001"
        alpha.wagi.krustlet.dev/modules: |
          {
            "wasm-demo": {"route": "/"}
          }
    spec:
      containers:
      - image: wasm0demo0$id.azurecr.io/wasm-demo:latest
        name: wasm-demo
        resources: {}
      nodeSelector:
        kubernetes.io/arch: "wasm32-wagi"
      tolerations:
      - key: "node.kubernetes.io/network-unavailable"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "kubernetes.io/arch"
        operator: "Equal"
        value: "wasm32-wagi"
        effect: "NoExecute"
      - key: "kubernetes.io/arch"
        operator: "Equal"
        value: "wasm32-wagi"
        effect: "NoSchedule"
status: {}
EOF
```
