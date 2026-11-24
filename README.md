# Active Directory login and administration with AAP

Version 0.1.2. Ansible Automation Platform (AAP) users sign in with their
Active Directory (AD) account over LDAPS. AD group membership decides which
AAP teams they join, and the teams hold scoped permissions on regional
automation resources. AAP also administers the AD side: the domain
controller, its OUs, groups and users are declared in configuration and
applied with the native `microsoft.ad` modules.

This repository holds only execution code and documentation. Every
environment value (domain, names, addresses, users) and every secret lives in
the private repository `AAP-ActiveDirectory-secrets`.

## Demo story

> "We maintain people and their regional group memberships in Active
> Directory. When someone signs in, AAP checks AD and puts them into the
> matching AAP teams. Those teams already have permissions on the country's
> local resources or on its EU-approved resources, at the person's reader,
> executor or admin level."

Two blocks:

1. **AD login to AAP**: LDAPS authenticator, nested AD groups mapped to AAP
   teams, teams with object-scoped roles over a country matrix.
2. **AD user administration from AAP**: the job template
   **Active Directory - Add User** creates an AD user in a country and role
   group; the user can sign in to AAP at once.

The presenter walkthrough is in [docs/demo-guide.md](docs/demo-guide.md).

## Country matrix

- 4 user cohorts: DK, NO, NZ, HU; each with a reader, executor and admin
  user (12 users).
- 24 groups: 12 global groups (`GG`, people: one per country and role) and
  12 domain-local groups (`DL`, access: one per access scope and role).
- 4 access scopes: DK, NO, EU, NZ. EU-approved access = DK + NO + HU (the
  DK, NO and HU global groups are members of the EU domain-local groups).
- 5 resource sets owned by countries: `DK-local`, `NO-local`,
  `DK-EU-approved`, `NO-EU-approved`, `NZ-local`. Each has one inventory, one
  mock host, one synthetic credential, one project and two inert mock job
  templates (`Read`, `Simulate Change`).
- 12 AAP teams (4 scopes x 3 roles), each mapped from one DL group.

| Cohort | DK-local | NO-local | DK-EU-approved | NO-EU-approved | NZ-local |
| --- | --- | --- | --- | --- | --- |
| DK | yes | - | yes | yes | - |
| NO | - | yes | yes | yes | - |
| HU | - | - | yes | yes | - |
| NZ | - | - | - | - | yes |

The role selects the permission level within the allowed sets: reader =
view; executor = view the project, use inventory and credential, execute the
job templates; admin = administer the scoped resources. Hungary owns no
resource set; EU is a clearance, not an owner.

The matrix is `aap-config/ad-matrix.yml` of the private repository; the
infra playbook reads it directly. The AAP trees there (teams, maps, resource
sets, role assignments, smoke test expectations) repeat its values, because
the CoP filetree format cannot refer to another file: change both together.

## Structure

```
playbooks/
  aap-infra.yml           domain controller VM, forest, AD objects, LDAPS
                          (AutoDeploy deployer, infra node)
  mock-regional-job.yml   inert job of the 10 regional job templates
  smoke-test.yml          AD logins against the AAP API (job template)
  ad-user-add.yml         new AD user in a country/role group (job template)
collections/
  requirements.yml        ansible.windows, microsoft.ad, community.crypto
docs/
  demo-guide.md           presenter walkthrough
```

The private repository provides `aap-infra/ad-infra.yml` (VM, network,
connection, CA source paths), `aap-config/` (matrix, organization, teams,
LDAPS authenticator and maps, view-only role definitions), `aap-content/`
(hub collections, resource sets, role assignments, admin project,
credentials, inventory, smoke test and Add User job templates) and the
vaulted `env_variables.yml`.

## Topology

```
  Internet
     |
  management network (libvirt NAT, "internal")
     |-- AutoDeploy AAP VMs (DHCP): the deployer, the AD AAP (target)
     |-- win-ad   Windows Server 2025, AD DS + DNS, LDAPS on 636
                  (libvirt DHCP host entry, fixed address)
```

win-ad and the AD AAP share the management network of one KVM hypervisor.
AAP connects to win-ad by address (LDAPS and WinRM); no DNS entry is needed.

## Deploy flow

The AD use case is deployed onto its own AutoDeploy AAP (an AAP VM
of its own) with the AutoDeploy use case deployer, catalog entry
**Active Directory** (`infra_repository: kvegh/AAP-ActiveDirectory`,
`infra_ref: main`, `infra_playbook: playbooks/aap-infra.yml`). Its workflow
runs three nodes:

1. **Infra** (`playbooks/aap-infra.yml`, see below): win-ad with the forest,
   the AD objects and the LDAPS certificate.
2. **Config** (`aap-config/` of the private repository): the regional
   organization with the hub Galaxy credentials, 12 teams, the LDAPS
   authenticator, its login/organization/team maps, the view-only role
   definitions.
3. **Content** (`aap-content/`): hub collections, the five resource sets
   (credential, inventory, mock host, project, two job templates each), the
   team role assignments, the admin project, credentials and inventory, the
   job templates **Active Directory - Smoke Test** and
   **Active Directory - Add User**.

After the three nodes the demo is ready; **Active Directory - Smoke Test**
is the functional check.

### Infra: `playbooks/aap-infra.yml`

The deployer clones this repository and the private repository and runs,
from this checkout:

```
ansible-playbook --inventory localhost, --vault-id cac@<password file> \
  -e usecase_target_aap=<AAP VM name> \
  -e usecase_secrets_dir=<absolute path of the -secrets checkout> \
  -e autodeploy_secrets_dir=<absolute path of the AutoDeploy private checkout> \
  playbooks/aap-infra.yml
```

Steps: DHCP host entry and VM on the hypervisor (`virsh`, `qemu-img`,
`virt-install`, as in Windows patching); WinRM; the Administrator password
(the base image's password once, then the domain Administrator password);
evaluation rearm (new VM only); OS version check; host name; evaluation
activation; LDAPS certificate; the forest (`microsoft.ad.domain`, reboots);
OUs, groups, the LDAP bind account and the 12 users
(`microsoft.ad.ou/group/user`, loops over the matrix); finally a TLS check
of LDAPS against the AutoDeploy CA. Rerunnable: an existing host entry or VM
is left as it is, AD objects are declared state, and a valid LDAPS
certificate is kept.

**Evaluation rearm.** A win-ad created in this run gets
`slmgr.vbs /rearm` and a reboot right after the first login, before
activation and the forest, so every clone starts a full evaluation period
instead of the base image's clock (user decision 2026-11-24, as in Windows
patching). An existing VM is never rearmed: the Server 2025 image allows one
rearm only.

**Contract with the deployer** (AutoDeploy v0.5.0 or later):

| Variable | Content |
| --- | --- |
| `usecase_target_aap` | target AAP VM name (not used by this playbook) |
| `usecase_secrets_dir` | checkout of `AAP-ActiveDirectory-secrets` |
| `autodeploy_secrets_dir` | checkout of the AutoDeploy private repository |
| vault ID `cac` | opens both vaults (same password) |

The AutoDeploy private repository is the shared storage for values several
use cases need (user decision 2026-11-24, don't copy). The playbook reads
from it and keeps no copy: from the vault
`usecases/autodeploy/env_variables.yml` the SSH key of the automation
account (`vaulted_controller_credentials_hactar_ng_ansible_ngng_ssh_key_data`),
the Windows Server base image's local Administrator password
(`vaulted_windows_base_image_admin_password`) and the CA key
`vaulted_autodeploy_ca_key`; the certificate
`autodeploy_ca_cert` from the AutoDeploy-NGNG inventory file (paths in
`aap-infra/ad-infra.yml`). The inventory file is read with `include_vars`,
not `from_yaml`, because it has `!unsafe` values; only the certificate is
used, which relies on the lazy templating of ansible-core 2.19 or later
(checked in the deployer EE `ee-proj-aap-cac` 2.0.0, ansible-core 2.20).

The LDAPS certificate is issued on localhost with `community.crypto`
(`x509_certificate`, provider `ownca`): subject and DNS SAN = the DC's FQDN
(AD DS selects its certificate by it), IP SAN = win-ad's address (AAP
connects by address), Server Authentication, 825 days. It travels as a PFX
with a random one-time password and is imported into the computer store with
a non-exportable key; the CA certificate goes into the computer's trusted
roots. AAP trusts the certificate because the AAP installer got the same CA
(`ca_tls_cert`).

Needs from the deployer's execution environment: Python `pywinrm` and
`cryptography` (present in `ee-proj-aap-cac` 2.0.0), an SSH client; the
collections of `collections/requirements.yml` are installed by the deployer
from the default galaxy server when the EE lacks them (it lacks all three).
The deployer must run on an AAP VM of the same hypervisor (it reaches the
hypervisor and win-ad on the management network).

## Documented shortcuts and tradeoffs

This is a functional demo, not a security reference.

1. **No Windows Update on win-ad.** The domain controller gets the
   evaluation activation (an unactivated evaluation shuts down hourly after
   10 days) but runs the base image's patch level. If AD or LDAPS problems
   trace back to missing updates, `ansible.windows.win_updates` is added back
   (user decision 2026-11-23).
2. **One instance per hypervisor.** VM name, MAC and address of win-ad are
   fixed in the configuration; several AAPs may run on the hypervisor, but
   only one of them deploys this use case. A second AD use case needs its own
   values.
3. **One shared demo password** for the 12 users (one secret for the
   presenter); also used by users created with **Add User**.
4. **Dynamic address on the DC.** win-ad keeps its DHCP-configured NIC; the
   libvirt host entry makes the address fixed in practice.
5. **WinRM with NTLM and an unverified self-signed listener certificate**
   (base image), as in Windows patching.

## Automated functional tests

- `aap-infra.yml` ends with a TLS handshake to win-ad:636 verified against
  the AutoDeploy CA, plus a subject check.
- **Active Directory - Smoke Test** signs in to the AAP API as
  the DK reader and the NZ executor (basic authentication through the
  LDAPS authenticator; the first sign-in creates the AAP user and applies
  the maps) and compares the job templates each sees in the regional
  organization with the matrix: DK reader = the DK-local, DK-EU-approved and
  NO-EU-approved templates; NZ executor = the NZ-local templates.

Local checks without targets: `ansible-playbook --syntax-check` on every
playbook, `yamllint`.

## Conventions

- Playbooks: `playbooks/<object>-<action>.yml` or the deployer's fixed name
  `aap-infra.yml`; one per job template.
- Variables: snake_case; configuration prefixed `ad_infra_` (infra) and `ad_`
  (matrix, job templates); playbook-internal by purpose (`smoke_`, ...).
- `no_log: true` only where a secret is not masked by the module's own
  argument spec (vault loading, key file, connection passwords, `set_fact`,
  the smoke test's API calls).
- Native modules first; inline PowerShell only for Windows licensing (no
  module), as in Windows patching.

## Relation to Disconnected Windows patching

The AD part was developed inside `kvegh/disconnected-windows-patching` and is
split out into this own use case (2026-11-23). The old code stays in that
repository's `archive/` as reference only; this repository is a fresh
implementation of the decided design. Both use cases use Windows Server VMs
on the same kind of KVM hypervisor, the same base images and the same
evaluation activation, but they are deployed independently.

## Versioning

`VERSION` holds the semantic version; history in
[design_decisions_history.md](design_decisions_history.md), open work in
[TODO.md](TODO.md).
