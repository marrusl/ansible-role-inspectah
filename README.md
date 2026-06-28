# ansible-role-inspectah

Ansible role for running [inspectah](https://github.com/mrussell/inspectah) migration analysis across a fleet of RHEL, CentOS, or Fedora hosts.

## What this role does

1. **(Optional) Install** inspectah via COPR or local RPM push
2. **Preflight** -- validate prerequisites (inspectah >= 0.8.0, podman >= 4.4, nsenter, supported platform)
3. **Scan** the target host, producing a migration snapshot tarball
4. **Fetch** the tarball to the control node (with optional campaign-scoped directories)
5. **Clean up** -- remove target tarball (default), sweep orphan containers (opt-in)
6. **(Optional) Uninstall** -- remove inspectah and COPR repo from the target

## What this role does NOT do

- Run `inspectah aggregate` (control-node operation; see [examples/site.yml](examples/site.yml))
- Run `inspectah refine` (interactive, not automatable)
- Manage container registries or image mirrors
- Configure podman (podman >= 4.4 is a prerequisite)

## Requirements

### Target hosts

| Dependency | Minimum Version | Package |
|-----------|----------------|---------|
| podman | 4.4 | podman |
| nsenter | any | util-linux |
| dnf | any | system default |

### Control node

| Dependency | Version | Purpose |
|-----------|---------|---------|
| Ansible | >= 2.14 | Role execution |
| inspectah | >= 0.8.0 | Aggregate (example playbook only) |
| community.general | >= 5.0 | COPR module (optional, fallback exists) |

Install collection dependencies:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Supported Platforms

| Distribution | Versions | Architectures | Tier | Notes |
|-------------|----------|---------------|------|-------|
| CentOS Stream | 9 | x86_64 | Release-blocking | Molecule + real-host smoke in CI |
| RHEL | 9 | x86_64 | Smoke-tested | Manual validation, promote to release-blocking at Galaxy graduation |
| RHEL | 9 | aarch64 | Smoke-tested | When aarch64 CI runner available; promote at Galaxy graduation |
| RHEL | 10 | x86_64, aarch64 | Smoke-tested | COPR builds available, needs real-host validation |
| CentOS Stream | 9 | aarch64 | Smoke-tested | When aarch64 CI runner available |
| CentOS Stream | 10 | x86_64 | Smoke-tested | Tracks RHEL 10 |
| RHEL | 8 | x86_64, aarch64 | Best-effort | EL8 hosts are a primary migration target; podman >= 4.4 required (available via container-tools module stream) |
| CentOS Stream | 8 | x86_64 | Best-effort | EL8 migration target |
| AlmaLinux | 8, 9 | x86_64, aarch64 | Best-effort | RHEL rebuild, should work, not CI-gated |
| Rocky Linux | 8, 9 | x86_64, aarch64 | Best-effort | RHEL rebuild, should work, not CI-gated |
| Fedora | 40, 41 | x86_64 | Best-effort | Latest 2 releases, COPR builds available |

## Role Variables

### Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `inspectah_install` | `false` | Install inspectah on target hosts |
| `inspectah_install_method` | `"copr"` | `"copr"` (GPG-verified) or `"rpm"` (operator-trusted) |
| `inspectah_copr_repo` | `"mrussell/inspectah"` | COPR repository identifier |
| `inspectah_rpm_path` | `""` | Path to local RPM on control node (rpm method only) |
| `inspectah_install_version` | `""` | Pin version across fleet (empty = latest) |

### Scan

| Variable | Default | Description |
|----------|---------|-------------|
| `inspectah_base_image` | `""` | Target base image for cross-distro conversion (prefer `@sha256:...` digest-pinned refs for fleet consistency) |
| `inspectah_preserve` | `[]` | Sensitive data to preserve: `password-hashes`, `ssh-keys`, `subscription`, `all` |
| `inspectah_no_redaction` | `false` | Skip redaction (secrets remain unmasked) |
| `inspectah_scan_output` | `"/var/lib/inspectah/scans/{{ inventory_hostname }}.tar.gz"` | Tarball output path on target |
| `inspectah_scan_timeout` | `900` | Async timeout in seconds |
| `inspectah_scan_poll` | `30` | Poll interval in seconds |
| `inspectah_extra_args` | `[]` | Additional CLI flags (list of strings) |

### Fetch

| Variable | Default | Description |
|----------|---------|-------------|
| `inspectah_campaign_id` | `""` | Campaign ID for grouping tarballs (see examples) |
| `inspectah_fetch_dest` | `"{{ playbook_dir }}/scans"` | Base directory on control node for tarballs |

### Cleanup

| Variable | Default | Description |
|----------|---------|-------------|
| `inspectah_cleanup_host_tarball` | `true` | Remove tarball from target after fetch |
| `inspectah_cleanup_orphan_containers` | `false` | Sweep orphan `inspectah-baseline-*` containers (NOT safe for concurrent scans) |
| `inspectah_cleanup_host` | `false` | Uninstall inspectah and COPR repo after campaign |

## Example Playbook

### Full pipeline (scan + aggregate)

```yaml
# Play 0: Generate campaign ID (truly once, before serial batching)
- name: Initialize campaign
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Generate campaign ID
      ansible.builtin.set_fact:
        inspectah_campaign_id: "{{ lookup('pipe', 'date +%Y%m%dT%H%M%S') }}"

# Play 1: Scan the fleet
- name: Scan fleet hosts
  hosts: scan_targets
  # serial: 10
  # max_fail_percentage: 10
  become: false
  vars:
    inspectah_campaign_id: "{{ hostvars['localhost'].inspectah_campaign_id }}"
  roles:
    - role: ansible-role-inspectah
      vars:
        inspectah_install: true

# Play 2: Aggregate
- name: Aggregate scan results
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Run inspectah aggregate
      ansible.builtin.command:
        argv:
          - inspectah
          - aggregate
          - "{{ playbook_dir }}/scans/{{ inspectah_campaign_id }}"
          - --output-dir
          - "{{ playbook_dir }}/aggregate"
      changed_when: true
```

See [examples/](examples/) for scan-only and RPM push playbooks.

### Why three plays?

Ansible's `run_once` fires once per `serial` batch, not once per play. With `serial: 10` and 50 hosts, `run_once` would fire 5 times, creating 5 different campaign directories. Play 0 generates the campaign ID on localhost before serial batching begins, guaranteeing all hosts write to the same directory.

## Fleet-Scale Notes

- **Thundering herd:** Use `serial: 10` (or appropriate batch size) in the scan play to avoid concurrent base image pulls overwhelming the registry.
- **Disk space:** Tarballs are typically 5-50MB per host. Plan for target-side (scan output + base image cache) and control-side (fetched tarballs + aggregate output) storage.
- **Network bandwidth:** At 500 hosts with 20MB average tarballs, expect ~10GB of transfer. Run the control node close to the fleet.
- **Scan duration:** Cold scans (no cached image) take 3-5 minutes. Warm scans take 30-60 seconds. Default timeout is 15 minutes.

## Security

- All commands use `ansible.builtin.command` with `argv` (no shell injection).
- `--ack-sensitive` is added automatically when preserve or no-redaction is set.
- Sensitive campaigns tighten fetch directory permissions to `0700`.
- COPR installs are GPG-verified by dnf. RPM push installs are operator-trusted (no provenance check).
- See the [spec](https://github.com/mrussell/inspectah/blob/main/process-docs/specs/proposed/ansible-role-spec.md) for full security analysis.

### Digest-pinned base images

Tag-based image references (e.g., `registry.redhat.io/rhel9/rhel-bootc:9.6`) can resolve to different digests across hosts if the tag is updated mid-campaign. For reproducible fleet-wide scans, use `@sha256:...` digest-pinned references:

```yaml
inspectah_base_image: "registry.redhat.io/rhel9/rhel-bootc@sha256:abc123..."
```

This guarantees every host in the campaign scans against the same image content, regardless of when each host pulls the image.

## Testing

```bash
# Lint
ansible-lint
yamllint .

# Molecule (requires podman)
molecule test                              # default scenario (COPR install)
molecule test --scenario-name air_gapped   # RPM push scenario
molecule test --scenario-name fallback     # COPR without community.general
```

## License

MIT

## Author

Mark Russell
