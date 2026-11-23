# TODO

- **Open decisions (block the playbooks)**:
  - who creates the domain controller VM and the AD AAP (AD workflow nodes
    or the AutoDeploy deployer's infra node);
  - where the AD AAP runs (development hypervisor by default; production
    only on explicit user request);
  - AutoDeploy multi-AAP support (bead-wd-0gnz.42).
- **Playbooks** after the decisions: domain controller VM, AD DS promotion,
  OUs/groups/users (`microsoft.ad`), LDAPS certificate from the AutoDeploy
  CA, AAP authenticator and team mappings, organization with hub Galaxy
  credentials, five resource sets, mock job, `uri` smoke tests.
- **First deploy check**: the AAP installer puts `ca_tls_cert` into the
  gateway container's trust store (needed for LDAPS verification).
- **Self-service portal** for AD/AAP requests: deferred (bead-wd-0gnz.45.4).
- **Later**: domain join of managed servers, Kerberos/WinRM with domain
  accounts, compliance checks.
- **Windows Update on the domain controller**: off for now; add `win_updates`
  back if AD/LDAPS problems trace back to missing updates.
