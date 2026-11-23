# Design decisions history

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
