# Katello 5.0 Ansible role

An Ansible role that installs Foreman 5.0 with the Katello 5.0 plugin, or a
Smart Proxy with content, on RHEL or AlmaLinux 9 and 10 (x86_64), with Molecule tests.

It follows the upstream guide,
[Installing Foreman Server with Katello 5.0](https://docs.theforeman.org/5.0/Installing_Server/index-katello.html),
and installs with one method per EL version:

| EL version | `katello_install_method` | What it does |
| --- | --- | --- |
| 9 | `installer` | RPM install with `foreman-installer --scenario katello` |
| 10 | `foremanctl` | Containerized install (Podman) with `foremanctl deploy` ([guide](https://docs.theforeman.org/5.0/Installing_Server/index-containerized-katello.html)) |

`katello_install_method` defaults to the host's method, and the role rejects
the other one. Katello 5.0 does not ship `foreman-installer-katello` for
EL10, and the role does not use containers on EL9. A smart proxy uses the same
method as its server, so it must run the same EL major version.

## What the role does

1. Checks the platform (RHEL/AlmaLinux 9 or 10, x86_64), that the install method matches it, the admin password and the hostname format. A smart proxy also checks that it fits its server (see "Smart proxy").
2. Optionally sets the hostname (`katello_set_hostname`).
3. Picks the tuning profile and runs pre-flight checks from the guide: RAM and
   cores for that profile (20 GB and 4 for default), 4 GB swap, SELinux not
   disabled, no EPEL, forward and reverse DNS for the FQDN.
4. On RHEL, enables the BaseOS and AppStream repos with subscription-manager.
5. Imports the Foreman (and, for the installer method, OpenVox) signing keys,
   pinned by fingerprint, then installs the `foreman-release`, `katello-repos`
   and (installer method) `openvox8-release` repo packages for the host's EL version.
6. Runs `dnf upgrade`, installs chrony, the UTF-8 locale, and `foreman-installer-katello` or `foremanctl`.
7. Opens the firewall (installer: 8000, 9090, http, https; foremanctl: 8000, 8443, http, https).
   Applies OS tuning (tuned, THP off) and the recommended installer settings.
8. Runs the installer, then waits for `/api/v2/ping` to report `ok`.
9. Optionally sets up a custom or role-generated SSL certificate and enables plugins.
10. Optionally creates organizations and locations, LDAP / Active Directory
   authentication, content credentials (GPG keys, certificates), products and
   repositories, a Red Hat subscription manifest per organization and Red Hat
   repositories enabled from it (see below).

The installer runs once. The role stores a fingerprint of the installer
arguments, the certificate and `custom-hiera.yaml` checksums and the version
of the installer package (`foreman-installer-katello`, `foreman-proxy-content`
or `foremanctl`) in `/var/lib/ansible-katello/install.sha256`. It re-runs the
installer only when one of those changes (or `katello_force_install: true`),
so an upgraded installer package (from `katello_upgrade_packages`) is applied.
Changing the initial organization, location, admin username or admin password
does not re-run it, because the `--initial-*` options only apply to a new
database.

Both installers remember settings between runs. The role always passes
`--tuning`, so emptying `katello_tuning` goes back to the `default` profile.
With `foremanctl`, the role also removes features and server aliases that are
no longer in `katello_foremanctl_features` or `katello_foremanctl_server_aliases`
(`--remove-feature`, and for aliases an edit of
`/var/lib/foremanctl/parameters.yaml`, because foremanctl cannot drop one
alias). With `foreman-installer`, removing a plugin from the lists does not
uninstall it.

## Usage

```bash
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
cp examples/inventory.example.yml inventory.yml   # edit the host name
ansible-vault create group_vars/all/vault.yml   # vault_katello_admin_password: ...
ansible-playbook examples/site.yml --ask-vault-pass
```

The role needs ansible-core 2.16 or later, the first release that supports
Python 3.12, which EL10 runs.

Run these from the repository root. `ansible.cfg` sets `roles_path = ../`, so
the clone must be in a directory named `ansible-role-katello` for
`examples/site.yml` to find the role.

The install takes 20 to 60 minutes. The installer runs as an async task, so
an SSH drop will not kill it.

### Main variables

| Variable | Default | Notes |
| --- | --- | --- |
| `katello_admin_password` | `""` | Required, 8+ characters. Use Vault. |
| `katello_admin_username` | `admin` | |
| `katello_initial_organization` | `Default Organization` | Its label cannot be changed later. |
| `katello_initial_location` | `Default Location` | |
| `katello_deployment` | `server` | `server` or `proxy` |
| `katello_install_method` | `installer` on EL9, `foremanctl` on EL10 | The only method the host's EL version supports. |
| `katello_ssl_mode` | `default` | `default`, `custom` or `selfsigned` |
| `katello_proxy_server` | `""` | Proxy only: inventory name of the server. |
| `katello_foreman_plugins` | `[]` | Also `katello_foreman_proxy_plugins`, `katello_foreman_cli_plugins`. |
| `katello_hostname` | `ansible_facts.fqdn` | Lowercase FQDN that resolves both ways. |
| `katello_set_hostname` | `false` | Set the hostname to `katello_hostname`. |
| `katello_tuning` | `""` | `auto`, `default`, `medium`, `large`, `extra-large`, `extra-extra-large` |
| `katello_managed_hosts` | `0` | Expected host count, used by `katello_tuning: auto`. |
| `katello_installer_extra_args` | `[]` | Extra `foreman-installer` flags. |
| `katello_foremanctl_features` | `[]` | Passed as `--add-feature=` (`remote_execution` becomes `remote-execution`); features no longer listed are removed. |
| `katello_foremanctl_server_aliases` | `[]` | Passed as `--server-alias=`; aliases no longer listed are removed. |
| `katello_preflight_enforce` | `true` | `false` turns failed checks into warnings. |
| `katello_manage_rhel_repos` | `true` | Turn off if RHEL content comes from elsewhere. |
| `katello_manage_firewall` | `true` | |
| `katello_firewall_extra_services` | `[]` | e.g. `dns`, `dhcp`, `tftp`, `puppetmaster`, `mqtt` |
| `katello_*_rpm` | upstream URLs | Point at a mirror if needed. |
| `katello_run_installer` | `true` | `false` only prepares the host. |

Every variable is listed in
[`defaults/main.yml`](defaults/main.yml) and
validated by [`meta/argument_specs.yml`](meta/argument_specs.yml).

## Performance tuning

Based on the Foreman 5.0
[predefined tuning profiles](https://docs.theforeman.org/5.0/Administering_Project/index-katello.html)
and the [Tuning performance guide](https://docs.theforeman.org/5.0/Tuning_Performance/index-katello.html).

**Profile.** `katello_tuning` is passed as `--tuning` (installer and foremanctl):

| Profile | Hosts | RAM | Cores |
| --- | --- | --- | --- |
| default | up to 5,000 | 20 GB | 4 |
| medium | up to 10,000 | 32 GB | 8 |
| large | up to 20,000 | 64 GB | 16 |
| extra-large | up to 60,000 | 128 GB | 32 |
| extra-extra-large | 60,000+ | 256 GB | 48 |

`katello_tuning: auto` picks the smallest profile that covers
`katello_managed_hosts`, or, when that is 0, the largest profile the server's
RAM and cores support. The pre-flight RAM and CPU checks use the chosen
profile's minimums (with 5% slack on RAM), so a `large` profile on a 32 GB
server fails before installing. Smart proxies cannot use profiles; the role
rejects `katello_tuning` on a proxy.

**Recommended settings** (installer method), on by default:

| Variable | Default | Installer setting |
| --- | --- | --- |
| `katello_tuning_puma_threads` | `16` | `--foreman-foreman-service-puma-threads-max` |
| `katello_tuning_puma_workers` | `0` (auto) | `--foreman-foreman-service-puma-workers` |
| `katello_tuning_dynflow_workers` | `3` | `--foreman-dynflow-worker-instances` |
| `katello_tuning_postgresql` | `true` | `max_connections: 1000`, `shared_buffers: 2GB`, `work_mem: 8MB`, `autovacuum_vacuum_cost_limit: 2000` in `custom-hiera.yaml`, for medium and larger profiles |

Set a number to `0` (or `katello_tuning_postgresql: false`) to keep the
installer's default. `katello_tuning_postgresql_settings` overrides the
PostgreSQL values, and `katello_custom_hiera` adds any other Hiera keys from
the guide, for example:

```yaml
katello_custom_hiera:
  apache::mod::proxy::proxy_timeout: 600        # very high registration concurrency
  apache::mod::event::serverlimit: 64
  apache::mod::event::maxrequestworkers: 1024
```

These go in a marked block in `/etc/foreman-installer/custom-hiera.yaml`;
anything you wrote outside the block is left alone. Settings there override
the tuning profile, so add only what you need. Changing the block re-runs the
installer.

**OS tuning** (`katello_os_tuning: true`, servers and proxies): installs
`tuned` and activates a `katello` profile that extends
`throughput-performance` (`katello_tuned_parent_profile`) and turns off
Transparent Huge Pages, which the guide recommends for PostgreSQL and Redis.

## Plugins

```yaml
katello_foreman_plugins: [remote_execution, ansible, openscap, templates]    # --enable-foreman-plugin-*
katello_foreman_proxy_plugins: [remote_execution_script, ansible, openscap] # --enable-foreman-proxy-plugin-*
katello_foreman_cli_plugins: [remote_execution, ansible, openscap]          # --enable-foreman-cli-*
katello_installer_extra_args:                                               # plugin options
  - --foreman-proxy-plugin-remote-execution-script-mode=pull-mqtt
```

Underscores and hyphens both work in names. Adding a plugin to the list
re-runs the installer to enable it. On a smart proxy only
`katello_foreman_proxy_plugins` applies. With `foremanctl`, the plugin lists
are rejected; use `katello_foremanctl_features` instead (see `foremanctl features`),
which the role rejects with the RPM installer.

## SSL certificates

`katello_ssl_mode` picks the certificate for the web UI, API and content:

| Mode | What happens |
| --- | --- |
| `default` | The installer's own self-signed CA (upstream default). |
| `custom` | Your certificate. The role writes it to `katello_ssl_dir` (`/root/foreman_cert`) and passes it to the installer. |
| `selfsigned` | The role creates its own CA and a server certificate signed by it in `katello_ssl_dir`. |

```yaml
# custom
katello_ssl_mode: custom
katello_ssl_server_cert: "{{ lookup('file', 'certs/katello.example.com.pem') }}"
katello_ssl_server_key: "{{ vault_katello_ssl_key }}"        # unencrypted PEM
katello_ssl_ca_bundle: "{{ lookup('file', 'certs/ca-bundle.pem') }}"  # root first, then intermediates

# selfsigned
katello_ssl_mode: selfsigned
katello_ssl_extra_sans: [DNS:katello.example.com, IP:192.0.2.10]
katello_ssl_selfsigned_subject: {O: Example Corp, C: US}
katello_ssl_selfsigned_days: 825      # server certificate
katello_ssl_selfsigned_ca_days: 3650  # CA
```

Before installing, the role checks the certificate against the upstream
requirements. It must match the key, the key must not have a passphrase, it
must not be expired or a CA, it must have the FQDN as a DNS SAN and allow Key
Encipherment, and it must verify against the CA bundle.

Replacing a certificate (new `katello_ssl_server_cert`, or a renewed selfsigned
one) re-runs the installer with `--certs-update-server --certs-update-server-ca`.
After switching away from `default`, registered hosts need the new CA (see
"Refreshing the self-signed CA certificate on hosts" in the Foreman docs). To
go back to the installer's CA, run once with
`katello_installer_extra_args: [--certs-reset]`.

## Smart proxy

Set `katello_deployment: proxy` to install a Smart Proxy with content
(`foreman-proxy-content`) instead of a server. The Katello server must be in
the same inventory and already installed (or earlier in the same playbook, as
in [`examples/site.yml`](examples/site.yml)).

```yaml
katello_deployment: proxy
katello_proxy_server: katello.example.com   # inventory name of the server
katello_proxy_server_fqdn: katello.example.com   # defaults to katello_proxy_server
katello_proxy_organizations: [My Organization]
katello_proxy_locations: [My Location]
katello_foreman_proxy_plugins: [remote_execution_script, ansible]
```

What happens:

1. Before changing anything, the role checks that the proxy runs the same EL
   major version as the server. With `foremanctl`, it also reads the server's
   `/var/lib/foremanctl/parameters.yaml` to check that the server was deployed
   and that the proxy's `katello_ssl_mode` fits the server's certificates
   (`foremanctl auth-bundle` needs a proxy certificate when the server uses
   its own, and refuses one otherwise).
2. The proxy gets the same repositories and its own pre-flight checks (12 GB RAM).
3. On the Katello server (with `delegate_to`), the role makes the proxy's
   certificate bundle: `foreman-proxy-certs-generate` for the installer method,
   `foremanctl auth-bundle` for foremanctl. Files go in
   `katello_proxy_cert_dir` (`/root/smart-proxy_cert`).
4. The bundle is copied straight from the server to the proxy through Ansible
   (it never lands on your machine) and kept in `/var/lib/ansible-katello`,
   because upgrades need it.
5. The proxy installs with `foreman-installer --scenario foreman-proxy-content`
   (OAuth credentials are read from the server's installer answers) or
   `foremanctl deploy-proxy`.
6. The role registers the proxy, assigns its organizations and locations, and
   refreshes its features from the server to prove the two can talk.

The proxy's certificate comes from its own `katello_ssl_mode`:

- `default`: signed by the Katello server's built-in CA.
- `custom`: your proxy certificate, key and CA bundle (same variables as above,
  set on the proxy). It must come from the same CA as the server's certificate.
- `selfsigned`: signed by the role-generated CA on the server. The server must
  use `katello_ssl_mode: selfsigned` too.

With `foremanctl`, a proxy of a server with `default` certificates must use
`default`, and a proxy of a server with `custom` or `selfsigned` certificates
must use `custom` or `selfsigned`.

The proxy needs the server's `katello_admin_username`/`katello_admin_password`
to register, so keep those in `group_vars/all`. Set
`katello_proxy_regenerate_bundle: true` to rebuild a bundle.

## Organizations, locations, products and content credentials

After the install, the role can create these through the Foreman API. They
are applied in this order: organizations, locations, LDAP sources, content
credentials, products, repositories. Every entry takes `state: absent` to remove it.

```yaml
katello_organizations:
  - name: Engineering
    label: engineering                   # optional, cannot be changed later

katello_locations:                       # list a parent before its children
  - name: Datacenter East
    organizations: [Default Organization, Engineering]
  - name: Rack 12
    parent: Datacenter East

katello_content_credentials:             # one of content, file or url
  - name: RPM-GPG-KEY-AlmaLinux-9
    url: https://repo.almalinux.org/almalinux/RPM-GPG-KEY-AlmaLinux-9
  - name: Internal CA
    content_type: cert                   # default gpg_key
    file: files/internal-ca.pem          # path on the Ansible controller
    organization: Engineering

katello_products:
  - name: AlmaLinux 9
    gpg_key: RPM-GPG-KEY-AlmaLinux-9     # content credential name
    repositories:
      - name: AlmaLinux 9 BaseOS x86_64
        url: https://repo.almalinux.org/almalinux/9/BaseOS/x86_64/os/
        download_policy: on_demand
        os_versions: [rhel-9]
      - name: AlmaLinux 9 AppStream x86_64
        url: https://repo.almalinux.org/almalinux/9/AppStream/x86_64/os/
        download_policy: on_demand
        os_versions: [rhel-9]
        sync: true                       # sync this one now
```

- `organization` defaults to `katello_initial_organization` for credentials
  and products, and locations default to that organization too.
- Repositories default to `content_type: yum`. They also accept `ssl_ca_cert`,
  `ssl_client_cert`, `ssl_client_key`, `mirroring_policy`,
  `upstream_username`/`upstream_password`, `http_proxy_policy`, `arch`,
  `docker_upstream_name`, `include_tags`, `deb_*` and more; the full list is
  in [`defaults/main.yml`](defaults/main.yml).
- Repositories are not synced unless `katello_sync_repositories: true` or the
  repository sets `sync: true`. A sync can take hours and always reports a
  change, so leave it off if you want idempotent runs.
- Credentials with `url` are downloaded by the Ansible controller, not the
  Katello server.
- To delete a credential that a product still uses, remove it from the
  product first.

## Red Hat subscription manifests and repositories

To serve Red Hat content, export a subscription manifest for each organization
from the Red Hat Customer Portal (Subscriptions > Subscription Allocations),
then let the role upload it and enable the Red Hat repositories you need:

```yaml
katello_subscription_manifests:
  - organization: ACME                  # default: katello_initial_organization
    file: files/manifest_acme.zip       # on the Ansible controller
  - organization: Lab
    file: /root/manifest_lab.zip        # already on the Katello server
    remote_src: true
    repository_url: https://cdn.redhat.com   # optional CDN URL override

katello_redhat_repositories:
  - organization: ACME
    label: rhel-9-for-x86_64-baseos-rpms
    repositories:
      - releasever: "9"
  - organization: ACME
    name: Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
    product: Red Hat Enterprise Linux for x86_64
    repositories:
      - releasever: "9"
    sync: true                          # sync the enabled repositories now
  - organization: ACME
    label: satellite-client-6-for-rhel-9-x86_64-rpms
    all_repositories: true
```

- Manifests from the controller are copied to
  `/var/lib/ansible-katello/manifests/` (mode 0600) and uploaded from there.
  Uploading the same manifest again reports no change; a newer one replaces
  it, and an older one fails.
- A manifest's `state` is `present` (upload, the default), `refreshed`
  (refresh the manifest already on the server from the portal; no `file`
  needed) or `absent` (delete it). Only one entry per organization.
- Pick a repository set by `label` (easiest, it matches the repo ID shown by
  `subscription-manager repos`) or by `name` plus `product`. List the
  `releasever`/`basearch` combinations to enable in `repositories` (some sets
  need only one of them, for example RHEL 9 BaseOS needs only
  `releasever: "9"`), or set `all_repositories: true`.
- `state: disabled` disables the listed repositories again.
- Red Hat repositories are synced only when `sync: true` or
  `katello_sync_repositories: true`. The role syncs every enabled repository of
  that set.
- Manifests are uploaded before your own content credentials and products, so
  custom products can sit beside Red Hat ones in the same organization.

## LDAP / Active Directory authentication

After the install, the role can add LDAP auth sources through the Foreman API
(with the `theforeman.foreman` collection), map LDAP groups to Foreman user
groups, and run Foreman's "Test LDAP connection" on each source
(`katello_ldap_test_connection`, default `true`).

```yaml
katello_ldap_auth_sources:
  - name: Corp AD
    server_type: active_directory      # or free_ipa, posix
    host: dc01.corp.example.com
    # port: 636 and tls: true are the defaults
    account: svc-foreman@corp.example.com
    account_password: "{{ vault_ldap_bind_password }}"
    base_dn: DC=corp,DC=example,DC=com
    groups_base: OU=Groups,DC=corp,DC=example,DC=com
    ldap_filter: (memberOf=CN=Katello Users,OU=Groups,DC=corp,DC=example,DC=com)
    cacert: "{{ lookup('file', 'corp-root-ca.pem') }}"
    attributes:                          # attribute mappings
      login: sAMAccountName              # or userPrincipalName
      firstname: givenName
      lastname: sn
      mail: mail
      photo: thumbnailPhoto
    organizations: all                   # every organization on the server
    locations: [Default Location, Datacenter East/Rack 12]   # nested by title
    group_mappings:
      - usergroup: Katello Admins        # Foreman user group to create
        external_group: katello-admins   # LDAP group (defaults to usergroup)
        admin: true
      - usergroup: Katello Viewers
        external_group: katello-viewers
        roles: [Viewer]
```

- Attribute mappings: `attributes` takes `login`, `firstname`, `lastname`,
  `mail` and `photo` (the `attr_login`-style keys work too; `attributes` wins).
  Anything you leave out defaults by `server_type`: Active Directory uses
  `sAMAccountName`, FreeIPA and POSIX use `uid`, and all use `givenName`, `sn`
  and `mail`. Photo has no default.
- `cacert` is the CA that signed the LDAP server certificate. Without it the
  system trust store is used. With `foremanctl`, Foreman runs in a container
  that does not see the host trust store, so set `cacert` there.
- New LDAP users are created on first login (`onthefly_register`) and group
  membership is synced (`usergroup_sync`). Both default to `true`.
- Bind password: Foreman never returns it, so the role sends
  `account_password` only when it creates the source
  (`katello_ldap_account_password_update: on_create`). Set it, or a source's
  `account_password_update`, to `always` to send it on every run, for example
  after you rotate the password (that run then always reports a change).
- Organizations and locations: a list of names (nested locations by title,
  such as `Datacenter East/Rack 12`), or `all` for every one on the server at
  the time of the run. The default is the initial organization and location.
  Organizations and locations from `katello_organizations`/`katello_locations`
  are created first, so you can use them here. Set `state: absent` to remove a source.
- For an AD forest, point `host` at a Global Catalog (port 3269 with TLS).

## Users and user groups

Internal (local) users and user groups are created after the LDAP sources,
so a user or group can refer to them.

```yaml
katello_users:
  - login: jdoe
    password: "{{ vault_jdoe_password }}"
    firstname: Jane
    lastname: Doe
    mail: jdoe@example.com
    roles: [Manager]
    organizations: [Default Organization, Engineering]   # default: initial one
    locations: [Default Location]
    default_organization: Engineering
  - login: svc-ansible
    password: "{{ vault_svc_ansible_password }}"
    admin: true
  - login: ldap.user            # pre-create an LDAP user (no password)
    auth_source: Corp AD
  - login: olduser
    state: absent

katello_usergroups:
  - name: Helpdesk
    users: [jdoe]                       # members, by login
    roles: [Viewer]
  - name: Operators
    usergroups: [Helpdesk]              # nested groups
    roles: [Remote Execution User, Manager]
    external_groups:                    # LDAP groups whose members belong here
      - name: katello-operators
        auth_source: Corp AD
  - name: Old Team
    state: absent
```

- Users also take `description`, `locale`, `timezone`, `default_location` and `auth_source`.
- Passwords: Foreman never returns a password, so the role sends it only when
  it creates a user (`katello_users_password_update: on_create`). Set it, or a
  user's `password_update`, to `always` to reset passwords on every run (that
  run then always reports a change).
- Groups: `users`, `usergroups` and `roles` replace what the group has. A key
  you leave out is left alone, so leave out `users` on LDAP-synced groups.
  Nested groups can be defined later in the list.
- `admin: true` on a user or group gives full admin rights. Roles are
  existing Foreman roles (built-in ones like Viewer, Manager, Site manager,
  or ones you created).
- LDAP `group_mappings` is still there for the simple case; a group can
  appear in both places, and keys left out in one do not reset the other.

## Testing with Molecule

Molecule runs from the repository root. The `default` and `foremanctl`
scenarios use rootless Podman. On Windows, run them in WSL2 (for example
AlmaLinux 10 with `podman` installed); Ansible does not run natively on Windows.

```bash
ansible-galaxy collection install containers.podman   # used by the Podman driver
molecule test                  # default scenario (EL9, installer)
molecule test -s foremanctl    # foremanctl scenario (EL10)
molecule test -s full          # full install on a VM
```

Negative tests check which assertion failed, so a test only passes when the
role rejects the input for the expected reason. The container scenarios use
AlmaLinux: RHEL UBI images lack packages the role installs (such as
`chrony`), so test RHEL with the `full` and `proxy` scenarios on subscribed
VMs.

### `default` scenario (fast, containers, EL9)

Uses two `almalinux/9-init` systemd containers under Podman: a server (with
`katello_ssl_mode: selfsigned`) and a smart proxy. A container cannot meet the
RAM, SELinux and DNS requirements or run the installer, so this scenario sets
`katello_run_installer: false`, turns the pre-flight checks into warnings and
skips firewalld. It covers everything up to the installer:

- converge, then an idempotence run
- the repo packages, `foreman-installer-katello`, chrony and the locale are installed
- `foreman-release` is version 5.0.x
- the foreman, katello, pulpcore, candlepin and openvox repos are enabled
- the katello installer scenario exists and chronyd is running
- the installer did not run
- the proxy gets `foreman-proxy-content` and not the server packages
- the self-signed CA and server certificate exist with private keys, the right
  SANs, key usage and chain; reusing them in `custom` mode passes the checks,
  while a wrong key or a certificate for another hostname fails them
- custom mode without a certificate, a proxy pointing at a host outside the
  inventory, `foremanctl` on EL9, foremanctl features with the RPM installer,
  and a tuning profile on a proxy are rejected; a proxy on the same EL
  version as its server passes the proxy checks
- both hosts get the `katello` tuned profile with THP off; the server's
  `custom-hiera.yaml` gets the medium-profile PostgreSQL settings and the
  extra Hiera key, and the proxy's is untouched
- an empty admin password, a short uppercase hostname, an incomplete LDAP
  source, a credential with two sources and an unknown repository type are
  rejected; complete LDAP and content configs are accepted
- an unknown LDAP attribute key, an internal user without a password and a
  group that contains itself are rejected; a full users, groups and LDAP
  mapping config is accepted
- the pre-flight checks report too little RAM, swap and CPU (and stop the run
  when enforced) and pass a large enough host; `katello_tuning: auto` picks
  the right profile for host counts and hardware sizes
- the `foreman-installer` commands for the server and the proxy: certificate,
  plugin, tuning and proxy options, the `--initial-*` options kept out of the
  fingerprint (changing them or the admin password keeps it, changing a
  plugin does not), and `--tuning default` when `katello_tuning` is empty

Override the image with `MOLECULE_IMAGE=...`. It tests the RPM installer, so
it needs an EL9 image.

### `foremanctl` scenario (fast, containers, EL10)

The same setup as `default` (it reuses its `prepare.yml`), but for the
`foremanctl` method on `almalinux/10-init` by default. The method is left
unset, so the run also checks that the role picks `foremanctl` on EL10. It
checks that:

- `foreman-release` and `katello-repos` match the host's EL version, the
  Foreman signing key is imported and `foremanctl` is installed, while
  `foreman-installer-katello`, `foreman-proxy-content` and the OpenVox repo are not
- no `custom-hiera.yaml` is written and `foremanctl` did not run
- chronyd runs, the tuned profile disables THP, and the server's self-signed
  certificate names the server and verifies against its CA
- `katello_custom_hiera`, the RPM installer, installer plugin lists and a
  malformed feature name are rejected on EL10
- the `foremanctl deploy` command: certificate, feature, alias and tuning
  options, underscores in feature names, the `--initial-*` options kept out of
  the fingerprint, and, given the parameters of an earlier foremanctl run,
  `--remove-feature` and the alias removal for entries no longer listed
- the `foremanctl deploy-proxy` command, and the proxy checks: a proxy whose
  certificate mode fits the server passes, while one that does not, one on
  another EL version and one for a server that was never deployed fail

### `full` scenario (real install, existing VM)

Runs the whole role, including the installer, against a VM you provide
(RHEL or AlmaLinux 9 with `foreman-installer`, or 10 with `foremanctl`; 20 GB
RAM, 4 cores, 4 GB swap, a resolvable FQDN, SSH key access). Molecule does not
create or destroy the VM, so use a throwaway one.

```bash
export KATELLO_TEST_HOST=katello.example.com
export KATELLO_TEST_USER=root
export KATELLO_TEST_ADMIN_PASSWORD='changeme123'
molecule test -s full
```

It runs converge, idempotence (the second run must skip the installer), then
checks that `/api/v2/ping` reports every service `ok`, that the
`Molecule Org` organization exists, and that the Katello plugin is 5.0.x.
It also creates and checks an extra organization, a nested location, the
AlmaLinux 9 GPG key, a product using it and an AlmaLinux BaseOS repository
(not synced), plus an internal user (details, role, organizations, default
organization, and a login with its password), an admin user, and two nested
groups with members and roles. It runs with `katello_tuning: auto` and checks that the
`katello` tuned profile is active with THP off, and (installer) that Puma runs
16 threads and there are 3 Dynflow workers.

To include LDAP, also set these and the run adds a `Molecule LDAP` source,
tests the connection, and checks the source and the `Molecule LDAP Admins`
admin group exist, and that the source is in every organization and the two
chosen locations:

```bash
export KATELLO_TEST_LDAP_HOST=dc01.corp.example.com
export KATELLO_TEST_LDAP_TYPE=active_directory    # default
export KATELLO_TEST_LDAP_ACCOUNT=svc-foreman@corp.example.com
export KATELLO_TEST_LDAP_PASSWORD='...'
export KATELLO_TEST_LDAP_BASE_DN='DC=corp,DC=example,DC=com'
export KATELLO_TEST_LDAP_ADMIN_GROUP=katello-admins
# optional: KATELLO_TEST_LDAP_PORT (636), KATELLO_TEST_LDAP_TLS (true)
```

To include Red Hat content (RHEL VM with CDN access), point
`KATELLO_TEST_MANIFEST` at a manifest zip on the controller. The run uploads it
to `Molecule Org`, enables `KATELLO_TEST_RH_REPO_LABEL` (default
`rhel-9-for-x86_64-baseos-rpms`, releasever 9) without syncing, and checks
both:

```bash
export KATELLO_TEST_MANIFEST=$HOME/manifest.zip
export KATELLO_TEST_RH_REPO_LABEL=rhel-9-for-x86_64-baseos-rpms   # default
```

### `proxy` scenario (server + proxy, existing VMs)

Installs a server and a smart proxy on two VMs you provide, both on the same
EL major version (server 20 GB RAM, proxy 12 GB), by default with
`katello_ssl_mode: selfsigned`:

```bash
export KATELLO_TEST_HOST=katello.example.com
export KATELLO_TEST_PROXY_HOST=proxy.example.com
export KATELLO_TEST_ADMIN_PASSWORD='changeme123'
export KATELLO_TEST_SSL_MODE=selfsigned   # or default
molecule test -s proxy
```

It runs converge and idempotence, then checks that the proxy is registered,
has the Pulpcore feature, is in the initial organization and location, and
(selfsigned) that the server and proxy certificates come from the same CA.

## Linting

```bash
yamllint .
ansible-lint
```
