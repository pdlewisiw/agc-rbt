# RBT Quadlet Units

These Quadlet files are a side-by-side alternative to the existing
`docker-compose*.yaml` files. The default units mirror
`docker-compose.prod.yaml`: nginx is exposed publicly, while MapProxy and
TileserverGL stay on the internal `rbt-network` bridge.

## Files

- `rbt-network.network` creates the `rbt-network` bridge.
- `rbt-tileservergl.container` runs TileserverGL.
- `rbt-mapproxy.container` runs MapProxy after TileserverGL.
- `rbt-nginx.container` runs nginx after MapProxy.
- `rbt-tileservergl-debug.container` and `rbt-mapproxy-debug.container` expose
  direct debug ports matching `docker-compose.debug.yaml`.
- `rbt-nginx-debug.container` is the nginx unit wired to the debug services.

## Path Assumption

The units assume the appliance repo is installed at:

```bash
/srv/rbt
```

If the repo lives somewhere else, update the `/srv/rbt/...` bind mounts before
installing the units.

## Airgap Image Load

Quadlet does not load image tar files. Load the images into the same Podman
storage that will run the Quadlet services before starting the units.

For the existing RBT airgap pattern, images are loaded as the rootless `podsvc`
user:

```bash
sudo runuser -u podsvc -- bash -lc 'podman load -i /opt/setup/rbt-airgap-kit/images/rbt-images-20260415-mapproxy.tar'
sudo runuser -u podsvc -- bash -lc 'podman load -i /opt/setup/rbt-airgap-kit/images/rbt-images-20260415-nginx.tar'
sudo runuser -u podsvc -- bash -lc 'podman load -i /opt/setup/rbt-airgap-kit/images/rbt-images-20260415-tileservergl.tar'
sudo runuser -u podsvc -- bash -lc 'podman images'
```

If the bundle uses one combined tar, load that instead:

```bash
sudo runuser -u podsvc -- bash -lc 'podman load -i /opt/setup/rbt-airgap-kit/images/rbt-images.tar'
```

The local image names must match the `Image=` values in the Quadlet files:

```text
docker.io/mjj203/mapproxy:latest
docker.io/mjj203/nginx:latest
docker.io/mjj203/rbt:tileservergl-3395-latest
```

If `podman images` shows different names or tags, retag them before starting:

```bash
sudo runuser -u podsvc -- bash -lc 'podman tag <loaded-image-id-or-name> docker.io/mjj203/mapproxy:latest'
```

## Rootless Install

Install these units for `podsvc` when using the existing airgap image load
commands.

```bash
sudo loginctl enable-linger podsvc
sudo install -d -m 0700 -o podsvc -g podsvc /srv/podsvc/podsvc-home/.config/containers/systemd
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-network.network /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-tileservergl.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-mapproxy.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-nginx.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo systemctl start user@$(id -u podsvc).service
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); systemctl --user daemon-reload'
```

Start the stack:

```bash
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); systemctl --user start rbt-nginx.service'
```

Check status and logs:

```bash
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); systemctl --user status rbt-nginx.service rbt-mapproxy.service rbt-tileservergl.service'
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); journalctl --user -u rbt-nginx.service -u rbt-mapproxy.service -u rbt-tileservergl.service -f'
```

Rootless nginx needs permission to bind ports `80` and `443`:

```bash
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

Persist that setting with the rest of the host setup.

## Rootful Install

Only use this mode if the images were loaded as root with `sudo podman load`.
From the repo root on the target Linux host:

```bash
sudo install -d /etc/containers/systemd
sudo install -m 0644 quadlet/rbt-network.network /etc/containers/systemd/
sudo install -m 0644 quadlet/rbt-tileservergl.container /etc/containers/systemd/
sudo install -m 0644 quadlet/rbt-mapproxy.container /etc/containers/systemd/
sudo install -m 0644 quadlet/rbt-nginx.container /etc/containers/systemd/
sudo systemctl daemon-reload
```

Start the stack:

```bash
sudo systemctl start rbt-nginx.service
```

Check status and logs:

```bash
sudo systemctl status rbt-nginx.service rbt-mapproxy.service rbt-tileservergl.service
sudo journalctl -u rbt-nginx.service -u rbt-mapproxy.service -u rbt-tileservergl.service -f
```

Stop the stack:

```bash
sudo systemctl stop rbt-nginx.service rbt-mapproxy.service rbt-tileservergl.service
```

## Optional Debug Ports

The debug units are mutually exclusive with the default `rbt-tileservergl` and
`rbt-mapproxy` units because they use the same container names.

To expose TileserverGL on `8080` and MapProxy on `5000`, install the debug
units instead of the default TileserverGL and MapProxy units:

```bash
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-tileservergl-debug.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-mapproxy-debug.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo install -m 0644 -o podsvc -g podsvc quadlet/rbt-nginx-debug.container /srv/podsvc/podsvc-home/.config/containers/systemd/
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); systemctl --user daemon-reload'
sudo runuser -u podsvc -- bash -lc 'export XDG_RUNTIME_DIR=/run/user/$(id -u); systemctl --user start rbt-nginx-debug.service'
```

## Validate Generation

On the target host, validate generated units before starting them:

```bash
sudo systemd-analyze --generators=true verify rbt-nginx.service
```
