# Overview

A repository containing my homelab containers and config files to be read using [orches](https://github.com/orches-team/orches)

# Quick Start

Allow unprivileged users to bind to port 53 (required for rootless Podman): Allow non-root users to bind to port 53 and above by running:

```
echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/50-unprivileged-ports.conf
sudo sysctl --system
```

run orches

```
loginctl enable-linger $(whoami)

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
