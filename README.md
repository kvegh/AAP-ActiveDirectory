# Active Directory login and administration with AAP

Version 0.0.1 (scaffold: documentation and decisions only, no playbooks yet).

Ansible Automation Platform (AAP) users sign in with their Active Directory
(AD) account over LDAPS. AD group membership decides which AAP teams they
join, and the teams hold scoped permissions on regional automation resources.
AAP also administers the AD side: organizational units, groups and users are
created and maintained by job templates.

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
2. **AD user/group administration from AAP**: OUs, groups and users declared
   in configuration and applied with the native `microsoft.ad` modules.

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

The role (reader / executor / admin) selects the permission level within the
allowed sets: view only; view plus use/execute; administer the scoped
resources. Hungary owns no resource set; EU is a clearance, not an owner.

## Design (decided)

- LDAPS (port 636, certificate verification), not plain LDAP.
- The domain controller's LDAPS certificate is signed by the shared
  AutoDeploy CA. AAP already trusts that CA through the installer
  (`ca_tls_cert`), so no SSH or container trust changes are needed.
- AD objects via native `microsoft.ad.ou`, `microsoft.ad.group` and
  `microsoft.ad.user`, looping over the matrix in the private repository.
- The domain controller gets the Windows evaluation activation, but no full
  Windows Update for now (known tradeoff: if AD or LDAPS problems trace back
  to missing updates, `win_updates` is added back).
- Verification by small native `uri` smoke tests (log in as one or two AD
  users, compare the visible job templates); no separate verifier program.
- The regional organization gets the hub Galaxy credentials through the
  native `ansible.controller.organization` module (`galaxy_credentials`),
  because the regional projects install collections from
  `collections/requirements.yml`.
- Dropped from the old implementation: retired users/groups, migration
  preview, Python verifier, gateway trust playbook, PowerShell identity
  script.

## Planned structure

Follows the common use case layout of the AutoDeploy deployer:

```
playbooks/        execution code (infra, AD objects, LDAPS, AAP config,
                  mock job, smoke test)
collections/      requirements.yml
docs/             demo guide
```

The private repository provides `aap-infra/` (domain controller VM),
`aap-config/` (matrix, authenticator, organization, teams, mappings) and
`aap-content/` (resource sets, projects, job templates, workflow), plus the
vaulted `env_variables.yml`.

## Open decisions

Not decided yet; no playbooks until they are:

- Who creates the domain controller VM and the AD AAP: nodes of the AD
  workflow, or the AutoDeploy deployer's infra node.
- Where the AD AAP runs: the development hypervisor by default; production
  only on explicit user request.
- AutoDeploy support for more than one AAP (bead-wd-0gnz.42).

## Relation to Disconnected Windows patching

The AD part was developed inside `kvegh/disconnected-windows-patching` and is
split out into this own use case (2026-11-23). The old code stays in that
repository's `archive/` as reference only; this repository is a fresh
implementation of the decided design. Both use cases use Windows Server VMs
on the same kind of KVM hypervisor and the same evaluation activation
approach, but they are deployed independently.

## Versioning

`VERSION` holds the semantic version; history in
[design_decisions_history.md](design_decisions_history.md), open work in
[TODO.md](TODO.md).
