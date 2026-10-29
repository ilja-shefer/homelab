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
