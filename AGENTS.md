# ansible-role-inspectah

Ansible role for fleet-wide inspectah scan orchestration. Installs inspectah on target hosts, runs scans, fetches tarballs to the control node, and optionally cleans up.

## Virtual Environment Rule

**Never use `source .venv/bin/activate`.** Claude Code's shell safety prompt blocks `source` in subagents.

Use the venv's binaries directly:
```bash
.venv/bin/python3 script.py
.venv/bin/ansible-playbook playbook.yml
.venv/bin/ansible-lint
.venv/bin/molecule test
```

This is functionally equivalent for single commands and avoids interactive approval prompts.

## Repo Structure

```
defaults/main.yml      # All role variables (inspectah_* prefix)
vars/main.yml          # Internal constants — do not override
tasks/
  main.yml             # Entry point: install -> preflight -> scan -> fetch -> cleanup
  install.yml          # COPR or RPM install paths
  preflight.yml        # Version check, platform validation, extra_args sanitization
  scan.yml             # Async scan execution + orphan container cleanup
  fetch.yml            # Campaign-scoped tarball fetch to control node
  host_cleanup.yml     # Uninstall inspectah + remove COPR repo (only what role installed)
molecule/
  default/             # COPR install scenario
  air_gapped/          # RPM push scenario
  fallback/            # Tests without community.general collection
examples/
  site.yml             # Full 3-play pipeline: campaign ID -> scan -> aggregate
templates/             # Jinja2 templates
```

## Testing

```bash
# Lint
.venv/bin/ansible-lint
.venv/bin/yamllint .

# Molecule (requires podman)
.venv/bin/molecule test                              # default (COPR install)
.venv/bin/molecule test --scenario-name air_gapped   # RPM push
.venv/bin/molecule test --scenario-name fallback     # without community.general
```

Molecule uses podman as the driver. Test containers run UBI9 with `/usr/sbin/init` in privileged mode.

## Linting Standards

- **ansible-lint:** production profile, FQCN required, `no-changed-when` enforced
- **yamllint:** 120 char line length (warning), 2-space indent, implicit octals forbidden

## Conventions

- All role variables use the `inspectah_` prefix. Internal constants in `vars/main.yml` use `_inspectah_` prefix.
- Tasks use FQCN for all modules (e.g., `ansible.builtin.command`, not `command`).
- The role tracks what it installed and only cleans up its own artifacts (ownership-tracking facts from `install.yml` gate `host_cleanup.yml`).
- Scan tasks run async with configurable timeout (`inspectah_scan_timeout`, default 900s).
- Conflicting `inspectah_extra_args` flags are stripped in preflight with warnings, not silently ignored.
