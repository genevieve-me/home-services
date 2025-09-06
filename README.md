This repository contains Podman quadlet files for running self-hosted services
on a single machine with systemd, managed and deployed using Ansible.

### Automated Deployment

The repository includes a GitHub Actions workflow (`.github/workflows/ansible-deploy.yml`) that automatically deploys changes when pushed to the main branch. The workflow:

- Detects changes to playbooks, templates, and configuration files
- Can be manually triggered with optional Ansible tags
- Uses `--vault-password-stdin` to securely pass the vault password without creating filesystem files
- Requires these GitHub Secrets to be configured:
  - `SSH_PRIVATE_KEY`: SSH private key for server access
  - `SSH_HOST_PUBLIC_KEY`: The server's public SSH host key (from `/etc/ssh/ssh_host_ed25519_key.pub` or similar)
  - `VAULT_PASSWORD`: Ansible vault password

### Local development

- Ansible is most easily installed with pipx: `[uvx] pipx install --include-deps ansible`
- To edit the ansible-vault secrets: `ansible-vault edit vars.vault.yml` (will prompt for password)

### Other server setup

* Podman role takes care of things like enabling user linger for rootless containers
* `PasswordAuthentication no` in `/etc/ssh/sshd_config.d/50-cloud-init.conf` and `~/.ssh/authorized_keys`
  are created during Ubuntu Server installation, alongside network config
* Could use FCOS or uCore, but it's low maintenance so Ansible works OK
    * Out of curiosity, may experiment with PXE network booting to provision server at some point

#### Manual Steps

**ZFS**

- created a zfs pool (`tank`) directly on one HDD
- Can replicate with `zpool create -o ashift=12 -O recordsize=256K -O compression=zstd -O atime=off tank /dev/sda1`
- ashift=12 is recommended: <https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Hardware.html#sector-size>
- recordsize is double default since pool stores media like photos/music
- created a filesystem (data) under it, with mount point /data and key encryption
- `zfs create -o mountpoint=/data -o encryption=on -o keyformat=raw -o keylocation=file:///root/datakey tank/data`
- I think you have to create `/root/datakey` first (eg with `openssl`)
- If I do eventually add disk, `zpool attach tank /dev/sda1 /dev/sdb1` will automatically create a mirror :)


To auto-mount (required a reboot for `zfs-load-key-tank-data.service` to work properly):

```sh
sudo mkdir /etc/zfs/zfs-list.cache/
sudo touch /etc/zfs/zfs-list.cache/tank
sudo zfs set canmount=on tank/data
```

**Networking**

For convenience, I gave `varda` (home server) a static internal IP on my router's DHCP server,
and of course I had to enable port forwarding for IPv4 HTTP(S) to it.
SSH is only allowed from internal network for now, I should probably set up Wireguard or Tailscale.

Also, all the DNS for the domain must of course be set up via your nameservers.

**Nextcloud admin interface certificate**

`nextcloud-aio-mastercontainer` uses a self-signed cert.

For the initial configuration, you can disable TLS verification if necessary:

```
	reverse_proxy nextcloud-aio-mastercontainer:8080 {
		transport http {
			tls_insecure_skip_verify
		}
	}
```

but the ideal way is to get Nextcloud's self-generated certificate by copying it into the caddy-data volume:
`podman cp nextcloud-aio-mastercontainer:/mnt/docker-aio-config/certs/ssl.crt caddy-reverse-proxy:/data/nextcloud-admin-ssl.crt`
where the Caddyfile expects to find it.

### Security

Since workflows only run on main, I'm the only contributor, and GH actions runners are ephemeral,
my secrets are safe (as long as I trust GitHub with them). The vault password is passed directly
to Ansible via stdin to avoid any filesystem exposure or command history logging.

I can also safely add `pull_request` workflows since the repository setting (now a GH default, nice)
is to require approval to run workflows for non-contributors, preventing drive-by
['pwn requests'](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/).
(`pull_request` workflows from forks also don't have access to secrets even if they are approved.)
