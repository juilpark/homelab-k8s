# AGENTS.md

## Purpose
- This repository tracks a learning-focused homelab Kubernetes environment built with Talos OS.
- The target deployment is a single bare-metal NAS/homelab machine hosting three VMs.
- High availability is not a goal for this lab because every VM runs on the same physical host.
- This repository is public, so secrets and environment-specific sensitive values must never be committed.

## Current Cluster Plan
- Kubernetes cluster name: `homelab-cluster`
- Control plane endpoint: `https://192.168.1.10:6443`
- VM layout:
  - `talos-control-1`: 1 control plane node
  - `talos-worker-1`: 1 worker node
  - `talos-worker-2`: 1 worker node
- Talos config generation command in use:

```bash
talosctl gen config homelab-cluster https://192.168.1.10:6443 \
  --install-disk /dev/vda \
  -o configs/
```

## Topology Snapshot
- Source of truth for the network diagram lives under [`00-topology/`](./00-topology).
- Diagram files:
  - [`00-topology/homelab-topology.drawio`](./00-topology/homelab-topology.drawio)
  - [`00-topology/homelab-topology.drawio.png`](./00-topology/homelab-topology.drawio.png)
- Network details captured in the diagram:
  - Home LAN gateway/OpenWRT: `192.168.1.1/24`
  - Bare-metal bridge uplink: `192.168.1.254/24`
  - Kubernetes-related VIPs:
    - Talos control plane VIP: `192.168.1.10`
    - MetalLB / ingress IP example: `192.168.1.2`
  - Node IPs:
    - `talos-control-1`: `192.168.1.11/24`, `10.123.0.11/24`
    - `talos-worker-1`: `192.168.1.21/24`, `10.123.0.21/24`
    - `talos-worker-2`: `192.168.1.22/24`, `10.123.0.22/24`
  - Talos internal/KVM switch network: `10.123.0.0/24`
  - Kubernetes logical networks:
    - Pod CIDR: `10.244.0.0/16`
    - Service CIDR: `10.96.0.0/12`

## Repository Layout
- [`README.md`](./README.md): top-level repository summary
- [`00-topology/`](./00-topology): topology source-of-truth and draw.io notes
- [`01-talos-os/README.md`](./01-talos-os/README.md): Talos bootstrap notes
- [`01-talos-os/configs/`](./01-talos-os/configs): generated Talos machine configs and per-node patches
- [`02-kubernetes/README.md`](./02-kubernetes/README.md): ordered Kubernetes add-on guide index
- [`02-kubernetes/`](./02-kubernetes): step-by-step add-on docs for Cilium, metrics-server, MetalLB, Traefik, and ExternalDNS

## Current State
- Talos bootstrap documentation exists in [`01-talos-os/README.md`](./01-talos-os/README.md).
- A generated Talos config directory already exists under [`01-talos-os/configs/`](./01-talos-os/configs).
- Node patch files now exist for the current three-node plan:
  - [`01-talos-os/configs/controlplane-1.patch.yaml`](./01-talos-os/configs/controlplane-1.patch.yaml)
  - [`01-talos-os/configs/worker-1.patch.yaml`](./01-talos-os/configs/worker-1.patch.yaml)
  - [`01-talos-os/configs/worker-2.patch.yaml`](./01-talos-os/configs/worker-2.patch.yaml)
- Kubernetes add-on documentation now exists under [`02-kubernetes/`](./02-kubernetes):
  - [`02-kubernetes/01-cilium.md`](./02-kubernetes/01-cilium.md)
  - [`02-kubernetes/02-metrics-server.md`](./02-kubernetes/02-metrics-server.md)
  - [`02-kubernetes/03-metallb.md`](./02-kubernetes/03-metallb.md)
  - [`02-kubernetes/04-traefik.md`](./02-kubernetes/04-traefik.md)
  - [`02-kubernetes/05-domain-cloudflare-external-dns.md`](./02-kubernetes/05-domain-cloudflare-external-dns.md)
- The current ingress direction is Traefik, not ingress-nginx.
- Domain and DNS automation notes currently assume Cloudflare plus ExternalDNS.

## Working Expectations
- Prefer keeping generated Talos base configs and handwritten patch files separate.
- Store node-specific configuration as explicit patch files so each VM bootstrap step is reproducible.
- Keep personal keys, certificates, kubeconfigs, secrets, and other sensitive machine-specific artifacts out of git via `.gitignore`.
- Prefer checked-in examples or sanitized templates for sensitive settings instead of real values.
- Preserve the learning-lab assumptions unless the user explicitly changes direction:
  - Single physical host
  - 1 control plane + 2 workers
  - No HA-driven complexity unless requested
  - External and management access stays on `192.168.1.0/24`
  - Node-to-node internal traffic prefers `10.123.0.0/24`
  - Pod and Service CIDRs stay internal-only unless explicitly redesigned
- When adding Talos or Kubernetes manifests, document why they are needed and where they should be applied from.
- Keep the numbered directory ordering meaningful for readers:
  - `00-topology`
  - `01-talos-os`
  - `02-kubernetes`

## Editing Guidance For Future Agents
- Read the topology assets and Talos notes before changing cluster network settings.
- Treat IP addresses and node roles in this file as the default baseline unless newer repository files override them.
- Do not replace generated Talos files blindly; prefer patching or regenerating intentionally with the command documented above.
- If you change bootstrap flow, also update [`01-talos-os/README.md`](./01-talos-os/README.md).
- If you change add-on order or the chosen ingress/DNS stack, also update [`02-kubernetes/README.md`](./02-kubernetes/README.md) and the step documents under [`02-kubernetes/`](./02-kubernetes).
- If you change node inventory, endpoint IPs, or VIPs, also update the topology documentation.

## Next Likely Tasks
- Capture VM resource sizing and hypervisor assumptions in documentation
- Add the next Kubernetes step document, likely `cert-manager`
- Document storage integration for the NAS-backed homelab
- After the cluster is stable, write a newcomer-friendly end-to-end setup guide
