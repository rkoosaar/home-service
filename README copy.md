# home-service

My home service stack running on a [Beelink EQ12](https://www.bee-link.com/eq12-n100-clone-1) with [Fedora IoT](https://fedoraproject.org/iot/). Applications are run as [podman](https://github.com/containers/podman) containers and managed by systemd to support my home infrastructure.

## Core components

- [direnv](https://github.com/direnv/direnv): Update environment per working directory.
- [podman](https://github.com/containers/podman): A tool for managing OCI containers and pods with native [systemd](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) integration.
- [renovate](https://github.com/renovatebot/renovate): Universal dependency automation tool.
- [sops](https://github.com/getsops/sops): Manage secrets which are commited to Git using [Age](https://github.com/FiloSottile/age) for encryption.
- [task](https://github.com/go-task/task): A task runner / simpler Make alternative written in Go.

## Setup

### System configuration

1. Install required system deps and reboot

    ```sh
    sudo rpm-ostree install --idempotent --assumeyes git go-task
    sudo systemctl reboot
    ```

2. Make a new [SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent), add it to GitHub and clone your repo

    ```sh
    export GITHUB_USER="rkoosaar"
    curl https://github.com/$GITHUB_USER.keys > ~/.ssh/authorized_keys
    sudo install -d -o $(logname) -g $(logname) -m 755 /var/opt/home-service
    git clone git@github.com:$GITHUB_USER/home-service.git /var/opt/home-service/.
    ```

3. Install additional system deps and reboot

    ```sh
    cd /var/opt/home-service
    go-task deps
    sudo systemctl reboot
    ```

4. Create an Age public/private key pair for use with sops

    ```sh
    age-keygen -o /var/opt/home-service/age.key
    ```

### Network configuration

> [!NOTE]
> _I am using [ipvlan](https://docs.docker.com/network/drivers/ipvlan) to expose most containers on their own IP addresses on the same network as this here device, the available addresses are mentioned in the `--ip-range` flag below. **Beware** of **IP addressing** and **interface names**._

1. Create the podman `containernet` network

    ```sh
    sudo podman network create \
        --driver=ipvlan \
        --ipam-driver=host-local \
        --subnet=10.1.0.0/24 \
        --gateway=10.1.0.1 \
        --ip-range=10.1.0.171-10.1.0.199 \
        containernet
    ```

2. Setup the currently used interface with `systemd-networkd`

    📍 _Setting the DNS server to a container used on this system might make dragons appear 🐉._

    ```sh
    sudo bash -c 'cat << EOF > /etc/systemd/network/enp6s18.network
    [Match]
    Name = enp6s18
    [Network]
    DHCP = yes
    DNS = 1.1.1.1
    DNS = 1.0.0.1
    [DHCPv4]
    UseDNS = false'
    ```

3. Setup `containernet` with `systemd-networkd`

    ```sh
    sudo bash -c 'cat << EOF > /etc/systemd/network/10-ipvlan-containernet.netdev
    [NetDev]
    Name=containernet
    Kind=ipvlan

    [IPVlan]
    Mode=l3
    EOF'
    sudo bash -c 'cat << EOF > /etc/systemd/network/containernet.network
    [Match]
    Name = containernet
    [Network]
    IPForward = yes
    Address = 10.1.0.170/24'
    ```

4. Disable `networkmanager`, the enable and start `systemd-networkd`

    ```sh
    sudo systemctl disable --now NetworkManager
    sudo systemctl enable systemd-networkd
    sudo systemctl start systemd-networkd
    ```

### Container configuration

> [!TIP]
> _To encrypt files with sops **replace** the **public key** in the `.sops.yaml` file with **your Age public key**. The format should look similar to the one already present._

View the [apps](./apps) directory for documentation on configuring an app container used here, or setup your own by reviewing the structure of this repository.

Using the included [Taskfile](./Taskfile.yaml) there are helper commands to start, stop, restart containers and more. Run the command below to view all available tasks.

```sh
go-task --list
```

### Optional configuration

#### Fish shell

> [!TIP]
> _🐟 [fish](https://fishshell.com/) is awesome, you should try fish!_

```sh
chsh -s /usr/bin/fish
# IMPORTANT: Log out and log back in
go-task dotfiles
```

#### Enable Chrony as a NTP server

> [!TIP]
> _⌚ You can also update `/etc/chrony.conf` with custom NTP servers._

```sh
sudo sed -i 's/^#allow .*/allow all/g' /etc/chrony.conf
sudo systemctl restart chronyd
```

#### Tune selinux

```sh
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
sudo systemctl reboot
```

#### Disable firewalld

```sh
sudo systemctl disable --now firewalld.service
```

## Network topology

| Name | Subnet | DHCP range | ARP reserved |
|------|--------|------------|--------------|
| LAN | 192.168.1.0/24 | 150-254 | 171-199 |
| TRUSTED | 192.168.10.0/24 | 150-254 | - |
| SERVERS | 192.168.42.0/24 | 150-254 | 120-149 |
| GUESTS | 192.168.50.0/24 | 150-254 | - |
| IOT | 192.168.70.0/24 | 150-254 | - |
| WIREGUARD | 192.168.80.0/28 | - | - |

## Related Projects

- [bjw-s/nix-config](https://github.com/bjw-s/nix-config/): NixOS driven configuration for running a home service machine, a nas or [nix-darwin](https://github.com/LnL7/nix-darwin) using [deploy-rs](https://github.com/serokell/deploy-rs) and [home-manager](https://github.com/nix-community/home-manager).
- [truxnell/nix-config](https://github.com/truxnell/nix-config): NixOS driven configuration for running your entire homelab.
- [joryirving/home-service](https://github.com/joryirving/home-service/): Docker-compose implementation of this repository.
