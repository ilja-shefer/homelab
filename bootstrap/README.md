# Bootstrap the Kubernetes cluster

## Disable swap on the host

kubelet refuses to start with swap on — memory accounting for pods must be
predictable.

### On Debian

1. Open `/etc/fstab` in an editor:

   ```shell
   sudo nano /etc/fstab
   ```

   Find the line with `swap` in the 3rd column, e.g.:

   ```
   UUID=<your-swap-uuid> none  swap  sw  0  0
   ```

   Prefix it with `#` to comment it out:

   ```
   # UUID=<your-swap-uuid> none  swap  sw  0  0
   ```

   Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

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

Pods on different nodes communicate via IP forwarding; kubelet refuses to start
without it.

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
