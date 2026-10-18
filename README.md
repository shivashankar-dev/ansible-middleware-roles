# Ansible Middleware Roles

Production-ready Ansible roles for the two middleware stacks that show up in almost every data-center-to-cloud migration: **Apache Tomcat on Linux** and **IIS on Windows Server**.

Both roles are idempotent, hardened by default, and pass `ansible-lint` on its strictest (`production`) profile. The Tomcat role is tested with Molecule on Ubuntu 22.04, Ubuntu 24.04 and Rocky Linux 9 on every push, through reusable GitHub Actions workflows.

## Roles

| Role | Targets | What it does |
| --- | --- | --- |
| [`tomcat`](roles/tomcat) | Ubuntu 22.04/24.04, RHEL/Rocky 9 | Installs OpenJDK 17 and Tomcat 10.1 from Apache with SHA-512 verification, runs it as a non-root system user under a hardened systemd unit, templates `server.xml` and JVM options, removes default apps, and checks a health endpoint |
| [`iis`](roles/iis) | Windows Server 2019/2022 | Installs IIS features, disables the Default Web Site, removes the `X-Powered-By` header, and creates app pools, sites, bindings, NTFS permissions and firewall rules from a simple list, then checks each site answers |

## Hardening built in

**Tomcat**
- Release tarball verified against Apache's published SHA-512 before install
- Runs as `tomcat` with `nologin`; config owned by root and read-only to Tomcat
- Shutdown port disabled; stop and start only through systemd
- Version banner and error-page server info hidden; default, manager and example apps removed
- systemd sandboxing: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem`, `ProtectHome`
- Versioned installs under `/opt/tomcat/apache-tomcat-<version>` with a `current` symlink, so upgrades and rollbacks are a variable change

**IIS**
- Each site gets its own app pool running as `ApplicationPoolIdentity` (no shared or domain accounts)
- Content folders grant that identity read-only access
- Default Web Site stopped and set not to auto-start
- `X-Powered-By` header removed server-wide

## Usage

```bash
ansible-galaxy collection install -r requirements.yml

# Linux
ansible-playbook -i inventories/example/hosts.yml playbooks/tomcat.yml

# Windows (WinRM)
ansible-playbook -i inventories/example/hosts.yml playbooks/iis.yml --ask-vault-pass
```

Override anything in `defaults/main.yml`, for example:

```yaml
tomcat_version: "10.1.34"
tomcat_http_port: 8080
tomcat_jvm_heap_max: 2g

iis_sites:
  - name: payments-api
    port: 443
    host_header: payments.internal
    physical_path: 'D:\sites\payments-api'
    dotnet_clr_version: ""   # No Managed Code (.NET Core / .NET 8 hosting)
```

## Testing

```bash
pip install ansible-core molecule "molecule-plugins[docker]"
cd roles/tomcat
MOLECULE_DISTRO=rockylinux9 molecule test
```

Each Molecule run: create a systemd container → converge → **idempotence** (a second run must report zero changes) → verify (service running, health page returns 200, example apps gone, JVM not running as root) → destroy.

## CI

```
.github/workflows/
├── reusable-ansible-lint.yml   # ansible-lint (production profile) + syntax checks
├── reusable-molecule.yml       # Molecule matrix for any role, any distros
└── ci.yml                      # caller: lint, then Tomcat on 3 distros
```

Adding a new role to CI is one more 5-line job in `ci.yml`.

## Author

**Shiva Shankar K**, Cloud Infrastructure & DevOps Architect. Google Professional Cloud Architect, CKA.
