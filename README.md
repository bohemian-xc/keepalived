# keepalived (containerised)

A lightweight, containerised way to run keepalived to provision a Virtual IP (VIP) for high-availability setups. This repository provides a Docker Compose configuration and example configuration files to run keepalived in a container and advertise a floating IP between hosts.

**Key features**
- Simple Docker Compose setup to run `keepalived` in a container
- Example `keepalived.conf` and `.env` to get started quickly
- Guidance for common troubleshooting and validation steps

**Quick start**

1. Copy the example environment file and edit values:

```powershell
cp .env.example .env
# edit .env to match your network/VIP settings
```

2. Review `keepalived.conf.example` and adapt to your network/interface.

3. Start the service with Docker Compose:

```powershell
docker-compose up -d
```

4. Verify the container is running:

```powershell
docker-compose ps
```

Configuration
-------------

Files of interest:
- [keepalived/.env.example](keepalived/.env.example) — environment variables used by the compose file.
- [keepalived/keepalived.conf.example](keepalived/keepalived.conf.example) — sample `keepalived` configuration.
- [keepalived/docker-compose.yml](keepalived/docker-compose.yml) — compose orchestration for the container.

keepalived.conf example
-----------------------

Below is a minimal example to run a VRRP instance that advertises a VIP. Adapt `interface`, `virtual_router_id`, `priority`, and `authentication` as required.

```text
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        192.0.2.100/32
    }
}
```

Notes:
- `interface` must match the host interface you want the VIP bound to inside the container (host networking or macvlan setups may be needed).
- `virtual_router_id` must be the same across peers but unique on the local network.

.env example
------------

The `.env.example` provided contains environment variables referenced by the `docker-compose.yml`. Typical variables include the VIP, interface name, and any other runtime flags. After copying to `.env`, edit values to match your environment.

Running modes and networking
---------------------------

- Container networking: By default, containers have isolated networking. To allow the container to bind host IPs (VIPs) you typically must run the container with `network_mode: host` or use a macvlan network that provides an interface with access to your LAN.
- If you prefer to keep bridge networking, the VIP cannot be bound on the host network from inside the container unless you use additional kernel/network tricks (not recommended here).

Troubleshooting
---------------

- Check container logs:

```powershell
docker-compose logs -f
```

- Verify `keepalived` process inside container:

```powershell
docker exec -it <container_name> ps aux | grep keepalived
```

- Verify VIP is present on the host network:

Windows host (WSL2 or Docker Desktop): use `ipconfig` and check the guest interface or inspect from another LAN machine.
Linux host:

```bash
ip addr show dev eth0
ip neigh show
```

- If the VIP is not present:
  - Ensure the container has permissions and the correct network mode (`host` or macvlan).
  - Confirm `interface` in `keepalived.conf` matches the host interface name (sometimes `eth0` vs `ens3` etc.).
  - Check for conflicting ARP / duplicate IP on the LAN.

- Common logs to watch for:
  - `Could not open device` — usually indicates wrong interface or missing privileges.
  - `Authentication mismatch` — VRRP auth mismatch between peers.
  - `Advertise timeout` — connectivity issues between peers.

Validation and testing
----------------------

- Use `arping` from another machine to probe the VIP and ensure the owner responds.
- Temporarily stop `keepalived` on the MASTER to ensure the BACKUP takes over the VIP.

Security
--------

- Keep the VRRP authentication secret (`auth_pass`) secure and consistent across peers.
- Avoid exposing privileged host networking to untrusted containers.

Credits
-------

This repository and examples were prepared by bohemian-xc. See their work at https://github.com/bohemian-xc/.
