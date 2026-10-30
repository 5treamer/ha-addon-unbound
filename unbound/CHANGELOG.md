# Changelog

## 1.0.0

- Initial release.
- Unbound 1.26.2 built from source (NLnet Labs, commit
  `f0cd7067d950921427c91ade4ecf060e944f88eb`, tag `release-1.26.2`; includes
  the fix for CVE-2026-35501).
- Fully recursive, DNSSEC-validating; no forwarding by default.
- Runs as unprivileged user `unbound`, bridge network, port 5335.
