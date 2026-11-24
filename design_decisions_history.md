# Design decisions history

## 2026-11-24 (CET): first implementation, version 0.1.0

User go-ahead 2026-11-24 13:05 CET (bead-wd-0gnz.45.7): code only, commit and
push; no deploy, no AAP jobs, no hosts touched. Implementation commit
`eca35c1`; private repository 0.1.0 alongside.

### User decisions (binding, bead-wd-0gnz.45 notes and the go-ahead)

- Own catalog entry "Active Directory" in the AutoDeploy deployer; the
  deployer's infra node creates win-ad (as in Windows patching); the use case
  is deployed onto its own AutoDeploy AAP, so config and content target that
  AAP.
- LDAPS kept; the win-ad certificate is signed by the shared AutoDeploy CA.
- 2026-11-24: no copy of the AutoDeploy CA in the AD repositories. The
  AutoDeploy private repository is the shared storage; the deployer passes
  its checkout as `autodeploy_secrets_dir` (AutoDeploy `944a138`, v0.5.0) and
  this playbook reads the key from its vault and the certificate from the
  AutoDeploy-NGNG inventory file. (Superseded intermediate instruction the
  same day: two CA file paths from the deployer; never implemented.)
- No full Windows Update on win-ad, activation kept; AD objects via native
  `microsoft.ad.ou/group/user`; old country matrix kept; retired users,
  migration preview and the Python verifier dropped; native `uri` smoke
  tests; self-service portal deferred.

### Agent choices

- win-ad's fixed address and MAC: below the DHCP range of the management
  network, next to the Windows patching entries (values in the private
  repository). The old AD reservation in the AutoDeploy hypervisor
  configuration is left to AutoDeploy (TODO).
- Activation directly over the management network's NAT (win-ad lives
  there), so no temporary activation NIC as in Windows patching.
- Administrator password: the base image's password is used once to set the
  domain Administrator password; later runs log in with that directly
  (`win_ping` with `ignore_unreachable` decides). The base image password and
  the hypervisor SSH key are copies of the Windows patching vault values in
  the AD vault (use case secrets, as Windows patching keeps its own copies).
- The LDAPS certificate is issued only when win-ad does not already serve a
  certificate the AutoDeploy CA verifies (`community.crypto.get_certificate`),
  so reruns keep the certificate; imported before the forest is created, so
  AD DS loads it at the promotion reboot; on an existing forest a new
  certificate triggers one reboot. IP SAN because AAP connects by address
  (no dependency on DNS).
- CA certificate read with `include_vars` (name-scoped), not
  `lookup + from_yaml`: tested in the deployer EE `ee-proj-aap-cac` 2.0.0
  (ansible-core 2.20.10), `from_yaml` fails on the file's `!unsafe` tags,
  `include_vars` works and leaves the file's other templates unevaluated.
  The CA key is passed as content (`ownca_privatekey_content`), never written
  to disk.
- Local test (no hosts): the localhost plays of `aap-infra.yml` ran in the
  deployer EE with a throwaway CA in a fake AutoDeploy checkout; the issued
  certificate verifies against that CA with the expected subject, SAN, key
  usage and EKU. It found that `include_vars dir` fails on
  `aap-infra/README.md` (fixed: `ignore_unknown_extensions`).
- `no_log` only where a module does not mask the secret itself (the
  `microsoft.ad`/`win_user`/`win_certificate_store` password options are
  `no_log` in their argument specs, so those tasks stay visible).
- Job templates: the 10 mock templates in the regional organization; the
  smoke test and **Active Directory - Add User** (demo part 2, survey
  user/country/role) in organization Default with an admin project. No
  workflow: deploy = infra + config + content already leaves the demo ready;
  the smoke test is one job.
- Custom view-only role definitions (`<prefix> <type> Reader`) for the
  reader teams and managed definitions for executor/admin, as in the old
  design; applied through the CoP `gateway_role_definitions` and
  `gateway_role_team_assignments` (the `ansible.platform` modules route
  `awx.*` types to Controller). The deployer does not dispatch them yet (open
  point, TODO).
- Smoke test signs in with HTTP basic authentication against
  `/api/controller/v2/job_templates/` (filtered to the regional
  organization) as the DK reader and the NZ executor; the AAP URL is
  evaluated from the deployer's `usecase_target_aap_fqdn` when the content is
  applied.

## 2026-11-23 (CET): split out of disconnected-windows-patching, version 0.0.1

The AD/AAP extension was developed in `kvegh/disconnected-windows-patching`.
Its last state with the AD code in place is commit `a382307` (v1.3.1); commit
`60829ee` (v2.0.0) moved it to that repository's `archive/` (AD parts:
`create_active_directory_vm.yml`, `prepare_active_directory_guest.yml`,
`deploy_active_directory.yml`, `configure_active_directory_ldaps.yml`,
`configure_aap_ad_regions.yml`, `tasks/configure_aap_ad_*.yml`,
`mock_regional_job.yml`, `docs/active-directory-demo-guide.md`,
`docs/active-directory-demo-quickstart.md` and the AD entries of
`archive/design_decisions_history.md`). The old code is reference only;
nothing is copied. Decisions tracked in bead-wd-0gnz.45.

### User decisions

- 20:49 Scope: AD login to AAP (AD group -> AAP team) and AD user/group
  administration from AAP. Self-service deferred (portal, bead-wd-0gnz.45.4);
  domain join, Kerberos and compliance later.
- 21:13 Keep the old country matrix: 12 users (DK/NO/NZ/HU x
  reader/executor/admin), 24 groups (GG people + DL access), 5 resource sets
  (DK-local, NO-local, DK-EU-approved, NO-EU-approved, NZ-local), each with
  inventory, host, credential, project and 2 inert mock job templates; EU
  access = DK + NO + HU.
- 21:18 Keep LDAPS (not plain LDAP).
- 21:19 The domain controller's LDAPS certificate is signed by the shared
  AutoDeploy CA, which AAP already trusts through the installer
  (`ca_tls_cert`); no SSH/container trust changes (old gateway trust
  playbook dropped). To verify at the first deploy.
- 21:20 No full Windows Update on the domain controller for now; activation
  kept. Add updates back if problems trace back to them (documented tradeoff).
- 21:21 AD objects via native `microsoft.ad.ou/group/user` instead of the
  old PowerShell identity script ("native ansible is always better").
- 21:22 Drop the old AAP access verifier playbook and Python verifier.
- 21:23 Native `uri` smoke tests approved (log in as 1-2 AD users, compare
  visible job templates). Drop retired users/groups and the migration
  preview.
- 21:25 First: AD stays in the Windows repositories as a second workflow.
  21:28 Superseded: AD/AAP becomes its own use case with the repository pair
  `kvegh/AAP-ActiveDirectory` (public) and `kvegh/AAP-ActiveDirectory-secrets`
  (private); start moving AD/AAP there.

### Agent choices

- 21:23 The regional organization gets the hub Galaxy credentials through
  `ansible.controller.organization` (`galaxy_credentials`) instead of the old
  `uri` reads, because the regional projects sync a repository with
  `collections/requirements.yml`.
- Scaffold first, playbooks after the open decisions (who creates the domain
  controller and the AD AAP, where the AD AAP runs, AutoDeploy multi-AAP
  bead-wd-0gnz.42).
- Readable resource set names (`DK-local`, ...) also as configuration keys
  (old compact keys such as `DKlocal` dropped).
- One shared, randomly generated password for the 12 demo users (one secret
  for the presenter), see the private repository.
