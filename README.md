# trybpf

Run eBPF code inside Jupyter notebooks without compromising your Kubernetes nodes. This setup runs each user notebook in a dedicated Kata Containers VM with a custom, BTF-enabled Linux kernel.

## What it does

Multi-tenant JupyterHub setups usually cannot let users run eBPF. Loading BPF programs requires root privileges and capabilities like `SYS_ADMIN`, which would expose the shared host kernel. 

This project solves that by isolating each Jupyter pod inside a lightweight Kata Containers virtual machine. Users get root access to load BPF programs, but they only interact with the virtualized VM kernel, not the host.

## How it works

The repository contains files to build and deploy this environment:

1. **Kernel configuration (`btf.conf`)**: Enables BPF Type Format (BTF), kprobes, and tracing in the guest kernel.
2. **Kata Deploy builder (`Dockerfile`)**: Downloads Kata Containers source, patches the kernel configuration with our BTF options, compiles the custom kernel, and outputs a custom `kata-deploy` image with the `kata-qemu-btf` runtime class.
3. **Jupyter Notebook image (`bpf-notebook.Dockerfile`)**: Creates an Ubuntu-based Jupyter environment with `libelf-dev` and passwordless sudo.
4. **JupyterHub Helm config (`config.yaml`)**: Configures the JupyterHub spawner to request the `kata-qemu-btf` runtime class and add `SYS_ADMIN` capabilities to user pods.
5. **Cilium Policy (`cilium.yaml`)**: Secures pod egress by limiting access to the Kubernetes API server.

## How to run it

### 1. Build the images
Build the custom Kata deployment image:
```bash
docker build --build-arg KATA_VERSION=3.22.0 -t ghcr.io/gqvz/trybpf-kata-deply:latest .
```

Build the notebook image:
```bash
docker build -f bpf-notebook.Dockerfile -t ghcr.io/gqvz/trybpf-notebook:x86_64-ubuntu-24.04-4 .
```

### 2. Deploy Kata Container artifacts
Deploy the built Kata image to your Kubernetes cluster and verify that the `kata-qemu-btf` RuntimeClass is active.

### 3. Deploy JupyterHub
Install JupyterHub using Helm with the provided configuration:
```bash
helm upgrade --install jhub jupyterhub/jupyterhub \
  --namespace jhub \
  --create-namespace \
  --values config.yaml
```

### 4. Limit network access
Apply the Cilium policy:
```bash
kubectl apply -f cilium.yaml
```
