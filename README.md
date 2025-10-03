This repository contains Podman quadlet files for running self-hosted services
on a single machine with systemd, managed and deployed using Ansible.

### Layout

The repository is composed of directories for Ansible actions (`playbooks`),
files to push to the host (`templates`), and variables (`group_vars`).

The podman services are mostly configured through variables in the `quadlets.yml` file,
which allows easily enabling/disabling services and centralizing config for things
like reverse proxying.

#### Reverse proxying

Caddy is used because it offers transparent proxying, websocket support, and Let's
Encrypt TLS with no configuration. It's run in a container and Caddy communicates
with other pods over the Podman network and its ports 80/443 are published to the host.
(Actually, published to 10080/10443, and nftables handles routing to/from 80/443).

This is a bit easier to deploy and I don't have to worry about publishing
other containers' ports (and dealing with potential overlap) or setting up
network bridges so that uncontainerized Caddy could access the Podman network.

Note that the Caddy target is the DNS name corresponding to the pod on the
same network that Caddy should reverse proxy to. It depends on if the service
is being run through a 'podman-style' quadlet or a `[Kube]`-style quadlet,
where the latter generates a service that runs `podman kube play` on a valid Kubernetes spec.

`podman kube play` will always append `-pod` to the name of the created pod
(e.g. the deployment `metadata.name`), which is why that suffix is present in some targets but not others.
`PodName=foo` in `foo.pod` will produce a pod with exactly the name `foo` and no suffix.

**Choice of Caddy**

Traefik is very popular for reverse proxying to containers because of its labeling system,
but unless you use an option like `providers.docker.defaultRule=Host(`{{ normalize .ContainerName }}.example.com`)"`,
you still need to declare the reverse proxy configuration, just on the target container rather than
for the reverse proxy container.
Personally I don't find the former any nicer so I'll stick with Caddy in a Podman context.

Traefik also wants to access the Docker/Podman socket to do its label magic;
even if Podman has good support for that now, it's still a very high level of privilege,
and it requires SELinux workarounds when rootless.

#### Variable assignment

The inventory only contains a single group `home` with a single host for now.
`group_vars/home` is used to contain both regular and encrypted (vault) variables
files that apply to all playbooks for the home group.

(They could also be under `group_vars/all/` as per
["Organizing host and group variables", Ansible docs](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables).)

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
  - You must also `ansible-galaxy install -r requirements.yml`
- To edit the ansible-vault secrets: `ansible-vault edit groups_vars/home/vars.vault.yml` (will prompt for password)

For testing you can use `ansible` to run ad-hoc commands. For example:

```
ansible localhost --module-name ansible.builtin.debug \
  -a "msg={{ lookup('template', 'templates/Caddyfile.j2') }}" \
  --extra-vars "@group_vars/home/quadlets.yml" -e "@group_vars/home/main.yml" \
  -e "@group_vars/home/vault.yml" --vault-id vault_password_file
```

In addition to the `debug` module, `template` with a local `dest` can be helpful to view file output.

`--vault-id` without specifying a vault-id is used to point to a password file,
see <https://docs.ansible.com/ansible/latest/vault_guide/vault_using_encrypted_content.html#using-vault-id-without-a-vault-id>.

You can also run `ansible-playbook --check --diff` to show the changes that would be implemented by a run.

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

### Nextcloud AIO Setup

The Nextcloud AIO service is managed by Ansible. However, one manual step is required after the initial deployment:

1.  Log into the Nextcloud AIO admin interface.
2.  Navigate to the backup settings.
3.  Set the "Local backup location" to `/data/nextcloud-backups`.

The Ansible playbook automatically creates this directory and sets the correct permissions. Once this path is set in the AIO interface, Nextcloud's automated backups will be saved to this directory, which is then included in the weekly offsite Restic backup.

Also, all the DNS for the domain must of course be set up via your nameservers.

**Nextcloud admin interface**

`nextcloud-aio-mastercontainer` uses a self-signed cert, so we have to
disable TLS verification in Caddy.

Theoretically we could copy Nextcloud's self-generated certificate into the `caddy-data` volume:
`podman cp nextcloud-aio-mastercontainer:/mnt/docker-aio-config/certs/ssl.crt caddy-reverse-proxy:/data/nextcloud-admin-ssl.crt`
and it would work with the commented out Caddyfile configuration, but unfortunately
this doesn't work because the Nextcloud self-signed cert doesn't have a SAN.

Note also that the "Open Nextcloud AIO Interface" button in the admin settings
of the nextcloud container creates a link based on where `nextcloud-aio-mastercontainer`
detected it was running at the time it started `nextcloud-aio-nextcloud`.
Therefore, if you changed subdomains or configuration without a full restart,
or initially accessed the master container from localhost, you'll have to manually adjust the link.

### Security

Since workflows only run on main, I'm the only contributor, and GH actions runners are ephemeral,
my secrets are safe (as long as I trust GitHub with them). The vault password is passed directly
to Ansible via stdin to avoid any filesystem exposure or command history logging.

I can also safely add `pull_request` workflows since the repository setting (now a GH default, nice)
is to require approval to run workflows for non-contributors, preventing drive-by
['pwn requests'](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/).
(`pull_request` workflows from forks also don't have access to secrets even if they are approved.)
