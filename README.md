# shefer-homelab

Self-hosted Kubernetes homelab.

## Hardware

| Role   | Count | RAM   | Disk            |
| ------ | ----- | ----- | --------------- |
| master | 1     | 16 GB | 256 GB NVMe     |
| worker | 2     | 8 GB  | 256 GB SATA SSD |

See `docs/nodes.example.yaml` for the inventory schema. Real inventory with MAC addresses and IPs lives in `secrets/` (gitignored).

## Layout

- `docs/` — node inventory template, network notes, install runbooks
- `bootstrap/` — scripts and notes for bringing up the k8s control plane and joining workers
- `cluster/` — platform components (ingress, cert-manager, storage, load balancer)
- `apps/` — workloads (media stack, DNS, etc.)
- `secrets/` — gitignored; real MACs, IPs, kubeconfigs, tokens
