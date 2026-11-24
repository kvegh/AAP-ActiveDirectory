# TODO

- **Deployer dispatch (blocks the team roles)**: the AutoDeploy deployer
  applies neither `gateway_role_definitions` (config node) nor
  `gateway_role_team_assignments` (content node). Without them the teams
  exist but hold no roles and the smoke test fails. Proposed (AutoDeploy,
  not this repository): one dispatch entry each, the role definitions after
  the teams in `usecase-aap-config.yml`, the assignments after the workflows
  in `usecase-aap-content.yml`.
- **First deploy checks**:
  - the AAP installer puts `ca_tls_cert` into the gateway container's trust
    store (LDAPS verification of the authenticator);
  - the gateway accepts basic authentication for LDAP users (smoke test);
  - `microsoft.ad` is in Red Hat's certified content (hub remote
    `rh-certified`), `community.crypto` comes from Galaxy;
  - the base image's WinRM firewall rule allows the deploying AAP VM's DHCP
    address;
  - no older VM `win-ad` exists on the hypervisor (the playbook stops if its
    NIC differs);
  - the job EE reaches the AD AAP by its public name (smoke test).
- **Hypervisor build cleanup** (AutoDeploy): the old reservation
  of the old AD VM and its DNS host entry in the AutoDeploy-NGNG inventory
  are obsolete; this use case uses its own address (private repository).
- **Self-service portal** for AD/AAP requests: deferred (bead-wd-0gnz.45.4).
- **Later**: domain join of managed servers, Kerberos/WinRM with domain
  accounts, compliance checks; destroy job for win-ad.
- **Windows Update on the domain controller**: off for now; add `win_updates`
  back if AD/LDAPS problems trace back to missing updates.
