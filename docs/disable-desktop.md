# Disable the desktop on a Debian node

A k8s node should not have a GUI. If the Debian installer accidentally
installed the desktop variant, strip it down without reinstalling.

Telltale signs:

- Folders like `Desktop/`, `Pictures/`, `Music/` in your home dir after a fresh install.
- Box boots to a graphical login screen.
- `systemctl get-default` returns `graphical.target`.

## Steps

```sh
# 1. Check
dpkg -l xserver-xorg-core 2>/dev/null   # installed = GUI is possible

# 2. Boot to text mode from now on
sudo systemctl set-default multi-user.target

# 3. Protect sudo — autoremove can drop it otherwise
sudo apt-mark manual sudo

# 4. Purge desktop packages
sudo apt purge 'gnome*' 'gdm3*' 'xserver-xorg*' 'xorg*' 'x11-*' \
               task-gnome-desktop task-desktop
sudo apt autoremove --purge
sudo apt clean
sudo reboot

# 5. Verify
systemctl get-default                   # multi-user.target
dpkg -l xserver-xorg-core 2>/dev/null   # no packages found
```

## If `sudo` got removed anyway

```sh
su -                      # uses the root password set during install
apt update
apt install sudo
exit
```
