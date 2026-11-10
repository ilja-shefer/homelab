# Bootstrap the Kubernetes cluster

Everything up to `kubeadm init` runs on every node. I use tmux with synchronized
panes to type into all of them at once:

```shell
tmux new -s nodes 'ssh node0' \; split-window 'ssh node1' \; split-window 'ssh node2' \; select-layout even-vertical \; setw synchronize-panes on
```

Run `sudo -v` before pasting a block, otherwise the first `sudo` prompt takes the
pasted lines as its password. Without tmux, run each block on every node.

## Disable swap on the host

kubelet refuses to start with swap on — memory accounting for pods must be
predictable.

### On Debian

1. In `/etc/fstab`, find the line with `swap` in the 3rd column, e.g.:

   ```
   UUID=<your-swap-uuid> none  swap  sw  0  0
   ```

   Prefix it with `#` to comment it out:

   ```
   # UUID=<your-swap-uuid> none  swap  sw  0  0
   ```

2. List all swap units. On UEFI/GPT systems, systemd auto-generates swap units
   from the partition table independent of fstab, so masking is also required:

   ```shell
   sudo systemctl list-units --type=swap
   ```

3. Mask every listed unit. Use single quotes so the shell preserves the
   `\x2d` escapes in the unit name:

   ```shell
   sudo systemctl mask '<unit-name-from-step-2>'
   ```

   Masking creates a symlink to `/dev/null` so the unit can never start —
   stronger than `disable`, which auto-generators can override on next boot.

4. Reboot and verify `Swap: 0B`:

   ```shell
   sudo reboot
   free -h
   ```

## Configure kernel networking

Pods on different nodes communicate via IP forwarding, and kubeadm's preflight
check fails without it.

### Enable IPv4 packet forwarding

```shell
# Persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply now without reboot
sudo sysctl --system
```

Verify: `sudo sysctl net.ipv4.ip_forward` → `net.ipv4.ip_forward = 1`

## Install container runtime

Kubernetes supports several
[container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd),
and containerd is the most common one. Debian 13 ships containerd 1.7, but
Kubernetes 1.36 and later need 2.x. The containerd project doesn't publish .deb
packages, so I take `containerd.io` from Docker's apt repo. It has containerd and
runc, but no Docker Engine.

### Install containerd

Repo setup from [Docker's Debian guide](https://docs.docker.com/engine/install/debian/),
then only `containerd.io` instead of the full Docker package list:

```shell
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y containerd.io
```

Verify: `containerd --version` shows 2.x.

### Prepare `/etc/containerd/config.toml`

The `containerd.io` package installs a config made for Docker, which disables the
CRI plugin that kubelet talks to. Replace it with containerd's default config:

```shell
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

In `/etc/containerd/config.toml`, change `SystemdCgroup = false` to
`SystemdCgroup = true`, then restart containerd:

```shell
sudo systemctl restart containerd
```

Verify: `grep SystemdCgroup /etc/containerd/config.toml` → `SystemdCgroup = true`

Verify: `systemctl is-active containerd` → `active`

## Install kubeadm, kubelet and kubectl

kubeadm sets up the cluster, kubelet is the agent on every node that starts pods
through containerd, and kubectl is the command line for talking to the cluster.
All three come from the
[Kubernetes apt repo](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).

Install the packages apt needs to use the Kubernetes repo:

```shell
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Each minor version has its own repo, so the `v1.37` in the repo URL keeps the
cluster on 1.37.x. Moving to 1.38 later means changing that URL and going through
kubeadm's upgrade steps. The signing key is the same for every version:

```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install all three and hold them, so `apt upgrade` doesn't bump them. Cluster
upgrades go through kubeadm:

```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Start kubelet now and on every boot:

```shell
sudo systemctl enable --now kubelet
```

kubelet now restarts every few seconds, and `systemctl status kubelet` shows
`activating (auto-restart)`. That's expected: it has no config until
`kubeadm init` or `kubeadm join` writes one.

Verify: `kubeadm version -o short` → `v1.37.x`

Verify: `apt-mark showhold` → `kubeadm`, `kubectl`, `kubelet`

## Give each node a fixed IP

The control-plane IP ends up in the certificates, the kubeconfig and the join
command that `kubeadm` creates, so the cluster breaks if the router later hands
out a different address. In the router, reserve a DHCP address for every node and
note it in `secrets/nodes.yaml`. Reboot the node if the reserved address differs
from the current one.

To find the node's address, `kubeadm` looks at the interface that holds the
default route, which only works when there is exactly one. Check the routes:

```shell
ip route show
```

> **Example output**
>
> ```
> default via 192.168.1.1 dev eno1 proto dhcp src 192.168.1.10 metric 1002 mtu 1500
> 192.168.1.0/24 dev eno1 proto dhcp scope link src 192.168.1.10 metric 1002 mtu 1500
> ```
>
> - `default via 192.168.1.1 dev eno1` sends all traffic outside the home network to the router 
> through the network port `eno1`, and there must be only one such line
> - `src 192.168.1.10` is this node's own IP and must match the address reserved in the router
> - `192.168.1.0/24` is the home network, which the nodes reach directly without the router
>
