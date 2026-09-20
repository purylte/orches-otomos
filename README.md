# orches-otomos

My homelab containers, deployed with quadlet using [orches](https://github.com/orches-team/orches).

## Services

| Service        | Port              | Data                                                                      |
| -------------- | ----------------- | ------------------------------------------------------------------------- |
| Technitium DNS | 53, 5380 (web UI) | volumes `technitium-*`                                                    |
| Immich         | 2283              | library at `/var/mnt/data/immich/library`, DB in volume `immich-postgres` |
| copyparty      | 3923              | `/var/mnt/data/copyparty`, config in `copyparty.conf`                     |

Two secrets are created by hand and never stored in git: `immich_db_password` and `copyparty_private` (copyparty accounts and private volumes).

## Setup

The data disk mounted at `/var/mnt/data`.

```bash
# Let rootless Podman bind port 53, and start user services at boot
echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/50-unprivileged-ports.conf
sudo sysctl --system
loginctl enable-linger $(whoami)

# Data directories
sudo mkdir -p /var/mnt/data/immich/library /var/mnt/data/copyparty/{public,shared}
# (plus any private copyparty folders defined in the secret below)
sudo chown -R $USER:$USER /var/mnt/data/immich/library /var/mnt/data/copyparty

# Secrets, created before orches starts
tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32 | podman secret create immich_db_password -

umask 077
nano ~/copyparty-private.conf   # example in copyparty-private.example
podman secret create copyparty_private ~/copyparty-private.conf
shred -u ~/copyparty-private.conf

# Firewall: allow SSH and the services from the LAN only
LAN=192.168.1.0/24
sudo firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=$LAN service name=ssh accept"
for p in 53/tcp 53/udp 5380/tcp 2283/tcp 3923/tcp; do
  sudo firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=$LAN port port=${p%/*} protocol=${p#*/} accept"
done
sudo firewall-cmd --permanent --remove-service=ssh --remove-service=cockpit || true
sudo firewall-cmd --reload

# Start orches
mkdir -p ~/.config/orches ~/.config/containers/systemd

podman run --rm -it --userns=keep-id --pid=host --pull=newer \
  --mount \
    type=bind,source=/run/user/$(id -u)/systemd,destination=/run/user/$(id -u)/systemd \
  -v ~/.config/orches:/var/lib/orches \
  -v ~/.config/containers/systemd:/etc/containers/systemd  \
  --env XDG_RUNTIME_DIR=/run/user/$(id -u) \
  ghcr.io/orches-team/orches init \
  https://github.com/purylte/orches-otomos.git
```
