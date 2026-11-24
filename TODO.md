# TODO

- **First deploy verified** (v33, 2026-11-24, Workflow 6 and Job 22 — Active
  Directory - Smoke Test): the deployer applies the role definitions and the
  75 team role assignments (AutoDeploy v0.6.0); the gateway verifies LDAPS
  with the installer's CA; basic authentication works for LDAP users;
  `microsoft.ad` 1.12.1 syncs from `rh-certified`; WinRM from the AAP VM's
  DHCP address works; the job EE reaches the AAP's public name.
  Gap: the infra node installs `community.crypto` (not certified, no
  certified module issues certificates from an own CA) and, as the deployer
  EE lacks them, also `ansible.windows`/`microsoft.ad` from Galaxy.
- **Hypervisor build cleanup** (AutoDeploy): the old reservation
  of the old AD VM and its DNS host entry in the AutoDeploy-NGNG inventory
  are obsolete; this use case uses its own address (private repository).
- **Self-service portal** for AD/AAP requests: deferred (bead-wd-0gnz.45.4).
- **Later**: domain join of managed servers, Kerberos/WinRM with domain
  accounts, compliance checks; destroy job for win-ad.
- **Windows Update on the domain controller**: off for now; add `win_updates`
  back if AD/LDAPS problems trace back to missing updates.
