# Satellite Full Deploy on VMware

Playbook that automates the full deployment of **Red Hat Satellite 6.18** on VMware vSphere — from creating the VM with RHEL 9 to installing and configuring Satellite.

The process is split into three plays:

1. **Collect credentials** — prompts for RHSM credentials upfront, before any provisioning begins
2. **VM Provisioning** — creates a VM on vSphere and installs RHEL 9 via ISO + automated Kickstart (role `install_rhel_from_iso_vmware`)
3. **Satellite Installation** — registers the server with RHSM, installs and configures Satellite with repositories, content views, lifecycle environments and activation keys (role `satellite_install`)

## Project structure

```
.
├── ansible.cfg                        # Ansible configuration
├── satellite-full-deploy-vmware.yml   # Main playbook
├── requirements.yml                   # Required roles and collections
├── files/
│   └── manifest.zip                   # Satellite manifest (required)
└── README.md
```

## Prerequisites

| Requirement | Details |
|---|---|
| Ansible | >= 2.14 |
| Python | >= 3.9 with `pyvmomi` and `requests` |
| xorriso | `xorriso` package installed on the controller |
| vCenter/ESXi | Access with permissions to create VMs |
| RHEL 9 ISO | Uploaded to an accessible datastore |
| Manifest | `manifest.zip` file in the `files/` directory |
| RHSM | Red Hat CDN access credentials |

## Installing dependencies

```bash
# System dependencies (kickstart ISO creation)
sudo dnf install xorriso

# Python dependencies
pip install pyvmomi requests

# Ansible roles and collections
ansible-galaxy install -r requirements.yml
```

## Configuration

### 1. Vault (passwords)

Create a vault to securely store passwords:

```bash
ansible-vault create vault.yml
```

Vault contents:

```yaml
vault_vcenter_password: "your-vcenter-password"
vault_ks_root_password: "$6$rounds=4096$salt$hash..."
```

To generate SHA-512 hashes compatible with Kickstart:

```bash
python3 -c "import crypt; print(crypt.crypt('YourPassword', crypt.mksalt(crypt.METHOD_SHA512)))"
```

### 2. ISO upload

Upload the RHEL 9 ISO to the VMware datastore and set the `rhel_iso_path` variable in the playbook:

```yaml
rhel_iso_path: "[datastore1] iso/rhel-9.6-x86_64-dvd.iso"
```

### 3. Satellite manifest

Place the `manifest.zip` file (generated from the [Red Hat Customer Portal](https://access.redhat.com)) inside the `files/` directory.

### 4. Playbook variables

Edit `satellite-full-deploy-vmware.yml` and adjust the variables to match your environment.

#### Play 2 — VM Provisioning

**Network variables** (defined in the `vars` block):

| Variable | Description | Example |
|---|---|---|
| `satellite_vm_ip` | VM static IP | `192.168.1.100` |
| `satellite_vm_mask` | Network mask | `255.255.255.0` |
| `satellite_vm_gateway` | Default gateway | `192.168.1.1` |
| `satellite_vm_dns` | DNS server | `192.168.1.1` |
| `satellite_vm_hostname` | VM FQDN | `satellite.example.com` |

**Role variables** (passed to `install_rhel_from_iso_vmware`):

| Variable | Description | Example |
|---|---|---|
| `vcenter_hostname` | vCenter address | `vcenter.example.com` |
| `vcenter_username` | vCenter user | `administrator@vsphere.local` |
| `vcenter_password` | vCenter password (vault) | `{{ vault_vcenter_password }}` |
| `vcenter_datacenter` | Datacenter name | `Datacenter` |
| `vcenter_cluster` | Cluster name | `Cluster01` |
| `vcenter_datastore` | Datastore | `datastore1` |
| `vcenter_network` | VM network | `VM Network` |
| `vcenter_folder` | VM folder | `/vm/satellite` |
| `vm_name` | VM name | `satellite` |
| `vm_num_cpus` | vCPUs | `4` |
| `vm_memory_mb` | RAM in MB | `20480` |
| `rhel_iso_path` | ISO path on datastore | `[datastore1] iso/rhel-9.6-x86_64-dvd.iso` |
| `ks_iso_datastore` | Datastore for kickstart ISO | `datastore1` |
| `ks_root_password` | Root password (vault) | `{{ vault_ks_root_password }}` |
| `ks_timezone` | Timezone | `America/Sao_Paulo` |
| `ks_lvs` | OS disk LVM layout | See playbook |
| `vm_extra_disk` | Extra disk for Satellite storage | See playbook |
| `ks_extra_packages` | Additional packages | See playbook |

#### Play 3 — Satellite Installation

| Variable | Description | Default |
|---|---|---|
| `satellite_install_username` | Satellite admin user | `admin` |
| `satellite_install_password` | Admin password | `redhat` |
| `satellite_install_organization` | Initial organization | `ACME` |
| `satellite_install_location` | Initial location | `ACME` |
| `satellite_install_version` | Satellite version | `6.18` |
| `satellite_install_base_os` | Base operating system | `rhel9` |
| `satellite_install_manifest_path` | Manifest file path | `{{ playbook_dir }}/files/manifest.zip` |
| `satellite_install_rhel_versions` | RHEL versions for repos/CVs | `[8, 9, 10]` |
| `satellite_install_epel` | Enable EPEL repos | `true` |
| `satellite_install_lifecycle_envs` | Lifecycle environments | DEV → HML → PRD |

## Usage

```bash
# Full execution (provisioning + installation)
ansible-playbook satellite-full-deploy-vmware.yml --ask-vault-pass

# With verbose output
ansible-playbook satellite-full-deploy-vmware.yml --ask-vault-pass -vvv

# Recreate an existing VM
ansible-playbook satellite-full-deploy-vmware.yml --ask-vault-pass \
  -e vm_force_recreate=true
```

At the start of execution, the playbook will interactively prompt for:
- **RHSM Username** — Red Hat Customer Portal username
- **RHSM Password** — Red Hat Customer Portal password

## Execution flow

```
┌─────────────────────────────────────────────────────────┐
│  Play 1: Collect credentials (localhost)                │
│                                                         │
│  1. Prompt for RHSM Username and Password               │
│  2. Store credentials for later plays                   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Play 2: VM Provisioning (localhost)                    │
│                                                         │
│  1. Verify xorrisofs is installed                       │
│  2. Generate Kickstart from template                    │
│  3. Build kickstart ISO (OEMDRV)                        │
│  4. Upload ISO to datastore                             │
│  5. Create VM on vSphere                                │
│  6. Power on VM — automated install via Anaconda        │
│  7. Wait for IP and SSH to become reachable             │
│  8. Cleanup (unmount ISOs, remove extra CD-ROM)         │
│  9. Add VM to in-memory inventory                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Play 3: Satellite Installation (provisioned VM)        │
│                                                         │
│  1. Configure firewall                                  │
│  2. Register with RHSM and enable repos                 │
│  3. Update the system                                   │
│  4. Install Satellite                                   │
│  5. Import manifest                                     │
│  6. Enable repositories (RHEL + EPEL)                   │
│  7. Synchronize repositories                            │
│  8. Create sync plan                                    │
│  9. Configure lifecycle environments (DEV/HML/PRD)      │
│ 10. Create content views                                │
│ 11. Publish and promote content views                   │
│ 12. Create activation keys                              │
│ 13. Configure global parameters and host groups         │
└─────────────────────────────────────────────────────────┘
```

## Roles

| Role | Repository | Description |
|---|---|---|
| `install_rhel_from_iso_vmware` | [GitHub](https://github.com/andrewlinuxadmin/ansible-install-rhel-from-iso-vmware) | VM provisioning and RHEL installation via ISO + Kickstart |
| `satellite_install` | [GitHub](https://github.com/andrewlinuxadmin/ansible-satellite-install) | Red Hat Satellite installation and configuration |

## Required collections

| Collection | Description |
|---|---|
| `community.vmware` >= 4.0 | VMware modules |
| `community.general` | General-purpose modules |
| `ansible.posix` | POSIX modules |
| `redhat.satellite_operations` | Satellite operations |
| `redhat.satellite` | Ansible modules for Satellite |

## Important notes

- The provisioning role **aborts** if a VM with the same name already exists. Use `-e vm_force_recreate=true` to recreate it
- Kickstart passwords can be SHA-512 hashes (starting with `$`) or plain text (auto-detected). SHA-512 hashes are recommended for production
- SSH root login is enabled (`PermitRootLogin: yes`) as it is required for the Satellite installation
- Default installation timeout is 900s (15 min); adjust `vm_wait_for_ip_timeout` as needed
- The `xorriso` package (provides `xorrisofs`) is required on the Ansible controller
- Satellite minimum hardware requirements (4 vCPUs, 20 GB RAM) are already configured in the playbook
- The OS disk uses custom LVM partitioning and a second 400 GB disk is added for Satellite storage (`/var/lib/pulp`, `/var/lib/pgsql`, `/opt/puppetlabs`, `/var/lib/containers`)
