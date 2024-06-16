# apps

## dhcp-proxy

<https://github.com/poseidon/dnsmasq>

## gatus

<https://github.com/TwiN/gatus>

## matchbox

<https://github.com/poseidon/matchbox>

## node-exporter

<https://github.com/prometheus/node_exporter>

## podman-exporter

<https://github.com/containers/prometheus-podman-exporter>

### podman-exporter configuration

1. Enable the `podman.socket` service

    ```sh
    sudo systemctl enable --now podman.socket
    ```

2. Start `podman-exporter`

    ```sh
    task start-podman-exporter
    ```

## traefik

<https://github.com/traefik/traefik>

## extra notes
bind and traefik fail to start becuse no cache folder
run
sudo mkdir /etc/containers/systemd/bind/cache
sudo mkdir /etc/containers/systemd/traefik/cache

traefik fails because no /run/podman/podman.sock
run sudo systemctl enable podman
sudo systemctl start podman
