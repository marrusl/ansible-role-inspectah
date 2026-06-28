# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial role implementation: install, preflight, scan, fetch, cleanup
- COPR and RPM install paths with version pinning
- Campaign-scoped fetch with sensitive data permission hardening
- Orphan container janitor (opt-in)
- Leave-no-trace host cleanup
- Molecule default and air-gapped scenarios
- CI: ansible-lint, yamllint, syntax-check, Molecule
- Example playbooks: full pipeline, scan-only, RPM push
